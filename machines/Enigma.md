# Enigma — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | <Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | <AAAA-MM-JJ> |
| **Vecteur** | |
| **CVE** | |
| **Tags** |  |

---

## TL;DR

<La chaîne complète en 2-3 phrases, de la recon au root. C'est le résumé que tu reliras avant l'entretien.>

**<recon> → <foothold : comment on entre, en tant que quel utilisateur> → <privesc : comment on passe root/SYSTEM>.**

---

## 1. Reconnaissance

```bash
┌──(kali㉿kali)-[~/nmap/Enigma]
└─$ nmap -sC -sV -p- 10.129.239.191
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-24 14:50 +0200
Nmap scan report for 10.129.239.191
Host is up (0.051s latency).
Not shown: 65522 closed tcp ports (reset)
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp    open  http     nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://enigma.htb/
110/tcp   open  pop3     Dovecot pop3d
| ssl-cert: Subject: commonName=enigma
| Subject Alternative Name: DNS:enigma
| Not valid before: 2026-02-18T20:33:33
|_Not valid after:  2036-02-16T20:33:33
|_pop3-capabilities: UIDL PIPELINING STLS CAPA AUTH-RESP-CODE RESP-CODES TOP SASL
|_ssl-date: TLS randomness does not represent time
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100003  3,4         2049/tcp   nfs
|   100005  1,2,3      36543/udp   mountd
|   100005  1,2,3      56409/tcp   mountd
|   100021  1,3,4      44425/udp   nlockmgr
|   100021  1,3,4      44797/tcp   nlockmgr
|   100024  1          39337/tcp   status
|   100024  1          59205/udp   status
|_  100227  3           2049/tcp   nfs_acl
143/tcp   open  imap     Dovecot imapd (Ubuntu)
|_ssl-date: TLS randomness does not represent time
|_imap-capabilities: OK more have SASL-IR listed post-login Pre-login STARTTLS LOGINDISABLEDA0001 LOGIN-REFERRALS ID IMAP4rev1 capabilities ENABLE IDLE LITERAL+
| ssl-cert: Subject: commonName=enigma
| Subject Alternative Name: DNS:enigma
| Not valid before: 2026-02-18T20:33:33
|_Not valid after:  2036-02-16T20:33:33
993/tcp   open  imaps?
|_ssl-date: TLS randomness does not represent time
|_imap-capabilities: OK more have listed post-login SASL-IR ID Pre-login LOGIN-REFERRALS AUTH=PLAINA0001 IMAP4rev1 capabilities ENABLE IDLE LITERAL+
995/tcp   open  pop3s?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=enigma
| Subject Alternative Name: DNS:enigma
| Not valid before: 2026-02-18T20:33:33
|_Not valid after:  2036-02-16T20:33:33
|_pop3-capabilities: UIDL PIPELINING SASL(PLAIN) CAPA AUTH-RESP-CODE RESP-CODES TOP USER
2049/tcp  open  nfs_acl  3 (RPC #100227)
39337/tcp open  status   1 (RPC #100024)
44797/tcp open  nlockmgr 1-4 (RPC #100021)
51721/tcp open  mountd   1-3 (RPC #100005)
56409/tcp open  mountd   1-3 (RPC #100005)
60633/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 193.31 seconds
                                                               
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

## 7. Leçons

- <ce que cette machine apprend de neuf par rapport aux précédentes>
- <piège rencontré / temps passé / ce que je referais différemment>
- <technique à ajouter à une cheatsheet ?>

---

## Références

- CVE — <lien NVD>
- <write-up officiel HTB / 0xdf / autre>
- Fiche(s) outil liée(s) : [<outil>](../outils/<outil>.md)
