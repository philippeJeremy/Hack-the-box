# Cheatsheets — Index

Aide-mémoires opérationnels pour le parcours pentest (Red & Blue Team). Une fiche par outil/thème, à garder dans le dépôt `pentest-notes`. Chaque fiche suit le même plan : commandes → recettes → **côté défense** → **à savoir expliquer à l'oral**.

> ⚠️ **Rappel légal (vaut pour toutes les fiches).** N'attaque que **tes labs** et les **plateformes autorisées** (TryHackMe, HackTheBox, PortSwigger Academy, Root-Me). Hors cadre contractuel écrit, un test d'intrusion est une infraction — **art. 323-1 du Code pénal**. En mission : autorisation écrite, périmètre défini, tout tracé et horodaté, mécanismes posés retirés en fin de test.

---

## Les fiches par phase

| # | Fiche | Phase | Module(s) | En une ligne |
| --- | --- | --- | --- | --- |
| 1 | [nmap](outils/nmap.md) | Reconnaissance | 1–2 | Découverte d'hôtes, scan de ports/services, NSE |
| 2 | [smb](outils/smb.md) | Recon / AD | 2, 5 | Énumération SMB (NetExec), partages, PtH, relais NTLM |
| 3 | [gobuster](outils/gobuster.md) | Web / Recon | 3 | Brute-force dossiers, vhosts, sous-domaines DNS |
| 4 | [burp](outils/burp.md) | Web | 3 | Proxy, Repeater, Intruder, OWASP Top 10 |
| 5 | [searchsploit](outils/searchsploit.md) | Exploitation | 4 | Trouver, lire et adapter un exploit public |
| 6 | [metasploit](outils/metasploit.md) | Exploitation | 4 | Framework, msfvenom, Meterpreter, pivot |
| 7 | [reverse-shells](outils/reverse-shells.md) | Exploitation | 4 | One-liners reverse/bind, stabilisation TTY |
| 8 | [impacket](outils/impacket.md) | Exploitation / AD | 4, 5 | Exécution distante, secretsdump, Kerberos, relais |
| 9 | [bloodhound](outils/bloodhound.md) | Active Directory | 5 | Cartographie AD, chemins d'attaque, Cypher |
| 10 | [ad-attacks](outils/ad-attacks.md) | Active Directory | 5 | Kerberoasting, AS-REP, DCSync, ADCS + remédiation |
| 11 | [ad-memo](outils/ad-memo.md) | Active Directory | 5 | **Mémo commandes AD consolidé** : toute la chaîne, prêt à copier |
| 12 | [linux-privesc](outils/linux-privesc.md) | Élévation | 6 | Checklist Linux : sudo, SUID, cron, capabilities |
| 13 | [windows-privesc](outils/windows-privesc.md) | Élévation | 6 | Checklist Windows : jeton, services, tâches |
| 14 | [gtfobins](outils/gtfobins.md) | Élévation | 6 | Binaires détournables (sudo/SUID/capabilities) |
| 15 | [lire-peas](outils/lire-peas.md) | Élévation | 6 | Lire la sortie linPEAS / winPEAS sans se noyer |
| 16 | [hashcat](outils/hashcat.md) | Cassage | 4, 5 | Modes, règles, masques — casse ce qu'on récupère |
| 17 | [file-transfer](outils/file-transfer.md) | Transverse | 4, 6 | Déposer un outil / exfiltrer : HTTP, SMB, certutil, nc, scp |
| 18 | [commandes-utiles](outils/commandes-utiles.md) | Transverse | tous | Réflexes Linux & Windows : chercher un fichier, lire, réseau, droits |
| 19 | [citrix](outils/citrix.md) | Accès / Breakout | — | Énumération et breakout d'environnement Citrix |
| 20 | [kubernetes](outils/kubernetes.md) | Cloud / Conteneurs | — | Énumération et abus d'un cluster k8s |
| 21 | [rhel-idm](outils/rhel-idm.md) | AD / IDM Linux | 5 | FreeIPA / Red Hat IdM : l'équivalent AD côté Linux |
| 22 | [wazuh-sysmon](outils/wazuh-sysmon.md) | Blue Team | 9 | Lab SIEM maison : détecter tes propres attaques (Purple Team) |

> Chemins relatifs depuis la racine du dépôt. Les fiches machines (`machines/`) pointent vers `../outils/<fiche>.md`.

---

## Progression — machines HackTheBox

Une note par machine dans `machines/`, à partir du modèle [`machines/_TEMPLATE.md`](machines/_TEMPLATE.md).
Toutes les machines ci-dessous sont **retirées** (retired) : write-ups disponibles, cadre 100 % légal.

### ✅ Faites

**Bloc 1 — « 1 CVE = root direct » (prise de contact)**

| Machine | OS | Diff. | Vecteur | Ce que ça a musclé |
| --- | --- | --- | --- | --- |
| [Lame](machines/Lame.md) | Linux | 🟢 Easy | Samba usermap → root | 1 CVE = root direct |
| [Legacy](machines/Legacy.md) | Windows | 🟢 Easy | MS17-010 / MS08-067 → SYSTEM | 1 CVE mémoire = SYSTEM |
| [Blue](machines/Blue.md) | Windows | 🟢 Easy | EternalBlue (MS17-010) → SYSTEM | 1 CVE mémoire = SYSTEM |
| [Netmon](machines/Netmon.md) | Windows | 🟢 Easy | FTP leak → PRTG RCE (CVE-2018-9276) | 1re **chaîne** logique |

**Bloc 2 — privesc réelle en deux temps (foothold user → escalade)**

| Machine | OS | Diff. | Vecteur | Ce que ça a musclé |
| --- | --- | --- | --- | --- |
| [Shocker](machines/Shocker.md) | Linux | 🟢 Easy | Shellshock (CGI) → `sudo perl` (GTFOBins) | 1re privesc Linux réelle |
| [Optimum](machines/Optimum.md) | Windows | 🟢 Easy | HFS RCE → Potato / MS16-032 | 1re privesc Windows réelle |
| [Cap](machines/Cap.md) | Linux | 🟢 Easy | IDOR → creds dans un `.pcap` → capability `setuid` | Web (A01) + capabilities |
| [Connected](machines/Connected.md) | Linux | 🟢 Easy | FreePBX SQLi → RCE asterisk → `modprobe.d` inscriptible | 🟡 **user OK, root en cours** |

**Bloc 3 — Active Directory (cœur du poste visé)**

| Machine | OS | Diff. | Vecteur | Ce que ça a musclé |
| --- | --- | --- | --- | --- |
| [Forest](machines/Forest.md) | Windows | 🟢 Easy | AS-REP roast → BloodHound → DCSync | 1re compromission **AD** |
| [Sauna](machines/Sauna.md) | Windows | 🟢 Easy | AS-REP → creds autologon → DCSync | 2e chaîne AD complète |
| [Active](machines/Active.md) | Windows | 🟢 Easy | GPP `cpassword` → Kerberoast du compte admin (SPN) | GPP + Kerberoasting |
| [Support](machines/Support.md) | Windows | 🟢 Easy | LDAP info leak (`.exe` .NET) → RBCD → admin | Décompilation + RBCD |
| [Timelapse](machines/Timelapse.md) | Windows | 🟢 Easy | `.pfx` cracké → WinRM par certificat → lecture **LAPS** | Cert auth + LAPS |
| [Cascade](machines/Cascade.md) | Windows | 🟡 Medium | LDAP leak → déchiffrement `.NET` (XOR) → AD Recycle Bin | Reversing + Recycle Bin |
| [Resolute](machines/Resolute.md) | Windows | 🟡 Medium | Password spray → creds dans des notes → **DnsAdmins** (DLL) → SYSTEM | DnsAdmins → SYSTEM |
| [Certified](machines/Certified.md) | Windows | 🟡 Medium | Chaîne ACL (owner→DACL→shadow creds) → ADCS **ESC9** | ACL abuse + ADCS ESC9 |
| [Escape](machines/Escape.md) | Windows | 🟡 Medium | Coercition MSSQL (`xp_dirtree`) → NetNTLMv2 → ADCS **ESC1** | 1re box ADCS complète |
| [Blackfield](machines/Blackfield.md) | Windows | 🔴 Hard | AS-REP roast → ForceChangePassword → **SeBackupPrivilege** → NTDS | Privilège de backup → DC |

**Lab local**

| Lab | OS | Vecteur | Livrable |
| --- | --- | --- | --- |
| [GOAD](machines/GOAD.md) | Windows (multi) | Lab AD multi-machines, mouvement latéral | Compromission d'un lab AD de A à Z (module 5) |

> **Décompte candidature : ~17 machines avec notes publiables** (+ Connected en cours). Objectif : **30**.

### 🎯 À faire (skeletons prêts — je cherche les commandes seul)

| Machine | OS | Diff. | Vecteur attendu | Ce que ça va muscler |
| --- | --- | --- | --- | --- |
| [Authority](machines/Authority.md) | Windows | 🟡 Medium | Ansible Vault cracké → PWM → ADCS (ESC1) | 2e box ADCS, angle différent |
| [Codify](machines/Codify.md) | Linux | 🟢 Easy | Évasion de sandbox **vm2** (Node.js) → sudo script | Sandbox escape + script sudo |
| [Usage](machines/Usage.md) | Linux | 🟢 Easy | SQLi (Laravel) → upload → wildcard **7z** | SQLi + wildcard injection |
| [GreenHorn](machines/GreenHorn.md) | Linux | 🟢 Easy | Git leak (Gitea) → RCE CMS → **depix** | Fuite de source + depix |
| [Builder](machines/Builder.md) | Linux | 🟡 Medium | Jenkins **CVE-2024-23897** (lecture de fichiers) → secret | Lecture arbitraire + déchiffrement secret |

> **Règle de progression** : sur chaque machine, dérouler la **checklist privesc AVANT d'exploiter**, chronométrer, et pour chaque étape savoir dire *pourquoi ça marche* (mécanisme + remédiation, pour le rapport).

En parallèle (module 3) : finir les labs **PortSwigger Apprentice**, puis la moitié des **Practitioner**.

---

## Chaînes de lecture

Les fiches se répondent. Parcours à lire dans l'ordre :

**Chaîne AD (test interne)**
```
nmap → smb → bloodhound → ad-attacks → ad-memo → impacket → hashcat
(découvrir → énumérer → cartographier → attaquer → dérouler les commandes → exécuter/dumper → casser)
```

**Chaîne exploitation → accès → privesc**
```
nmap → searchsploit → metasploit / reverse-shells → lire-peas → privesc (linux|windows + gtfobins)
```

**Chaîne web**
```
nmap → gobuster → burp  (avec sqlmap/ffuf en complément à venir)
```

**Transverse (sur toutes les box)**
```
commandes-utiles (chercher/lire/réseau/droits) + file-transfer (déposer/exfiltrer)
```

---

## À compléter (pistes)

- **sqlmap.md** / **ffuf.md** — automatisation web (module 3) ← prioritaire pour Usage/GreenHorn
- **john.md** — extraction de hashes (`*2john`) en complément de hashcat
- **pivoting.md** — tunnels SSH, chisel, ligolo-ng, proxychains (module 9) ← avant Dante Pro Lab
- **adcs.md** — fiche dédiée ESC1→ESC16 (pour l'instant couvert par `ad-memo` + Escape/Certified)
- **wireshark.md** — analyse de trafic (module 1)
- **mitre-attack.md** — mapping des techniques (module 7, pour le rapport)

---

## Rappel : où pratiquer légalement

TryHackMe (guidé), HackTheBox (réaliste + Pro Labs AD), Root-Me (FR, bien vu des recruteurs français), PortSwigger Web Security Academy (web, gratuit), lab perso GOAD pour l'AD.
