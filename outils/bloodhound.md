# Cheatsheet — BloodHound

Cartographie des relations Active Directory et calcul des **chemins d'attaque** vers les comptes à privilèges. L'outil d'énumération central du module 5.

> ⚠️ **Cadre.** Labs (GOAD) ou mission autorisée uniquement. La collecte génère du trafic LDAP/SMB visible dans les journaux.

**Idée clé** : BloodHound modélise l'AD en **graphe** — les nœuds (utilisateurs, groupes, machines, GPO, OU, domaines) et les **arêtes** (droits/relations). Une élévation = un chemin d'arêtes du nœud qu'on contrôle jusqu'à *Domain Admins*.

---

## 1. Deux versions

| Version | Notes |
| --- | --- |
| **BloodHound CE** (Community Edition) | Actuelle, web + Docker, remplace l'ancienne. Collecteur **SharpHound** |
| **BloodHound Legacy** | Ancienne app Electron + Neo4j en direct. Encore vue dans les tutos |

> Vérifier la **compatibilité collecteur ↔ interface** (format de données CE ≠ Legacy).

---

## 2. Installer (BloodHound CE)

```bash
# Le plus simple : Docker Compose fourni par le projet
curl -L https://ghst.ly/getbhce | docker compose -f - up
# UI : http://localhost:8080  (mot de passe initial affiché dans les logs)
```
Ou via `pipx install bloodhound-ce` selon la distribution.

---

## 3. Collecter les données

### Depuis Linux (sans toucher à une machine Windows)
```bash
# bloodhound-python (collecteur distant)
bloodhound-python -u user -p 'pass' -d DOMAINE -ns <DC_IP> -c All --zip

# via NetExec
nxc ldap <DC> -u user -p pass --bloodhound --collection-method All --dns-server <DC>
```

### Depuis Windows (SharpHound)
```powershell
# Exécutable
SharpHound.exe -c All --zip
# ou script PowerShell
Import-Module .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Temp
```

**Méthodes de collecte (`-c`)** :
| Valeur | Récupère |
| --- | --- |
| `All` | Presque tout (défaut recommandé) |
| `DCOnly` | Uniquement via LDAP au DC — **discret**, pas de contact des postes |
| `Session` | Sessions ouvertes (qui est connecté où) |
| `LoggedOn` | Sessions (méthode privilégiée) |
| `ACL` | Droits (DACL) |
| `Group`, `LocalAdmin`, `Trusts`, `Container` | Ciblé |

> **DCOnly** = collecte furtive de base ; **All** contacte les machines (Session/LocalAdmin) donc plus bruyant.

### Import
Glisser le `.zip` (ou les `.json`) dans l'UI BloodHound.

---

## 4. Lire un graphe — le vocabulaire

- **Nœuds** : User, Group, Computer, GPO, OU, Domain, Container, Cert (ADCS).
- **Arêtes** (relations) — les plus importantes :

| Arête | Signification | Débouché |
| --- | --- | --- |
| `MemberOf` | Appartenance à un groupe | Hérite des droits du groupe |
| `AdminTo` | Admin local d'une machine | Exécution de code, dump |
| `HasSession` | Session ouverte sur une machine | Vol de credentials si on l'a en admin |
| `GenericAll` | Contrôle total sur l'objet | Reset MDP, ajout au groupe, etc. |
| `GenericWrite` | Écriture d'attributs | Targeted Kerberoast, script logon |
| `WriteDacl` | Modifier les droits | S'octroyer GenericAll |
| `WriteOwner` | Devenir propriétaire | Puis WriteDacl |
| `ForceChangePassword` | Réinitialiser le MDP | Prise de contrôle du compte |
| `AddMember` | Ajouter au groupe | S'ajouter à un groupe privilégié |
| `AllowedToDelegate` | Délégation Kerberos | Usurpation |
| `DCSync` (GetChanges + GetChangesAll) | Répliquer les secrets | Extraction krbtgt/hashes |
| `CanRDP` / `CanPSRemote` | Accès distant | Mouvement latéral |

> Clic droit sur une arête → **Help** : BloodHound explique l'abus **et** la remédiation. À lire systématiquement.

---

## 5. Les requêtes pré-enregistrées à connaître

Dans l'onglet requêtes (Cypher / Pre-built) :

- **Shortest Paths to Domain Admins** — la question n°1.
- **Find Principals with DCSync Rights**
- **List all Kerberoastable Accounts**
- **Find AS-REP Roastable Users**
- **Shortest Path from Owned Principals** (après avoir marqué tes accès).
- **Find Computers where Domain Users are Local Admin**
- **Find Workstations where Domain Users can RDP**

> **Marquer « Owned »** : clic droit sur un nœud contrôlé → *Mark as Owned*. Ensuite « Shortest path from owned » calcule tes chemins réels.

---

## 6. Requêtes Cypher utiles (personnalisées)

Le langage de Neo4j. Bases :

```cypher
// Tous les utilisateurs Kerberoastables
MATCH (u:User {hasspn:true}) RETURN u

// Utilisateurs sans pré-auth (AS-REP)
MATCH (u:User {dontreqpreauth:true}) RETURN u

// Chemin le plus court d'un utilisateur vers Domain Admins
MATCH p=shortestPath((u:User {name:"USER@DOMAINE"})-[*1..]->(g:Group))
WHERE g.name STARTS WITH "DOMAIN ADMINS"
RETURN p

// Qui est admin local de quoi
MATCH p=(u)-[:AdminTo]->(c:Computer) RETURN p

// Comptes à privilèges avec une session ouverte quelque part
MATCH (u:User)-[:MemberOf*1..]->(g:Group)
WHERE g.name CONTAINS "ADMIN"
MATCH p=(c:Computer)-[:HasSession]->(u) RETURN p

// Chemins depuis les comptes marqués "Owned"
MATCH p=shortestPath((u {owned:true})-[*1..]->(t:Group {highvalue:true})) RETURN p
```

> `highvalue:true` = nœuds sensibles marqués par BloodHound (DA, Enterprise Admins, DC…).

---

## 7. Exploiter un chemin (renvoi vers ad-attacks.md)

Le graphe **désigne** l'attaque ; ad-attacks.md donne la **commande** :

| Ce que montre le chemin | Attaque à mener |
| --- | --- |
| `hasspn:true` sur un compte | Kerberoasting (`-m 13100`) |
| `dontreqpreauth:true` | AS-REP roasting (`-m 18200`) |
| `ForceChangePassword` → user | `net rpc password` |
| `AddMember` / `GenericWrite` sur groupe | S'ajouter au groupe (bloodyAD) |
| `WriteDacl` sur objet | S'octroyer GenericAll puis exploiter |
| `DCSync` | `secretsdump` |
| `AdminTo` / `CanPSRemote` | PtH / wmiexec (smb.md) |

---

## 8. Côté défense — lire BloodHound en Blue Team

BloodHound sert aussi à **auditer** son propre AD :
- Repérer les **chemins courts** vers DA à casser en priorité.
- Traquer les **ACL aberrantes** (GenericAll/WriteDacl sur des objets sensibles).
- Réduire les **AdminTo** trop larges (Domain Users admin local partout).
- Outils voisins d'audit : **PingCastle**, **ORADAD** (état de durcissement chiffré).

> **Détection de la collecte** : pic de requêtes LDAP, énumération de sessions massive (SharpHound). Une collecte `All` est bruyante ; `DCOnly` beaucoup moins.

---

## 9. À savoir expliquer à l'oral (module 5)

- **Ce que modélise BloodHound** : un graphe nœuds/arêtes, où une élévation est un chemin.
- **Lire un chemin** : partir du nœud contrôlé, suivre les arêtes, nommer l'abus à chaque saut.
- **DCOnly vs All** : le compromis discrétion/complétude de la collecte.
- **Le réflexe « Mark as Owned »** puis *shortest path from owned* pour raisonner sur ses accès réels.
- **BloodHound côté défense** : prioriser les chemins vers DA à supprimer.
