# Cheatsheet — nmap

Aide-mémoire opérationnel pour le scan réseau. **Rappel légal : n'utiliser que sur tes labs et les périmètres autorisés par écrit (article 323-1 du Code pénal).**

---

## 1. Anatomie d'une commande

```
nmap [type de scan] [options] [détection] [sortie] <cible>
```

Exemple de référence à retenir :

```bash
nmap -sS -sV -sC -p- -oA scan_cible <IP>
```

> SYN scan + versions + scripts par défaut, tous les ports, sauvegarde des 3 formats.

---

## 2. Désignation des cibles

| Syntaxe | Signification |
| --- | --- |
| `192.168.1.10` | Une IP |
| `192.168.1.10 192.168.1.20` | Plusieurs IP |
| `192.168.1.0/24` | Un sous-réseau (CIDR) |
| `192.168.1.1-50` | Une plage |
| `192.168.1.*` | Tout le dernier octet |
| `scanme.nmap.org` | Un nom d'hôte |
| `-iL cibles.txt` | Liste depuis un fichier |
| `--exclude 192.168.1.1` | Exclure une IP |
| `-iR 100` | 100 cibles aléatoires (labo perso only) |

---

## 3. Découverte d'hôtes (host discovery)

| Option | Effet |
| --- | --- |
| `-sn` | Ping sweep : découverte sans scan de ports |
| `-Pn` | Pas de ping : traiter toutes les cibles comme actives (ICMP bloqué) |
| `-PS<ports>` | Découverte par SYN sur ports donnés (ex. `-PS22,80,443`) |
| `-PA<ports>` | Découverte par ACK |
| `-PU<ports>` | Découverte par UDP |
| `-PE` | Ping ICMP echo |
| `-n` | Pas de résolution DNS (plus rapide) |
| `-R` | Forcer la résolution DNS |

```bash
# Cartographie rapide d'un sous-réseau
nmap -sn 10.0.0.0/24

# ICMP filtré ? On sonde des ports courants
nmap -sn -PS22,80,443,445 10.0.0.0/24
```

---

## 4. Types de scan de ports

| Option | Nom | Notes |
| --- | --- | --- |
| `-sS` | SYN scan | **Le standard** : rapide, semi-ouvert, discret (root requis) |
| `-sT` | TCP connect | Handshake complet, sans privilèges, plus bruyant |
| `-sU` | UDP scan | DNS, SNMP, DHCP… lent, à cibler sur des ports précis |
| `-sA` | ACK scan | Cartographier les règles de pare-feu (filtré vs non) |
| `-sN` / `-sF` / `-sX` | Null / FIN / Xmas | Contournement de filtres simples |
| `-sO` | Protocol scan | Quels protocoles IP répondent |

**Lecture des états** : `open` (service à l'écoute), `closed` (RST reçu), `filtered` (rien / bloqué par un pare-feu), `open|filtered` (indécis, fréquent en UDP).

Rappel handshake ↔ scan : SYN envoyé → `SYN-ACK` = **ouvert**, `RST` = **fermé**, silence = **filtré**.

---

## 5. Sélection des ports

| Option | Effet |
| --- | --- |
| `-p 80` | Un port |
| `-p 22,80,443` | Une liste |
| `-p 1-1000` | Une plage |
| `-p-` | **Les 65535 ports** |
| `-p U:53,T:80` | Mixer UDP et TCP |
| `-F` | Fast : les 100 ports les plus courants |
| `--top-ports 20` | Les N ports les plus fréquents |
| `-r` | Scanner dans l'ordre (pas aléatoire) |

---

## 6. Détection de services et d'OS

| Option | Effet |
| --- | --- |
| `-sV` | Version des services |
| `--version-intensity 0-9` | Effort de détection de version (9 = max) |
| `-O` | Empreinte de l'OS |
| `--osscan-guess` | OS : deviner de façon plus agressive |
| `-A` | **Agressif** : `-sV -O -sC --traceroute` d'un coup |

```bash
nmap -sV -O 192.168.1.10
```

---

## 7. Le moteur de scripts (NSE)

| Option | Effet |
| --- | --- |
| `-sC` | Scripts de la catégorie `default` |
| `--script <nom>` | Un script précis |
| `--script <catégorie>` | Une catégorie entière |
| `--script "smb-*"` | Par motif (wildcard) |
| `--script-args <args>` | Passer des arguments |
| `--script-help <nom>` | Doc d'un script |

**Catégories utiles** : `default`, `safe`, `discovery`, `auth`, `vuln`, `brute`, `exploit` (⚠️ intrusif), `malware`.

```bash
# Énumération SMB
nmap -p445 --script smb-enum-shares,smb-os-discovery 192.168.1.10

# Recherche de vulnérabilités connues
nmap -sV --script vuln 192.168.1.10

# Transfert de zone DNS (AXFR)
nmap --script dns-zone-transfer --script-args dns-zone-transfer.domain=cible.tld -p53 <DNS>
```

> Scripts par service local : `/usr/share/nmap/scripts/` — `ls /usr/share/nmap/scripts | grep smb`.

---

## 8. Rythme et furtivité (timing)

| Option | Effet |
| --- | --- |
| `-T0` … `-T5` | Modèles : 0 paranoïaque → 5 insane. **`-T4` = bon défaut en lab** |
| `--min-rate` / `--max-rate` | Paquets par seconde (plancher / plafond) |
| `--max-retries 1` | Moins de retransmissions |
| `--host-timeout 30m` | Abandonner un hôte trop lent |
| `-f` | Fragmenter les paquets |
| `-D RND:10` | Leurres (decoys) |
| `-S <IP>` | Usurper l'IP source |
| `--source-port 53` | Sortir d'un port « de confiance » |
| `--data-length 25` | Ajouter des octets aléatoires |

⚠️ En **OT / industriel** : jamais de `-T4/-T5`, jamais de `-A` à l'aveugle. Un scan agressif peut faire tomber un automate. Test passif d'abord, rythme très lent, fenêtre planifiée.

---

## 9. Formats de sortie

| Option | Effet |
| --- | --- |
| `-oN fichier` | Sortie normale (lisible) |
| `-oG fichier` | Grepable (pour `grep`/`awk`) |
| `-oX fichier` | XML (parsing, import) |
| `-oA base` | **Les 3 à la fois** (`.nmap`, `.gnmap`, `.xml`) |
| `-v` / `-vv` | Verbeux |
| `--open` | N'afficher que les ports ouverts |
| `--reason` | Pourquoi cet état (drapeau reçu) |
| `--append-output` | Ne pas écraser |

```bash
# Rejouer une sortie sans re-scanner
nmap --resume scan_cible.gnmap

# XML → HTML lisible
xsltproc scan_cible.xml -o rapport.html
```

---

## 10. Recettes prêtes à l'emploi

```bash
# 1) Balayage initial d'un réseau : qui est vivant ?
nmap -sn 10.0.0.0/24 -oA hosts

# 2) Scan complet d'une cible (le réflexe)
nmap -sS -sV -sC -p- -T4 -oA full 10.0.0.10

# 3) Scan rapide « premier coup d'œil »
nmap -sS -F -T4 10.0.0.10

# 4) Top 20 UDP (le plus rentable en UDP)
sudo nmap -sU --top-ports 20 -oA udp 10.0.0.10

# 5) Deux temps : découverte massive puis approfondissement
nmap -p- --min-rate 1000 -oA ports 10.0.0.10          # tous les ports vite
nmap -sV -sC -p<ports_trouvés> -oA deep 10.0.0.10      # détail sur les ouverts

# 6) Énumération Windows / AD
nmap -p445,139 --script smb-os-discovery,smb-enum-shares,smb-security-mode 10.0.0.10

# 7) Web
nmap -p80,443 --script http-title,http-headers,http-methods,http-enum 10.0.0.10

# 8) Contourner un ICMP bloqué
nmap -Pn -sS -sV 10.0.0.10
```

---

## 11. Méthode : de l'IP au tableau

Chaque scan alimente ta base de connaissances (`pentest-notes`). Reporte systématiquement :

| IP | Port | Service | Version | Piste | Statut |
| --- | --- | --- | --- | --- | --- |
| .10 | 445 | SMB | Windows Server 2019 | session nulle ? | à tester |
| .10 | 80 | HTTP | Apache 2.4.49 | CVE path traversal ? | à vérifier |

---

## 12. À savoir expliquer à l'oral

- **Pourquoi `-sS` plutôt que `-sT` ?** Semi-ouvert (pas de handshake complet) → plus rapide et plus discret ; nécessite les droits root pour forger les paquets.
- **Pourquoi scanner d'abord `-p-` vite, puis `-sV` sur les ports trouvés ?** La détection de version est coûteuse : on ne la lance que là où c'est utile.
- **Différence open / closed / filtered** et le drapeau TCP associé (SYN-ACK / RST / silence).
- **Pourquoi on ne scanne pas un automate en aveugle** (disponibilité prioritaire en OT).
