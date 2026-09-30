# Cheatsheets — Index

Aide-mémoires opérationnels pour le parcours pentest (Red & Blue Team). Une fiche par outil/thème, à garder dans le dépôt `pentest-notes`. Chaque fiche suit le même plan : commandes → recettes → **côté défense** → **à savoir expliquer à l'oral**.

> ⚠️ **Rappel légal (vaut pour toutes les fiches).** N'attaque que **tes labs** et les **plateformes autorisées** (TryHackMe, HackTheBox, PortSwigger Academy, Root-Me). Hors cadre contractuel écrit, un test d'intrusion est une infraction — **art. 323-1 du Code pénal**. En mission : autorisation écrite, périmètre défini, tout tracé et horodaté, mécanismes posés retirés en fin de test.

---

## Les fiches par phase

| # | Fiche | Phase | Module(s) | En une ligne |
| --- | --- | --- | --- | --- |
| 1 | [nmap](nmap.md) | Reconnaissance | 1–2 | Découverte d'hôtes, scan de ports/services, NSE |
| 2 | [smb](smb.md) | Recon / AD | 2, 5 | Énumération SMB (NetExec), partages, PtH, relais NTLM |
| 3 | [burp](burp.md) | Web | 3 | Proxy, Repeater, Intruder, OWASP Top 10 |
| 4 | [searchsploit](searchsploit.md) | Exploitation | 4 | Trouver, lire et adapter un exploit public |
| 5 | [metasploit](metasploit.md) | Exploitation | 4 | Framework, msfvenom, Meterpreter, pivot |
| 6 | [reverse-shells](reverse-shells.md) | Exploitation | 4 | One-liners reverse/bind, stabilisation TTY |
| 7 | [impacket](impacket.md) | Exploitation / AD | 4, 5 | Exécution distante, secretsdump, Kerberos, relais |
| 8 | [bloodhound](bloodhound.md) | Active Directory | 5 | Cartographie AD, chemins d'attaque, Cypher |
| 9 | [ad-attacks](ad-attacks.md) | Active Directory | 5 | Kerberoasting, AS-REP, DCSync, ADCS + remédiation |
| 10 | [linux-privesc](linux-privesc.md) | Élévation | 6 | Checklist Linux : sudo, SUID, cron, capabilities |
| 11 | [windows-privesc](windows-privesc.md) | Élévation | 6 | Checklist Windows : jeton, services, tâches |
| 12 | [gtfobins](gtfobins.md) | Élévation | 6 | Binaires détournables (sudo/SUID/capabilities) |
| 13 | [hashcat](hashcat.md) | Cassage | 4, 5 | Modes, règles, masques — casse ce qu'on récupère |
| 14 | [wazuh-sysmon](wazuh-sysmon.md) | Blue Team | 9 | Lab SIEM maison : détecter tes propres attaques (Purple Team) |
| 15 | [file-transfer](file-transfer.md) | Transverse | 4, 6 | Déposer un outil / exfiltrer : HTTP, SMB, certutil, nc, scp |

---

## Progression — machines HackTheBox

Une note par machine dans `machines/`, à partir du modèle [`machines/_TEMPLATE.md`](machines/_TEMPLATE.md).
Toutes les machines ci-dessous sont **retirées** (retired) : write-ups disponibles, cadre 100 % légal.

### ✅ Faites

| Machine | OS | Diff. | Vecteur | Ce que ça a musclé |
| --- | --- | --- | --- | --- |
| [Lame](machines/Lame.md) | Linux | Easy | Samba usermap → root | 1 CVE = root direct |
| [Legacy](machines/Legacy.md) | Windows | Easy | MS17-010 / MS08-067 → SYSTEM | 1 CVE mémoire = SYSTEM |
| [Blue](machines/Blue.md) | Windows | Easy | EternalBlue (MS17-010) → SYSTEM | 1 CVE mémoire = SYSTEM |
| [Netmon](machines/Netmon.md) | Windows | Easy | FTP leak → PRTG RCE (CVE-2018-9276) | 1re **chaîne** logique |

**Constat :** les 4 amènent directement en admin/SYSTEM. Deux trous à combler → une **vraie privesc en deux temps** (foothold user → escalade), et l'**Active Directory** (cœur du poste visé).

### 🎯 À faire, dans l'ordre

| Ordre | Machine | OS | Diff. | Vecteur attendu | Objectif / fiches travaillées |
| --- | --- | --- | --- | --- | --- |
| 1 | **Shocker** | Linux | Easy | Shellshock (CGI) → `sudo perl` (GTFOBins) | 1re privesc Linux réelle → `linux-privesc`, `gtfobins` |
| 2 | **Optimum** | Windows | Easy | HFS RCE → escalade jeton/kernel (Potato / MS16-032) | 1re privesc Windows réelle → `windows-privesc` |
| 3 | **Cap** | Linux | Easy | IDOR (web) → creds dans un `.pcap` → capability `setuid` | Web (A01/OWASP) + privesc → `burp`, `linux-privesc` |
| 4 | **Forest** | Windows | Easy | AS-REP roast → BloodHound → DCSync | 1re compromission **AD** → `ad-attacks`, `bloodhound`, `impacket`, `hashcat` |
| 5 | **Sauna** | Windows | Easy | AS-REP → creds autologon → DCSync | Consolider l'AD (2e chaîne complète) |
| 6 | **GOAD** (lab local) | Windows | — | Multi-machines, mouvement latéral | Livrable module 5 : compromission d'un lab AD de A à Z |

> **Règle de progression** : sur chaque machine, dérouler la **checklist privesc AVANT d'exploiter**, chronométrer, et pour chaque étape savoir dire *pourquoi ça marche* (mécanisme + remédiation, pour le rapport).

En parallèle (module 3) : finir les labs **PortSwigger Apprentice**, puis la moitié des **Practitioner**.

Objectif checklist candidature : **30 machines** avec notes publiables.

---

## Chaînes de lecture

Les fiches se répondent. Trois parcours qui se lisent dans l'ordre :

**Chaîne AD (test interne)**
```
nmap → smb → bloodhound → ad-attacks → impacket → hashcat
(découvrir → énumérer → cartographier → attaquer → exécuter/dumper → casser)
```

**Chaîne exploitation → accès**
```
nmap → searchsploit → metasploit / reverse-shells → privesc (linux|windows + gtfobins)
```

**Chaîne web**
```
nmap → burp  (avec sqlmap/ffuf en complément à venir)
```

---

## À compléter (pistes)

- **sqlmap.md** / **ffuf.md** — automatisation web (module 3)
- **john.md** — extraction de hashes (`*2john`) en complément de hashcat
- **wireshark.md** — analyse de trafic (module 1)
- **mitre-attack.md** — mapping des techniques (module 7, pour le rapport)
- **pivoting.md** — tunnels SSH, chisel, ligolo-ng, proxychains (module 9)
- **kerberoasting.md** / **delegation.md** / **adcs.md** — techniques AD avancées (module 5)

---

## Rappel : où pratiquer légalement

TryHackMe (guidé), HackTheBox (réaliste + Pro Labs AD), Root-Me (FR, bien vu des recruteurs français), PortSwigger Web Security Academy (web, gratuit), lab perso GOAD pour l'AD.
