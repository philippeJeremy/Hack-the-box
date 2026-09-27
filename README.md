# pentest-notes

Base de connaissances et write-ups de remise à niveau pentest (Red & Blue Team).
Chaque machine est documentée selon la structure d'un rapport d'audit :
**reconnaissance → énumération → exploitation → post-exploitation → remédiation → leçons**.

> ⚠️ Conformément aux règles des plateformes (HackTheBox…), **aucun flag n'est publié**.
> Les write-ups décrivent la méthode et le raisonnement, pas les réponses.

---

## Machines

| Machine | OS | Diff. | Vecteur | CVE | Famille |
| --- | --- | --- | --- | --- | --- |
| [Lame](machines/Lame.md) | Linux | Easy | Samba `username map script` | CVE-2007-2447 | Injection de commande |
| [Blue](machines/Blue.md) | Windows 7 | Easy | SMBv1 EternalBlue | CVE-2017-0143 | Corruption mémoire |
| [Legacy](machines/Legacy.md) | Windows XP | Easy | SMB `NetPathCanonicalize` | CVE-2008-4250 | Corruption mémoire |
| [Netmon](machines/Netmon.md) | Windows Server 2016 | Easy | Fuite config → RCE PRTG authentifié | CVE-2018-9276 | Injection de commande |

---

## Ce que couvrent ces write-ups

- **Énumération** : nmap (scan de référence, scripts NSE de détection), énumération SMB/FTP.
- **Recherche de vulnérabilité** : searchsploit, tri des exploits (RCE vs DoS), lecture avant exécution.
- **Exploitation** : Metasploit **et** voie manuelle (exigence OSCP), injection de commande, dérivation de mot de passe.
- **Post-exploitation** : obtention SYSTEM/root, psexec (Impacket), reverse shells.
- **Deux grandes familles de failles** : injection de commande (faute applicative) vs corruption mémoire (bug bas niveau).

---

## Organisation du dépôt

```
pentest-notes/
├── README.md              # ce fichier — index des machines
├── machines/              # un write-up par machine
│   ├── Lame.md
│   ├── Blue.md
│   ├── Legacy.md
│   └── Netmon.md
├── outils/                # cheatsheets par outil
│   ├── nmap.md
│   ├── smb.md
│   ├── searchsploit.md
│   ├── metasploit.md
│   ├── reverse-shells.md
│   └── gtfobins.md
└── templates/
    └── machine-template.md
```

---

## Contexte

Remise à niveau pentest structurée en modules (fondamentaux réseau, reconnaissance,
web, exploitation système, Active Directory, élévation de privilèges, post-exploitation,
reporting, Blue Team, systèmes industriels/OT). Objectif : opérationnel en mission et
en pilotage d'équipe pentest.

Les tests sont réalisés **exclusivement** sur des environnements autorisés (labs
personnels et plateformes légales). Hors cadre contractuel, un test d'intrusion est
une infraction (article 323-1 du Code pénal).
