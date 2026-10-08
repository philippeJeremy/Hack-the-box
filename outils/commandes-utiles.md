# Cheatsheet — commandes utiles (Linux & Windows)

Le réflexe de tous les jours : **trouver un fichier, lire un contenu, se déplacer, voir le réseau, les processus, les droits**. Deux colonnes dans ta tête en permanence : *je suis sur une cible Linux* ou *je suis sur une cible Windows*. Sur Windows il y a **deux shells** — `cmd` (vieux, DOS) et **PowerShell** (le moderne) — je donne les deux quand ça diverge.

**Rappel légal : uniquement sur tes labs et les périmètres autorisés par écrit (article 323-1 du Code pénal).**

---

## 1. Chercher un fichier par son nom

### Linux
```bash
# Partout depuis la racine (le réflexe)
find / -name "user.txt" 2>/dev/null
find / -iname "*.kdbx" 2>/dev/null        # -iname = insensible à la casse

# Dans un dossier précis
find /home -name "*.txt" 2>/dev/null

# Base d'index (instantané, mais pas toujours à jour)
locate user.txt
updatedb && locate user.txt               # rafraîchir l'index d'abord

# Trouver un binaire dans le PATH
which python3
whereis nmap
```
> `2>/dev/null` masque les *Permission denied* qui noient la sortie. À avoir dans tous les `find`.

### Windows — PowerShell
```powershell
# Récursif depuis C:\  (l'équivalent de find)
Get-ChildItem -Path C:\ -Recurse -Filter "user.txt" -ErrorAction SilentlyContinue
Get-ChildItem -Path C:\Users -Recurse -Include *.txt,*.kdbx -ErrorAction SilentlyContinue

# Alias courts : gci = Get-ChildItem, ls, dir fonctionnent aussi
gci C:\ -r -fi "*.config" -ea 0
```

### Windows — cmd
```cmd
:: /s = sous-dossiers, /b = chemin nu (bare)
dir C:\*.txt /s /b
dir C:\Users\flag.txt /s /b
where /r C:\ user.txt
```

---

## 2. Chercher DANS le contenu des fichiers

### Linux — `grep`
```bash
# Récursif, insensible à la casse, avec numéro de ligne
grep -rin "password" /var/www/ 2>/dev/null
grep -rin "password" . --include="*.php"       # filtrer par extension
grep -rinE "pass(word)?|pwd|secret|api[_-]?key" /etc 2>/dev/null

# Afficher seulement les fichiers qui matchent
grep -ril "connectionstring" /opt 2>/dev/null
```
| Option | Effet |
| --- | --- |
| `-r` | récursif | 
| `-i` | insensible à la casse |
| `-n` | numéro de ligne |
| `-l` | n'afficher que le nom du fichier |
| `-E` | regex étendue (ou \|, +, ?) |
| `-o` | n'afficher que la partie qui matche |
| `-v` | inverser (lignes qui NE matchent PAS) |

### Windows — PowerShell `Select-String`
```powershell
# L'équivalent de grep -r
Select-String -Path C:\inetpub\*.config -Pattern "password"
gci C:\ -r -include *.config,*.xml,*.ini -ea 0 | Select-String "password|connectionString"

# Chaîner : trouver puis lire
gci C:\Users -r -fi web.config -ea 0 | % { Get-Content $_.FullName }
```

### Windows — cmd `findstr`
```cmd
:: /s récursif, /i insensible casse, /m noms de fichiers seulement, /n numéro de ligne
findstr /sin "password" C:\*.config
findstr /sim "password" C:\Users\*.txt
```

---

## 3. Lire / afficher un fichier

| Besoin | Linux | Windows PowerShell | Windows cmd |
| --- | --- | --- | --- |
| Tout afficher | `cat f` | `Get-Content f` / `gc f` / `cat f` | `type f` |
| Début | `head -n 20 f` | `gc f -TotalCount 20` | — |
| Fin | `tail -n 20 f` | `gc f -Tail 20` | — |
| Suivre en direct | `tail -f f` | `gc f -Wait` | — |
| Paginer | `less f` | `gc f \| more` | `more f` |
| Hex | `xxd f` / `hexdump -C f` | `Format-Hex f` | — |

---

## 4. Se déplacer & lister

| Besoin | Linux | Windows (les deux shells) |
| --- | --- | --- |
| Où suis-je | `pwd` | `pwd` (PS) · `cd` (cmd) |
| Lister | `ls -la` | `ls` / `dir` |
| Lister caché | `ls -la` | `gci -Force` · `dir /a` |
| Changer de dossier | `cd /tmp` | `cd C:\Temp` |
| Arborescence | `tree` / `find . -type d` | `tree /f` |
| Taille d'un dossier | `du -sh *` | `gci \| measure Length -sum` |

> Sur Linux les fichiers « cachés » commencent par un point (`.bash_history`, `.ssh`). Sur Windows c'est un **attribut** → `dir /a` ou `gci -Force` pour les voir.

---

## 5. Les fichiers qui valent de l'or (post-expl)

### Linux — à checker systématiquement
```bash
cat ~/.bash_history                       # historique de commandes
cat /etc/passwd                           # liste des users
ls -la ~/.ssh/                            # clés privées (id_rsa)
cat ~/.ssh/id_rsa
find / -name "*.conf" -o -name "*.config" 2>/dev/null
cat /var/www/html/*config*                # creds d'appli web
env ; cat /proc/self/environ              # variables d'environnement
sudo -l                                   # ce que je peux lancer en sudo
```

### Windows — à checker systématiquement
```powershell
# Historique PowerShell (mine d'or : mots de passe en clair)
Get-Content (Get-PSReadlineOption).HistorySavePath
gc $env:APPDATA\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt

# Fichiers « unattend » / GPP / creds
gci C:\ -r -include unattend.xml,sysprep.xml,Groups.xml -ea 0
# Variables d'environnement
Get-ChildItem Env:
# Dossiers users et bureaux
gci C:\Users\*\Desktop -ea 0
```

---

## 6. Réseau

| Besoin | Linux | Windows |
| --- | --- | --- |
| Mes IP | `ip a` / `ifconfig` | `ipconfig /all` |
| Ports en écoute | `ss -tlnp` / `netstat -tlnp` | `netstat -ano` |
| Table de routage | `ip r` | `route print` |
| Table ARP (voisins) | `ip n` / `arp -a` | `arp -a` |
| Table DNS cache | `resolvectl` | `ipconfig /displaydns` |
| Tester un port | `nc -zv IP 445` | `Test-NetConnection IP -Port 445` |
| Télécharger | `wget URL` / `curl -O URL` | `curl.exe -O URL` · `iwr URL -OutFile f` |
| Partages SMB distants | `smbclient -L //IP/` | `net view \\IP` |
| Qui est connecté | `w` / `who` | `query user` / `qwinsta` |

```bash
# Linux : port ouvert + quel process l'écoute (PID/programme)
ss -tlnp
```
```powershell
# Windows : relier un port à son process
netstat -ano | Select-String "LISTEN"
Get-Process -Id <PID>
```

---

## 7. Processus & services

| Besoin | Linux | Windows PowerShell | Windows cmd |
| --- | --- | --- | --- |
| Lister processus | `ps aux` | `Get-Process` / `ps` | `tasklist` |
| Chercher un process | `ps aux \| grep ssh` | `ps \| ? Name -like "*sql*"` | `tasklist \| findstr sql` |
| Tuer | `kill -9 PID` | `Stop-Process -Id PID` | `taskkill /PID n /F` |
| Services | `systemctl list-units --type=service` | `Get-Service` | `sc query` / `net start` |
| Détail service | `systemctl status ssh` | `Get-Service -Name W32Time` | `sc qc <nom>` |
| Tâches planifiées | `crontab -l` ; `cat /etc/crontab` | `Get-ScheduledTask` | `schtasks /query` |

---

## 8. Utilisateurs & droits

| Besoin | Linux | Windows |
| --- | --- | --- |
| Qui suis-je | `whoami` ; `id` | `whoami` ; `whoami /all` |
| Mes privilèges | `id` ; `sudo -l` | `whoami /priv` |
| Mes groupes | `groups` ; `id` | `whoami /groups` |
| Lister les users | `cat /etc/passwd` | `net user` · `Get-LocalUser` |
| Détail d'un user | `id bob` | `net user bob` |
| Lister les groupes | `cat /etc/group` | `net localgroup` |
| Membres admin | `getent group sudo` | `net localgroup Administrators` |
| Changer un mot de passe | `passwd bob` | `net user bob NewPass123!` |

```bash
# Linux : voir les droits et le propriétaire d'un fichier
ls -l fichier          # -rwxr-xr-x  owner group
stat fichier
```
```powershell
# Windows : voir les ACL d'un fichier/dossier
Get-Acl C:\chemin\fichier | Format-List
icacls C:\chemin\fichier
```

---

## 9. Permissions sur un fichier (Linux)

```bash
chmod +x script.sh          # rendre exécutable
chmod 600 id_rsa            # clé SSH : lecture/écriture owner seulement (obligatoire)
chmod 777 fichier           # tout le monde tout (à éviter en vrai)
chown bob:bob fichier       # changer le propriétaire
```
> La clé SSH volée refuse de servir si elle est trop permissive → `chmod 600 id_rsa` avant tout `ssh -i`.

| Chiffre | Droits |
| --- | --- |
| 7 | rwx (lire+écrire+exécuter) |
| 6 | rw- |
| 5 | r-x |
| 4 | r-- |
| 0 | --- |

Ordre : **propriétaire · groupe · autres**. `640` = owner rw, groupe r, autres rien.

---

## 10. Trier / filtrer une sortie (les tuyaux)

### Linux
```bash
commande | grep motif            # garder les lignes qui matchent
commande | grep -v motif         # jeter les lignes qui matchent
commande | sort | uniq           # trier + dédupliquer
commande | sort | uniq -c        # + compter les occurrences
commande | wc -l                 # compter les lignes
commande | awk '{print $1}'      # extraire la 1ʳᵉ colonne
commande | cut -d: -f1 /etc/passwd  # 1ʳᵉ colonne séparée par ':'
cat gros.txt | head -20          # limiter
```

### Windows — PowerShell
```powershell
commande | Select-String motif          # grep
commande | Where-Object { $_ -notmatch "motif" }  # grep -v
commande | Sort-Object | Get-Unique     # sort | uniq
commande | Measure-Object -Line         # wc -l
commande | Select-Object -First 20      # head
commande | ForEach-Object { $_.Split(":")[0] }  # extraire colonne
```

> **Fabriquer un users.txt depuis une sortie** (ex. noms de dossiers ou liste nxc) :
> - Linux : `... | awk '{print $1}' | sort -u > users.txt`
> - PowerShell : `gci C:\Users -Directory | select -expand Name > users.txt`

---

## 11. Encodage / décodage rapide

### Linux
```bash
echo -n "texte" | base64                 # encoder
echo "dGV4dGU=" | base64 -d              # décoder
echo -n "texte" | xxd                     # voir le hex
echo "68656c6c6f" | xxd -r -p            # hex -> ascii
echo -n "texte" | md5sum                  # hash md5
```
### Windows — PowerShell
```powershell
# base64 (attention : PowerShell encode en UTF-16 par défaut)
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("texte"))
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("dGV4dGU="))
# base64 pour une commande encodée (-EncodedCommand attend de l'UTF-16LE)
```

---

## 12. Recettes « premier réflexe » post-accès

```bash
# LINUX — bloc d'orientation à coller dès qu'on a un shell
id; hostname; uname -a
sudo -l 2>/dev/null
cat ~/.bash_history 2>/dev/null
find / -perm -4000 -type f 2>/dev/null     # binaires SUID (privesc)
ls -la /home/*/
```

```powershell
# WINDOWS — bloc d'orientation PowerShell
whoami /all
hostname; systeminfo | Select-String "OS Name","System Type"
gc (Get-PSReadlineOption).HistorySavePath -ea 0
gci C:\Users -ea 0
Get-ChildItem Env:
```

> Pour l'énumération automatisée, enchaîne avec **linPEAS / winPEAS** (cf. [`lire-peas.md`](lire-peas.md)) et, côté AD, [`ad-memo.md`](ad-memo.md).

---

## 13. À savoir expliquer à l'oral

- **`find` vs `locate`** : `find` parcourt le disque en direct (toujours à jour, lent) ; `locate` lit une base d'index (instantané, mais peut rater un fichier récent).
- **`grep -r` vs `findstr /s` vs `Select-String`** : même idée (chercher un motif dans des fichiers), trois environnements. Savoir basculer sans réfléchir.
- **Pourquoi `2>/dev/null`** : la recherche depuis `/` génère des *Permission denied* sur les dossiers interdits ; on les jette pour ne garder que les vrais résultats.
- **Pourquoi `chmod 600` sur une clé SSH** : SSH refuse une clé privée lisible par d'autres (sécurité) → `Permissions 0644 are too open`.
- **Les fichiers à creuser en priorité** : historiques shell, clés SSH, fichiers de config d'appli, variables d'environnement, binaires SUID (Linux) / historique PowerShell + GPP (Windows).

---

## Références
- Fiches liées : [file-transfer](file-transfer.md) · [linux-privesc](linux-privesc.md) · [windows-privesc](windows-privesc.md) · [lire-peas](lire-peas.md) · [ad-memo](ad-memo.md)
