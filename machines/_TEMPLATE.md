# <NOM_MACHINE> — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | <Linux / Windows + version> |
| **Difficulté** | <Easy / Medium> |
| **Date** | <AAAA-MM-JJ> |
| **Vecteur** | <foothold> → <privesc>  (résumé en une ligne) |
| **CVE** | <CVE-XXXX-XXXX ou "aucune / misconfig"> |
| **Tags** | <web · smb · sudo · SeImpersonate · AS-REP …> |

> Remplacer `<IP_CIBLE>` par l'IP de ta session (change à chaque reset).
> Mon IP VPN (`tun0`) = `10.10.14.x` · Cible HTB = `10.10.10.x` / `10.129.x.x`.

---

## TL;DR

<La chaîne complète en 2-3 phrases, de la recon au root. C'est le résumé que tu reliras avant l'entretien.>

**<recon> → <foothold : comment on entre, en tant que quel utilisateur> → <privesc : comment on passe root/SYSTEM>.**

---

## 1. Reconnaissance

```bash
nmap -sC -sV -p- -oA nmap/<nom> <IP_CIBLE>
# si besoin : nmap -sU --top-ports 20 <IP_CIBLE>
```

Ports notables :

| Port | Service | Version | Piste |
| --- | --- | --- | --- |
| | | | |

> Question de méthode : *quelle surface est la plus prometteuse, et pourquoi ?*

Énumération par service (selon les ports) :
```bash
# Web    : whatweb / gobuster / ffuf, robots.txt, en-têtes
# SMB    : nxc smb <IP> -u '' -p '' --shares ; enum4linux-ng -A <IP>
# autre  : ...
```

---

## 2. Accès initial (foothold)

**Vecteur :** <injection / creds trouvés / upload / exploit web / …>

```bash
# commandes exactes de la prise de pied
```

- **En tant que quel utilisateur j'atterris :** `<user>` (⚠️ noter : ce n'est PAS root/SYSTEM)
- **Stabilisation du shell :**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'   # puis Ctrl+Z, stty raw -echo, fg
# ou reverse shell propre (voir fiche reverse-shells)
```

**Flag user :** `<chemin>` (non publié — règle HTB)

> Fuite d'info ≠ exploitation : bien distinguer ce qui donne un shell de ce qui donne juste de la lecture.

---

## 3. Énumération post-accès

Dérouler la checklist AVANT d'escalader (voir fiches `linux-privesc` / `windows-privesc`).

```bash
# Linux
sudo -l ; id ; find / -perm -4000 -type f 2>/dev/null ; getcap -r / 2>/dev/null
# transférer et lancer linPEAS / pspy

# Windows
whoami /priv ; whoami /all
# transférer et lancer winPEAS / PowerUp ; Seatbelt
```

Ce que l'énumération révèle : <la piste retenue et pourquoi>

---

## 4. Élévation de privilèges

**Piste :** <sudo GTFOBins / SUID / cron / SeImpersonate-Potato / service mal configuré / kernel / AS-REP → DCSync …>

```bash
# commandes exactes de la privesc
```

**Résultat :** `<root / nt authority\system>`

```bash
id    # ou : whoami  → uid=0(root) / nt authority\system
```

**Flag root :** `<chemin>` (non publié)

> Pourquoi cette piste fonctionne (le mécanisme, pas juste la commande) :
> <explication — c'est ça qu'on te demandera à l'oral>

---

## 5. (si AD) Chaîne Active Directory

```
énumération → BloodHound (chemin) → attaque (AS-REP/Kerberoast/ACL) → DCSync → DA
```
- Chemin BloodHound retenu : <...>
- Requête / outil : <impacket-GetNPUsers / GetUserSPNs / secretsdump …>

---

## 6. Remédiation

- <correctif 1 — concret, orienté défense>
- <correctif 2>
- <durcissement transverse : moindre privilège, MAJ, politique MDP, segmentation…>

---

## 7. Côté Blue Team — détection

| Étape de l'attaque | Trace / Event | Détection |
| --- | --- | --- |
| foothold (<...>) | | |
| privesc (<...>) | | |

> Objectif module 9 : pour chaque attaque que je sais mener, savoir quelle trace elle laisse.

---

## 8. Leçons

- <ce que cette machine apprend de neuf par rapport aux précédentes>
- <piège rencontré / temps passé / ce que je referais différemment>
- <technique à ajouter à une cheatsheet ?>

---

## Références

- CVE — <lien NVD>
- <write-up officiel HTB / 0xdf / autre>
- Fiche(s) outil liée(s) : [<outil>](../outils/<outil>.md)
