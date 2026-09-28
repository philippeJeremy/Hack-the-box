# Parcours HackTheBox — montée en compétence

Ordre de machines pour progresser des bases vers l'Active Directory, calé sur les modules de la formation.
**Toutes sont des machines _retirées_** (retired) : write-ups disponibles, cadre 100 % légal. Une note par machine à partir de [`_TEMPLATE.md`](_TEMPLATE.md).

> Méthode sur chaque box : recon complète → foothold → **énumérer AVANT d'escalader** → privesc → remplir la section Blue Team (trace/détection). Chronométrer. Objectif candidature : **30 machines** avec notes publiables.

Légende difficulté : 🟢 Easy · 🟡 Medium

---

## Phase 0 — Déjà faites ✅

| Machine | OS | Vecteur | Limite |
| --- | --- | --- | --- |
| Lame | 🐧 Linux | Samba usermap → root | 1 CVE = root direct |
| Legacy | 🪟 Win | MS17-010 / MS08-067 → SYSTEM | 1 CVE mémoire |
| Blue | 🪟 Win | EternalBlue → SYSTEM | 1 CVE mémoire |
| Netmon | 🪟 Win | FTP leak → PRTG RCE | 1re chaîne, mais SYSTEM direct |

**Trou identifié** : jamais de vraie privesc en 2 temps (foothold user → escalade), et zéro AD.

---

## Phase 1 — Fondations Linux : foothold → privesc (modules 4 & 6)

Le but n'est pas le flag, c'est de dérouler `linux-privesc` + `gtfobins` pour de vrai.

| # | Machine | Diff | Foothold | Privesc | Fiches |
| --- | --- | --- | --- | --- | --- |
| 1 | **Shocker** | 🟢 | Shellshock (CGI) | `sudo perl` (GTFOBins) | linux-privesc, gtfobins |
| 2 | **Nibbles** | 🟢 | Nibbleblog (upload → RCE) | `sudo` script modifiable | linux-privesc |
| 3 | **Bashed** | 🟢 | webshell phpbash | tâche `cron` en root | linux-privesc |
| 4 | **Knife** | 🟢 | backdoor PHP 8.1 (en-tête) | `sudo knife` (GTFOBins) | gtfobins |
| 5 | **Cap** | 🟢 | IDOR web → creds dans un `.pcap` | capability `cap_setuid` (python) | burp, linux-privesc |

---

## Phase 2 — Fondations Windows : foothold → privesc (modules 4 & 6)

La famille jeton/`SeImpersonate` (« Potato ») et les services mal configurés — cœur de `windows-privesc`.

| # | Machine | Diff | Foothold | Privesc | Fiches |
| --- | --- | --- | --- | --- | --- |
| 6 | **Devel** | 🟢 | upload ASPX via FTP | JuicyPotato / kernel | windows-privesc |
| 7 | **Optimum** | 🟢 | HFS RCE (CVE-2014-6287) | MS16-032 / Potato | windows-privesc, searchsploit |
| 8 | **Jerry** | 🟢 | Tomcat manager → `.war` | (service en SYSTEM) | metasploit, reverse-shells |
| 9 | **Bounty** | 🟢 | upload `web.config` | SeImpersonate (JuicyPotato) | windows-privesc |
| 10 | **Arctic** | 🟢 | ColdFusion (LFI → RCE) | kernel / churrasco | searchsploit |

---

## Phase 3 — Web plus poussé (module 3)

En parallèle : labs **PortSwigger Apprentice** puis moitié des **Practitioner**.

| # | Machine | Diff | Thème principal | Fiches |
| --- | --- | --- | --- | --- |
| 11 | **Sense** | 🟢 | pfSense (recon + CVE) | nmap, searchsploit |
| 12 | **Blocky** | 🟢 | WordPress → réutilisation de creds → `sudo` | burp, linux-privesc |
| 13 | **Networked** | 🟢 | contournement d'upload → RCE → cron | burp, linux-privesc |
| 14 | **Cronos** | 🟡 | SQLi (bypass login) + injection de commande → cron | burp |
| 15 | **Node** | 🟡 | fuite d'API + MongoDB → privesc | burp, linux-privesc |

---

## Phase 4 — Active Directory : le cœur du poste (module 5)

C'est là que se jouent tes questions d'entretien (« compromettre un domaine sans identifiants »). Fiches `bloodhound`, `ad-attacks`, `impacket`, `hashcat` de bout en bout.

| # | Machine | Diff | Chaîne | Notion clé |
| --- | --- | --- | --- | --- |
| 16 | **Forest** | 🟢 | AS-REP roast → BloodHound → DCSync | AS-REP, DCSync |
| 17 | **Sauna** | 🟢 | AS-REP → creds autologon → DCSync | énum. utilisateurs, autologon |
| 18 | **Active** | 🟢 | GPP `Groups.xml` (cpassword) → Kerberoast | GPP, Kerberoasting |
| 19 | **Support** | 🟢 | fuite info LDAP → RBCD | délégation (RBCD) |
| 20 | **Cascade** | 🟡 | creds LDAP/VNC → corbeille AD | énum. LDAP, AD recycle bin |
| 21 | **Blackfield** | 🟡 | AS-REP → BloodHound → Backup Operators | groupe à privilèges |
| 22 | **Resolute** | 🟡 | mot de passe en description → DnsAdmins (DLL) | abus DnsAdmins |

---

## Phase 5 — Élargir : techniques variées & pivot (Medium)

Diversifier les vecteurs et attaquer le mouvement latéral.

| # | Machine | Diff | Thème |
| --- | --- | --- | --- |
| 23 | **Bastion** | 🟢 | montage VHD → creds mRemoteNG |
| 24 | **OpenAdmin** | 🟢 | OpenNetAdmin → tunnel SSH → `sudo nano` |
| 25 | **Tabby** | 🟢 | Tomcat → groupe `lxd` (évasion conteneur) |
| 26 | **Valentine** | 🟢 | Heartbleed → détournement de session tmux |
| 27 | **Traverxec** | 🟢 | nostromo (CVE) → `journalctl` (GTFOBins) |
| 28 | **SolidState** | 🟡 | James (SMTP/POP) → rbash → cron |

---

## Phase 6 — Lab AD complet & pivot (livrable module 5)

- **GOAD** (Game of Active Directory, lab local) : compromission d'un domaine multi-machines de A à Z → c'est ton **livrable de fin de bloc AD**.
- **HTB Pro Labs** quand tu seras à l'aise : **Dante** (pivot/réseau) puis **Zephyr** ou **Cerberus** (AD d'entreprise).

---

## Repère externe

La référence pour ce type de préparation (surtout si tu vises l'OSCP) est la **liste « TJnull » OSCP-like** (NetSecFocus) : machines HTB + PG classées Linux/Windows/AD. Beaucoup des boxes ci-dessus en viennent. À garder sous le coude pour élargir au-delà de ces 28.

> ⚠️ Rappel : n'attaque que ces plateformes autorisées et tes labs (art. 323-1). Flags jamais publiés dans les write-ups (règle HTB).
