# pentest-notes

Base de connaissances et write-ups de remise à niveau pentest (offensif — pentester).
Chaque machine est documentée selon la structure d'un rapport d'audit :
**reconnaissance → énumération → exploitation → post-exploitation → remédiation → leçons**.

> ⚠️ Conformément aux règles des plateformes (HackTheBox…), **aucun flag n'est publié**.
> Les write-ups décrivent la méthode et le raisonnement, pas les réponses.

- 📇 **Index des cheatsheets** : [`outils/` → index](index.md)
- 🗺️ **Roadmap de progression** (machines dans l'ordre) : [`machines/PARCOURS.md`](machines/PARCOURS.md)
- 🧩 **Modèle de write-up** : [`machines/_TEMPLATE.md`](machines/_TEMPLATE.md)

---

## Machines

| Machine | OS | Diff. | Vecteur | CVE / faille | Famille |
| --- | --- | --- | --- | --- | --- |
| [Lame](machines/Lame.md) | Linux | Easy | Samba `username map script` | CVE-2007-2447 | Injection de commande |
| [Blue](machines/Blue.md) | Windows 7 | Easy | SMBv1 EternalBlue | CVE-2017-0143 | Corruption mémoire |
| [Legacy](machines/Legacy.md) | Windows XP | Easy | SMB `NetPathCanonicalize` | CVE-2008-4250 | Corruption mémoire |
| [Netmon](machines/Netmon.md) | Windows Server 2016 | Easy | Fuite config → RCE PRTG authentifié | CVE-2018-9276 | Injection de commande |
| [Shocker](machines/Shocker.md) | Linux | Easy | Shellshock (CGI) → `sudo perl` | CVE-2014-6271 | Injection env. + privesc sudo |
| [Optimum](machines/Optimum.md) | Windows Server 2012 R2 | Easy | HFS 2.3 RCE → noyau MS16-032 | CVE-2014-6287 + MS16-032 | RCE web + privesc noyau |
| [Cap](machines/Cap.md) | Linux | Easy | *en cours* — IDOR → pcap → capability | misconfig (A01) | Faille logique web + privesc capability |

**Progression** : phase 1 (fondations Linux) ✅ · phase 2 (fondations Windows) ✅ · phase 3 (web) en cours · phase 4 (Active Directory) à venir. Détail dans [PARCOURS.md](machines/PARCOURS.md).

---

## Ce que couvrent ces write-ups

- **Énumération** : nmap (scan de référence, NSE), SMB/FTP, brute force web (gobuster).
- **Web** : découverte de contenu caché, IDOR / contrôle d'accès (OWASP A01), Burp.
- **Recherche de vulnérabilité** : searchsploit, repo bin-sploits, tri RCE vs DoS, lecture avant exécution.
- **Exploitation** : Metasploit **et** voie manuelle (exigence OSCP), injection de commande, RCE web.
- **Transfert de fichiers** : HTTP, SMB, certutil, netcat (déposer un outil, exfiltrer une preuve).
- **Élévation de privilèges** : checklists Linux (sudo, SUID, **capabilities**, cron) et Windows (jeton, services, tâches, **noyau**) ; lecture méthodique de winPEAS/linPEAS.
- **Post-exploitation & pivoting** : obtention SYSTEM/root, psexec/Impacket, reverse shells, tunnels (chisel/ligolo-ng).
- **Active Directory** : BloodHound, Kerberoasting/AS-REP, DCSync, délégations (RBCD), Impacket.

---

## Organisation du dépôt

```
pentest-notes/
├── README.md              # ce fichier
├── index.md               # index des cheatsheets (par phase + chaînes de lecture)
├── machines/              # un write-up par machine
│   ├── _TEMPLATE.md        # modèle (recon → foothold → énum → privesc → leçons)
│   ├── PARCOURS.md         # roadmap : machines HTB dans l'ordre de montée en compétence
│   ├── Lame.md  Blue.md  Legacy.md  Netmon.md
│   ├── Shocker.md  Optimum.md
│   └── Cap.md              # en cours
└── outils/                # cheatsheets par outil / thème
    ├── nmap.md  smb.md  gobuster.md  burp.md
    ├── searchsploit.md  metasploit.md  reverse-shells.md  impacket.md
    ├── bloodhound.md  ad-attacks.md
    ├── linux-privesc.md  windows-privesc.md  gtfobins.md  lire-peas.md
    └── hashcat.md  file-transfer.md
```

---

## Contexte

Remise à niveau pentest structurée en modules (fondamentaux réseau, reconnaissance,
web, exploitation système, Active Directory, élévation de privilèges, post-exploitation,
reporting, pivoting, systèmes industriels/OT). Objectif : opérationnel en mission de pentest.

Les tests sont réalisés **exclusivement** sur des environnements autorisés (labs
personnels et plateformes légales). Hors cadre contractuel, un test d'intrusion est
une infraction (article 323-1 du Code pénal).
