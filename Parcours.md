# Formation Pentest — Red & Blue Team
 
Parcours de remise à niveau pour un poste de responsable pentest en environnement industriel / défense.
 
---
 
## Mode d'emploi
 
Objectif : être opérationnel en entretien et en mission de pentest en 12 semaines, à raison de 8 à 10 h par semaine à côté du poste actuel. Chaque module suit le même cycle : 1 soirée de théorie, 2 soirées de labs, 1 séance le week-end de rédaction (notes + mini-rapport).
 
| Semaines | Modules | Livrable de fin de bloc |
| --- | --- | --- |
| 1–2 | 1. Fondamentaux · 2. Reconnaissance | Fiche méthodo recon + 5 machines faciles |
| 3–4 | 3. Web · 4. Exploitation système | Rapport sur une appli web (Juice Shop / DVWA) |
| 5–6 | 5. Active Directory · 6. Élévation de privilèges | Compromission d'un lab AD de A à Z |
| 7–8 | 7. Post-exploitation · 8. Reporting | Rapport de pentest complet, niveau client |
| 9–10 | 9. Blue Team · 10. Industriel / naval | Règles de détection pour tes propres attaques |
| 11–12 | 11. Piloter une équipe pentest · préparation entretien | Dossier de candidature Naval Group |
 
**Environnement à monter (semaine 1)** :
 
- Un poste hôte avec 32 Go de RAM idéalement (16 Go minimum), VirtualBox ou Proxmox.
- Kali Linux (attaquant), une VM Windows Server 2019/2022 (contrôleur de domaine), deux Windows 10/11 clients, une Ubuntu serveur.
- Wazuh ou Elastic + Sysmon pour la partie Blue Team (module 9).
- Un dépôt GitHub privé `pentest-notes` : une note Markdown par machine, par technique et par outil. C'est ta base de connaissances, et une preuve de travail à montrer.
**Plateformes** : TryHackMe (parcours guidés, idéal pour reprendre), HackTheBox (machines plus réalistes, Pro Labs pour l'AD), Root-Me (francophone, bien vu des recruteurs français), PortSwigger Web Security Academy (gratuit, la référence web).
 
**Règle d'or** : n'attaque que tes labs et les plateformes autorisées. Hors cadre contractuel, un test d'intrusion est une infraction (article 323-1 du Code pénal).
 
---
 
## Module 1 — Cours réseau approfondi
 
Ce cours te donne le socle réseau attendu d'un pentester : comprendre exactement ce qui circule, sur quel port, sous quelle forme, pour savoir quoi observer et quoi tester. Objectif : pouvoir expliquer chaque notion à l'oral en entretien.
 
### 1. Les modèles en couches
 
On utilise surtout le modèle TCP/IP (4 couches), en gardant OSI (7 couches) comme vocabulaire de référence.
 
| Couche TCP/IP | Couches OSI | Rôle | Exemples | Unité |
| --- | --- | --- | --- | --- |
| Accès réseau | Physique + Liaison | Transmettre sur le lien local, adressage MAC | Ethernet, Wi-Fi, ARP | Trame |
| Internet | Réseau | Adresser et router entre réseaux | IP, ICMP | Paquet |
| Transport | Transport | Acheminer de bout en bout, fiabilité | TCP, UDP | Segment |
| Application | Session + Présentation + Application | Service rendu à l'utilisateur | HTTP, DNS, SMB | Message |
 
À retenir : une donnée est encapsulée de haut en bas, puis décapsulée à l'arrivée. Une attaque vise une couche précise : ARP spoofing en couche liaison, scan de ports en transport, injection SQL en application.
 
### 2. Adressage IP
 
- **IPv4** : 32 bits, 4 octets (ex. 192.168.10.25). Un masque sépare la partie réseau de la partie hôte.
- **Notation CIDR** : /24 = 255.255.255.0 = 254 adresses utilisables. /25 = 128, /26 = 64, /27 = 32.
- **Adresses privées (RFC 1918)** : 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16.
- **Adresses particulières** : 127.0.0.1 (loopback), 0.0.0.0, broadcast, passerelle (souvent .1).
- **IPv6** : 128 bits ; connaître au moins les adresses lien-local fe80::/10, souvent actives et oubliées.
### 3. Le cheminement d'un paquet
 
1. Comparaison de l'IP de destination à la table de routage. Même sous-réseau → livraison directe ; sinon → passerelle.
2. Résolution MAC via **ARP** (mise en cache → d'où l'ARP spoofing).
3. Chaque routeur lit l'IP de destination, décrémente le TTL, réachemine.
4. **NAT** : remplacement de l'IP privée par l'IP publique en sortie.
Outils : `ip route`, `traceroute`/`tracert`, `arp -a`.
 
### 4. TCP et UDP
 
- **TCP** : orienté connexion, fiable. Handshake : SYN → SYN-ACK → ACK. Drapeaux : SYN, ACK, FIN, RST, PSH, URG.
- **UDP** : sans connexion, rapide (DNS, DHCP, SNMP, VoIP).
- **Ports** : 0–1023 réservés, 1024–49151 enregistrés, 49152+ éphémères.
Lien avec le scan : `nmap -sS` envoie un SYN → SYN-ACK = ouvert, RST = fermé, rien = filtré.
 
### 5. Ports et services à connaître par cœur
 
| Port | Service | Intérêt en test |
| --- | --- | --- |
| 21 | FTP | Accès anonyme, identifiants en clair |
| 22 | SSH | Accès distant, attaques par identifiants |
| 25 / 465 / 587 | SMTP | Relais ouvert, énumération d'utilisateurs |
| 53 | DNS | Transfert de zone, exfiltration par DNS |
| 80 / 443 | HTTP / HTTPS | Surface web, la plus riche en failles |
| 88 | Kerberos | Authentification Active Directory |
| 110 / 143 | POP3 / IMAP | Messagerie, identifiants |
| 135 / 139 / 445 | RPC / NetBIOS / SMB | Partages Windows, cœur des attaques AD |
| 389 / 636 | LDAP / LDAPS | Annuaire, énumération d'objets |
| 1433 | MSSQL | Base SQL Server |
| 3306 | MySQL | Base de données |
| 3389 | RDP | Bureau à distance Windows |
| 5985 / 5986 | WinRM | Administration distante Windows |
 
### 6. Services d'infrastructure
 
- **DNS** : A, AAAA, MX, CNAME, TXT (SPF/DKIM), NS. En recon : sous-domaines, transfert de zone (AXFR), enregistrements TXT.
- **DHCP** : échange DORA (Discover, Offer, Request, Ack). Un DHCP pirate impose une passerelle malveillante.
- **NAT / PAT** : partage d'une IP publique par le port.
### 7. Segmentation et filtrage
 
- **VLAN** : découpage logique ; mal conçu → VLAN hopping.
- **Pare-feu** : filtrage stateful, logique « tout interdit sauf autorisé ».
- **DMZ** : segment des services exposés, isolé de l'interne.
- **Proxy / reverse proxy** : intermédiaire sortant / entrant.
### 8. Chiffrement des flux
 
- **TLS** : négociation, certificat, chiffrement. Signaler versions obsolètes (SSLv3, TLS 1.0/1.1), suites faibles, certificats expirés.
- **SSH** : préférer l'authentification par clé.
- **VPN** : IPsec ou SSL/TLS.
### 9. Travaux pratiques
 
1. Capture Wireshark : handshake TCP, résolution DNS, échange HTTP en clair.
2. `nmap -sS -sV -p- <cible>` puis lecture de chaque port.
3. Cache ARP (`arp -a`) avant/après.
4. Calcul à la main d'un /26 et d'un /27.
Auto-évaluation : décrire le trajet d'un paquet, nommer 15 ports, expliquer le handshake TCP et son lien avec le scan SYN.
 
---
 
## Module 2 — Reconnaissance et énumération
 
La reconnaissance détermine la qualité de tout le test : une cible non découverte n'est jamais testée.
 
### 1. Passive vs active
 
- **Passive** : aucune interaction directe, sources tierces. Discrète.
- **Active** : envoi de paquets vers la cible. Plus riche, mais visible dans les journaux.
Ordre : passive d'abord, puis active dans le périmètre autorisé.
 
### 2. OSINT (recon passive)
 
- **DNS et sous-domaines** : crt.sh, amass, subfinder.
- **Moteurs de recherche** : Google dorking (site:, filetype:, intitle:).
- **Fuites de code** : GitHub, GitLab (clés d'API, mots de passe).
- **Empreinte humaine** : LinkedIn (format des identifiants, organigramme, technos).
- **Bases spécialisées** : Shodan, Censys, haveibeenpwned.
### 3. Découverte d'hôtes
 
- `nmap -sn 10.0.0.0/24` (ping sweep). ICMP parfois bloqué → combiner avec des sondes sur ports courants.
### 4. Scan de ports et de services
 
| Option nmap | Ce qu'elle fait |
| --- | --- |
| `-sS` | Scan SYN, rapide et discret (le standard) |
| `-sT` | Scan TCP complet |
| `-sU` | Scan UDP (DNS, SNMP, DHCP) |
| `-sV` | Détection de version |
| `-O` | Empreinte de l'OS |
| `-p-` | Les 65535 ports |
| `-sC` | Scripts NSE par défaut |
| `-oA` | Sauvegarde 3 formats |
 
Référence : `nmap -sS -sV -sC -p- -oA scan_cible <IP>`.
 
### 5. Énumération par service
 
- **SMB (445)** : partages, sessions nulles, version. Outils : smbclient, enum4linux-ng, NetExec.
- **LDAP (389)** : objets, utilisateurs, groupes (ldapsearch).
- **HTTP (80/443)** : technologies (whatweb), répertoires (gobuster, ffuf), robots.txt, en-têtes.
- **SNMP (161/UDP)** : chaînes de communauté par défaut.
- **DNS (53)** : transfert de zone (AXFR).
- **FTP (21)** : connexion anonyme.
### 6. La méthode : tout tracer
 
| IP | Port | Service | Version | Piste | Statut |
| --- | --- | --- | --- | --- | --- |
| .25 | 445 | SMB | Windows Server 2019 | session nulle ? | à tester |
 
### 7. Travaux pratiques
 
1. Carte des sous-domaines d'un domaine autorisé, en passif.
2. Scan de référence + tableau par machine.
3. Énumération d'un partage SMB avec enum4linux-ng.
Auto-évaluation : passer d'une IP à un tableau complet de services, versions et pistes.
 
---
 
## Module 3 — Sécurité des applications web
 
Le web est la surface d'attaque la plus riche en audit.
 
### 1. Comment fonctionne une application web
 
- **HTTP** : méthodes (GET, POST, PUT, DELETE), en-têtes, corps, codes (200, 301, 403, 404, 500).
- **État et sessions** : cookie ou jeton.
- **Cookies** : HttpOnly, Secure, SameSite.
- Ton atout Selenium/Scrapy : tu manipules déjà requêtes, cookies et sessions.
### 2. L'outil central : Burp Suite
 
- **Proxy** : intercepte et modifie.
- **Repeater** : rejoue à la main (travaille tout ici avant d'automatiser).
- **Intruder** : automatise (fuzzing).
- **Decoder / Comparer** : encode/décode, compare.
### 3. L'OWASP Top 10 (2021)
 
| Catégorie | Idée | À rechercher |
| --- | --- | --- |
| A01 Contrôle d'accès défaillant | Accéder à ce qui n'est pas autorisé | IDOR, forcer une URL |
| A02 Défauts cryptographiques | Données mal protégées | HTTP en clair, TLS obsolète |
| A03 Injection | Donnée interprétée comme du code | SQL, commandes, LDAP |
| A04 Conception non sécurisée | Faille de logique métier | étapes contournables |
| A05 Mauvaise configuration | Réglages dangereux | pages d'admin, erreurs verbeuses |
| A06 Composants vulnérables | Bibliothèques obsolètes | versions faillées |
| A07 Authentification | Identité mal vérifiée | mots de passe faibles |
| A08 Intégrité | Données/maj non vérifiées | désérialisation |
| A09 Journalisation | On ne voit pas l'attaque | absence de logs |
| A10 SSRF | Le serveur requête une cible interne | URL contrôlée |
 
### 4. Les failles à maîtriser
 
Pour chacune : détection, démonstration d'impact propre, correctif.
 
- **Injection SQL** : UNION, aveugle booléenne/temporelle → requêtes paramétrées. Outil : sqlmap (mais savoir le faire à la main).
- **XSS** : réfléchie, stockée, DOM → échappement en sortie, CSP.
- **CSRF** : jeton anti-CSRF, SameSite.
- **IDOR** : contrôle d'accès côté serveur.
- **LFI/RFI et traversal**, **upload de fichiers**, **injection de commandes**, **SSRF**, **failles JWT/sessions**.
### 5. Méthodologie de référence
 
**OWASP WSTG** (Web Security Testing Guide) : la checklist reconnue.
 
### 6. Travaux pratiques
 
1. PortSwigger : labs Apprentice, puis moitié des Practitioner.
2. OWASP Juice Shop en local : 5 failles de catégories différentes.
3. Mini-rapport : criticité CVSS, preuve, correctif.
Auto-évaluation : expliquer une injection SQL et une XSS (détection, impact, correctif).
 
---
 
## Module 4 — Exploitation système et réseau
 
Passer d'une vulnérabilité identifiée à un accès, de façon contrôlée. Pratique uniquement sur labs autorisés.
 
### 1. De la vulnérabilité à l'exploitation
 
- **CVE** + score **CVSS**.
- **Recherche** : à partir de la version d'un service (searchsploit).
- **Règle d'or** : ne jamais lancer un exploit public à l'aveugle. On le lit, comprend, adapte, évalue le risque.
### 2. Notions clés de vulnérabilités système
 
- **Buffer overflow** : principe, protections (ASLR, DEP/NX, canaris).
- **Mauvaise configuration** : services exposés, droits larges, identifiants par défaut.
- **Composants obsolètes** : première cause d'intrusion réelle.
### 3. Metasploit
 
- Modules (exploit, auxiliary, post), payloads, options (RHOSTS, LHOST, LPORT), Meterpreter.
- Pratiquer aussi **sans** Metasploit (exigence OSCP).
### 4. Les shells
 
- **Reverse** (cible → attaquant) vs **bind** (cible en écoute).
- Stabilisation (TTY), transfert de fichiers.
### 5. Attaques sur les identifiants
 
- **En ligne** : Hydra (lent, verrouillages).
- **Hors ligne** : John, Hashcat, listes de mots, règles.
- Hachage vs chiffrement : un bon hachage est lent et salé.
### 6. Le pivot
 
- Tunnels SSH (-L, -D), Chisel, ligolo-ng. Relais vers un réseau interne → montre l'impact réel.
### 7. Cadre et éthique
 
- Autorisation écrite, périmètre, actions interdites (DoS, données de production). Tout tracé et horodaté.
### 8. Travaux pratiques
 
1. TryHackMe « Offensive Pentesting ».
2. 5 machines HTB Easy/Medium (Linux + Windows).
3. Cassage d'empreintes de test (Hashcat).
Auto-évaluation : reverse vs bind shell ; pourquoi on lit un exploit avant de le lancer.
 
---
 
## Module 5 — Active Directory
 
Dans un grand groupe industriel, AD est la cible n°1 d'un test interne.
 
### 1. Le modèle Active Directory
 
- Forêt, domaine, OU ; groupe **Domain Admins** = clé du royaume.
- GPO, relations d'approbation (trusts), comptes de service et SPN.
### 2. Authentification : NTLM et Kerberos
 
- **NTLM** : relais, Pass-the-Hash.
- **Kerberos** : tickets (TGT puis TGS) via le DC.
### 3. Énumération
 
- **BloodHound** : chemins d'attaque vers les comptes à privilèges.
- **PowerView, NetExec, ldapsearch**.
### 4. Familles d'attaques
 
| Attaque | Principe | Remédiation clé |
| --- | --- | --- |
| Password spraying | Un mot de passe courant sur beaucoup de comptes | Politique MDP, MFA, verrouillage |
| AS-REP roasting | Comptes sans pré-auth Kerberos | Activer la pré-authentification |
| Kerberoasting | Ticket de service cassé hors ligne | MDP de service longs, gMSA |
| Relais NTLM | Rejouer une authentification | Signature SMB/LDAP, désactiver NTLM |
| Abus d'ACL | Droits mal attribués | Revue des ACL |
| Pass-the-Hash / Ticket | Réutiliser empreinte/ticket | Tiering, comptes protégés |
| DCSync | Se faire passer pour un DC | Restreindre les droits de réplication |
| ADCS (ESC1–ESC8) | Abus de modèles de certificats | Durcir l'autorité de certification |
 
### 5. Durcissement d'un AD
 
- Tiering, LAPS, comptes protégés, désactiver LLMNR/NBT-NS, signature SMB, gMSA.
- Références : guides ANSSI AD, PingCastle / ORADAD.
### 6. Travaux pratiques
 
1. Lab **GOAD** ou TryHackMe « Compromising Active Directory ».
2. Lecture d'un chemin BloodHound.
3. Rédaction de 5 remédiations, ton RSSI.
Auto-évaluation : expliquer le Kerberoasting et sa remédiation ; lire un chemin BloodHound.
 
---
 
## Module 6 — Élévation de privilèges
 
Compétence de méthode : une checklist déroulée dans le même ordre à chaque fois.
 
### 1. Principe
 
Escalade verticale (plus de droits) ou horizontale (autre compte de même niveau).
 
### 2. Linux — points à vérifier
 
| Piste | À regarder |
| --- | --- |
| `sudo -l` | Commandes/binaires détournables (GTFOBins) |
| SUID/SGID | Droits du propriétaire |
| Capabilities | Droits fins sur un binaire |
| Cron | Scripts modifiables |
| PATH | Binaire appelé sans chemin absolu |
| NFS | `no_root_squash` |
| Noyau | Version vulnérable (dernier recours) |
 
Outils : linPEAS, pspy.
 
### 3. Windows — points à vérifier
 
| Piste | À regarder |
| --- | --- |
| Privilèges du jeton | SeImpersonate (familles « Potato ») |
| Services | Chemins non entre guillemets, droits faibles |
| AlwaysInstallElevated | MSI en SYSTEM |
| Identifiants stockés | Fichiers, registre |
| DLL hijacking | Chemin modifiable |
| Tâches planifiées | Actions modifiables |
 
Outils : winPEAS, PowerUp, Seatbelt.
 
### 4. La méthode
 
1. Énumérer d'abord, exploiter ensuite.
2. Une checklist personnelle, même ordre à chaque fois.
3. Noter le temps par machine.
4. Comprendre pourquoi une piste fonctionne (rapport + correctif).
### 5. Côté défense
 
Retirer les SUID inutiles, corriger les chemins de services, moindre privilège, mises à jour.
 
### 6. Travaux pratiques
 
1. TryHackMe « Linux PrivEsc » et « Windows PrivEsc ».
2. 10 machines HTB (piste + correctif).
3. Checklist personnelle Linux et Windows.
Auto-évaluation : dérouler la checklist et justifier la piste et sa remédiation.
 
---
 
## Module 7 — Post-exploitation et démonstration d'impact
 
Montrer ce qu'un attaquant atteindrait vraiment. Cadre strictement autorisé.
 
### 1. Objectif
 
Démontrer l'**impact métier** sans dommage. On collecte des preuves, pas du butin.
 
### 2. Ce qu'on évalue
 
- Collecte d'identifiants (sources → recommander leur protection).
- Mouvement latéral (WinRM, PsExec, WMI, RDP).
- Accès aux données sensibles (capture, pas exfiltration).
### 3. Persistance (et son nettoyage)
 
Tout mécanisme posé est noté et **supprimé** en fin de mission ; le rapport le confirme.
 
### 4. Command and Control (C2) — notion
 
Frameworks (Sliver, Havoc, Cobalt Strike). Différence pentest (couverture) vs Red Team (furtivité, objectif, détection).
 
### 5. Cartographie MITRE ATT&CK
 
Chaque action rattachée à une technique (ex. T1021). Pont avec la Blue Team.
 
### 6. Règles d'engagement
 
Périmètre, fenêtres, actions interdites, contact d'urgence, RGPD.
 
### 7. Travaux pratiques
 
1. Mouvement latéral documenté sur le lab AD.
2. 5 actions → technique MITRE ATT&CK.
3. Mini « règle d'engagement » type.
Auto-évaluation : pentest vs Red Team ; relier une action à sa technique ATT&CK et sa détection.
 
---
 
## Module 8 — Rapport et restitution
 
Le rapport est le seul livrable que le client garde. Compétence la plus discriminante pour un responsable.
 
### 1. Structure d'un rapport
 
1. Synthèse managériale (1 page, sans jargon).
2. Périmètre et méthodologie (PTES, OWASP WSTG).
3. Constats détaillés.
4. Plan de remédiation (priorité + effort).
5. Annexes (chemins, outils, journal horodaté).
### 2. Modèle d'un constat
 
| Champ | Contenu |
| --- | --- |
| Titre | Court et parlant |
| Criticité | CVSS 3.1/4.0 + niveau |
| Description | Le problème |
| Preuve | Capture, requête/réponse |
| Impact métier | Ce que ça change |
| Recommandation | Le correctif concret |
| Références | CVE, OWASP, ANSSI |
 
### 3. Coter la criticité : CVSS
 
Score de base vs environnemental (contexte client). Ne pas sur-coter.
 
### 4. Bien écrire un constat
 
Factuel, orienté impact métier, actionnable, priorisé.
 
### 5. La restitution orale
 
Présentation direction (risque, priorités, budget) + session technique. Préparer : « c'est grave ? », « combien de temps ? », « par quoi commencer ? ».
 
### 6. Automatiser avec Python (ton atout)
 
Squelette Markdown → PDF (Pandoc) ou Word (python-docx). Un générateur maison = argument concret.
 
### 7. Travaux pratiques
 
1. Rapport complet du lab AD.
2. Coter 5 constats en CVSS.
3. Modèle + script Python de génération.
Auto-évaluation : un rapport compris en une page par un dirigeant et applicable par une équipe technique.
 
---
 
## Module 9 — Blue Team : détecter ce que tu sais attaquer
 
La vraie valeur d'un profil Red + Blue.
 
### 1. Journalisation
 
| ID Windows | Événement |
| --- | --- |
| 4624 / 4625 | Connexion réussie / échouée |
| 4768 / 4769 | Tickets Kerberos (TGT / TGS) |
| 4672 | Connexion à privilèges élevés |
| 4688 | Création de processus |
| 7045 | Installation d'un service |
 
Sysmon (Windows), auditd/syslog (Linux).
 
### 2. SIEM et détection
 
- SIEM : Wazuh (gratuit), Elastic, Splunk.
- Règles Sigma (format générique).
- Threat hunting guidé par MITRE ATT&CK.
### 3. La démarche Purple Team
 
1. Rejouer une attaque (modules 5–7).
2. Voir ce que le SIEM détecte.
3. Écrire/ajuster la règle.
4. Rejouer pour confirmer.
Outil : Atomic Red Team.
 
### 4. Réponse à incident
 
Cycle NIST : préparation, détection/analyse, confinement, éradication, rétablissement, retour d'expérience. Forensic : Velociraptor, Volatility.
 
### 5. Durcissement
 
Guides ANSSI (42 mesures, AD, Linux), CIS Benchmarks.
 
### 6. Le livrable qui vend ton profil
 
| Attaque | Trace | Détection | Remédiation |
| --- | --- | --- | --- |
| Kerberoasting | 4769 chiffrement RC4 | Règle Sigma | gMSA, MDP longs |
 
### 7. Travaux pratiques
 
1. Wazuh + Sysmon sur le lab.
2. 10 techniques (Atomic Red Team) → tableau.
3. Deux règles Sigma.
Auto-évaluation : pour toute attaque, dire quelle trace elle laisse et comment la détecter.
 
---
 
## Module 10 — Systèmes industriels et navals
 
Ton plus grand différenciateur, grâce à tes 9 ans en production.
 
### 1. OT vs IT
 
- **IT** : priorité confidentialité, mises à jour fréquentes.
- **OT** : priorité **disponibilité et sûreté** ; systèmes anciens, peu mis à jour.
- Conséquence : **test passif d'abord** (un scan peut arrêter un automate).
### 2. Composants d'un SI industriel
 
PLC (automates), SCADA / IHM, Historian, réseaux de terrain.
 
### 3. Le modèle de Purdue
 
Niveaux 0 à 5. Base de la **ségrégation** IT / OT via une DMZ industrielle.
 
### 4. Protocoles industriels
 
- Modbus, S7, OPC UA, Profinet : souvent **non authentifiés**.
- Protocoles marins : NMEA 0183 / 2000.
- La défense repose sur segmentation et surveillance.
### 5. Normes et cadre réglementaire
 
- **IEC 62443**, guides ANSSI systèmes industriels, recommandations OMI (maritime).
- Cadre défense : Diffusion Restreinte, informations classifiées, opérateur d'importance vitale, **habilitation défense** (souvent requise).
### 6. Tester en OT : les règles
 
Non-interruption prioritaire, test passif, fenêtres planifiées, souvent sur une réplique (banc de test).
 
### 7. Ton atout à formuler en entretien
 
« Je sais ce que coûte un arrêt de production et pourquoi on ne scanne pas un automate en aveugle. »
 
### 8. Travaux pratiques
 
1. TryHackMe « ICS/SCADA ».
2. OpenPLC + simulateur Modbus (Wireshark).
3. Guide ANSSI « Maîtriser la SSI pour les systèmes industriels » → 5 mesures clés.
Auto-évaluation : pourquoi l'OT privilégie la disponibilité ; le modèle de Purdue ; pourquoi tester passivement.
 
---
 
## Module 11 — Piloter une équipe pentest
 
Là où tes 9 ans de management compensent une expérience pentest plus courte.
 
### 1. Le cycle d'une mission
 
1. Expression de besoin et cadrage.
2. Convention d'audit (autorisation écrite, périmètre, règles).
3. Planification.
4. Exécution des tests.
5. Rédaction du rapport.
6. Restitution (direction + technique).
7. Contre-audit après correction.
### 2. Le référentiel PASSI
 
PASSI (ANSSI) : exigences pour les prestataires d'audit qualifiés. Une équipe interne s'en inspire souvent.
 
### 3. Piloter par le risque
 
- Plan d'audit annuel selon la criticité.
- Suivi des vulnérabilités jusqu'à correction.
- Indicateurs : délai de correction par criticité, taux de re-test, couverture.
- Ton atout data : tableau de bord Power BI / SQL.
### 4. Manager l'équipe
 
Grille de compétences, montée en compétence, relecture croisée des rapports, gestion de prestataires externes.
 
### 5. Cadre juridique et éthique
 
Autorisation écrite obligatoire (article 323-1), confidentialité, RGPD, traçabilité.
 
### 6. Faire le pont entre technique et direction
 
Traduire des constats techniques en risques métier et priorités budgétaires.
 
### 7. Travaux pratiques
 
1. Plan d'audit annuel fictif d'un site industriel.
2. Tableau de bord de suivi des vulnérabilités (Power BI).
3. Grille de compétences type.
Auto-évaluation : présenter en 5 minutes l'organisation et le pilotage d'une équipe pentest sur un an, indicateurs à l'appui.
 
---
 
## Certifications, entretien et checklist
 
| Certification | Éditeur | Ce qu'elle prouve | Quand |
| --- | --- | --- | --- |
| eJPT | INE | Bases du pentest, examen pratique | Semaine 4 |
| PNPT | TCM Security | Pentest externe + AD + restitution | Semaine 10–12 |
| CRTP | Altered Security | Attaque AD en profondeur | Après le module 5 |
| OSCP | OffSec | La référence des offres pentest | 6 à 9 mois |
| BTL1 | Security Blue Team | Compétences Blue Team | En option |
 
Vérifie les tarifs sur le site de chaque éditeur ; demande un financement employeur / France Travail (CPF selon éligibilité).
 
### Questions d'entretien à préparer
 
1. Déroule un pentest interne de A à Z.
2. Compromettre un domaine AD sans identifiants ? Et l'empêcher ?
3. Pentest vs audit de configuration vs Red Team.
4. Adapter un test sur un réseau industriel en production.
5. Vulnérabilité critique le premier jour : que fais-tu ?
6. Comment mesures-tu la performance d'une équipe pentest ?
7. Un auditeur techniquement bon mais illisible : comment gères-tu ?
8. Pourquoi passer de la data/production à la cyber ? (réponse de 2 minutes reliant les trois)
### Checklist avant de postuler
 
- [ ] Lab personnel monté (Kali, AD, SIEM)
- [ ] 30 machines résolues avec notes publiables
- [ ] PortSwigger : labs Apprentice terminés
- [ ] eJPT obtenue, PNPT ou OSCP planifiée
- [ ] Un rapport de pentest complet anonymisé, prêt à montrer
- [ ] Tableau attaque → détection de 10 techniques
- [ ] GitHub nettoyé : scripts d'automatisation d'audit en Python
- [ ] Profil LinkedIn repositionné cyber (titre, résumé, certifications)
- [ ] CV mis à jour
- [ ] Réponses aux 8 questions répétées à voix haute
 
