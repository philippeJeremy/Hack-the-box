# Cheatsheet — gobuster

Brute force de contenu web : répertoires, fichiers, sous-domaines, vhosts. Le complément de `nmap` en phase web — on découvre ce qui n'est pas lié depuis la page d'accueil.

> ⚠️ **Cadre.** Labs et périmètres autorisés uniquement (art. 323-1). Un brute force de répertoires génère **beaucoup** de requêtes et de 404 dans les logs : bruyant, à réserver au cadre d'audit.

---

## 1. Les modes

| Mode | Rôle |
| --- | --- |
| `dir` | Répertoires et fichiers d'un site (le plus courant) |
| `dns` | Sous-domaines d'un domaine |
| `vhost` | Virtual hosts (même IP, en-tête `Host` différent) |
| `fuzz` | Fuzzing libre avec un mot-clé `FUZZ` (paramètres, valeurs…) |
| `s3` / `gcs` | Buckets AWS S3 / Google Cloud |

```bash
gobuster dir   -u http://<IP>/       -w <wordlist>
gobuster dns   -d cible.tld          -w <wordlist>
gobuster vhost -u http://cible.tld   -w <wordlist> --append-domain
gobuster fuzz  -u "http://<IP>/?FUZZ=test" -w <wordlist>
```

---

## 2. Options clés (mode dir)

| Option | Effet |
| --- | --- |
| `-w` | Wordlist (obligatoire) |
| `-x` | **Extensions** à tester : `-x php,txt,bak` |
| `-t` | Threads (défaut 10 ; `-t 40` en lab) |
| `-s` / `-b` | Codes à inclure / **exclure** (blacklist, défaut `404`) |
| `-k` | Ignorer les erreurs de certificat TLS (HTTPS) |
| `-r` | Suivre les redirections |
| `-e` | Afficher l'URL complète |
| `-o` | Écrire la sortie dans un fichier |
| `-d` | (avec un motif) recherche récursive limitée |
| `-c` | Envoyer un cookie (`-c "PHPSESSID=..."`) |
| `-H` | En-tête custom (`-H "Authorization: Bearer ..."`) |
| `-a` | User-Agent custom |
| `--exclude-length` | Filtrer par longueur de réponse (vire les faux positifs) |

> Astuce faux positifs : si tout répond `200` (page « soft 404 »), filtre avec
> `-b 404` **et** `--exclude-length <taille de la page d'erreur>`, ou passe à `ffuf -fs`.

---

## 3. Quelle wordlist pour quel objectif

Chemins usuels sur Kali (paquets `dirb`, `dirbuster`, `seclists`). Installer SecLists si absent :
`sudo apt install seclists` → tout est sous `/usr/share/seclists/`.

### Répertoires & fichiers (mode `dir`)

| Objectif | Wordlist |
| --- | --- |
| **Coup d'œil rapide** | `/usr/share/wordlists/dirb/common.txt` (~4600) |
| **Le standard** (bon ratio) | `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` (~220k) |
| Version courte | `.../directory-list-2.3-small.txt` |
| SecLists — répertoires | `/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt` |
| SecLists — fichiers | `/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt` |
| SecLists — gros | `/usr/share/seclists/Discovery/Web-Content/big.txt` |
| Fichiers courants | `/usr/share/seclists/Discovery/Web-Content/common.txt` |

> Méthode : commence par `common.txt` (rapide) pour dégrossir, puis lance
> `directory-list-2.3-medium.txt` en fond pour la couverture.

### Extensions selon la techno détectée (`-x`)

Adapter d'après `nmap -sV` / `whatweb` (quelle stack tourne ?) :

| Stack | `-x` |
| --- | --- |
| PHP (Apache/Nginx) | `php,phps,txt,html,bak,zip` |
| IIS / .NET | `asp,aspx,config,txt,bak` |
| Java | `jsp,do,action,txt` |
| Général / secrets | `txt,bak,old,zip,tar.gz,conf,config,sql,log,git` |

```bash
# Exemple PHP
gobuster dir -u http://<IP>/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,bak -t 40 -o gobuster/dir.txt
```

### Fichiers spécifiques / cas particuliers

| Cible | Wordlist |
| --- | --- |
| Endpoints d'**API** | `/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt` |
| Fichiers de **sauvegarde** | `.../Web-Content/BackupFiles.wordlist` ou extensions `bak,old,~,swp` |
| CMS **WordPress** | `.../Web-Content/CMS/wordpress.fuzz.txt` |
| Panneaux d'**admin** | `.../Web-Content/AdminPanels.fuzz.txt` |
| `raft` (fichiers cachés) | `.../raft-medium-files.txt` (inclut `.git/`, `.htaccess`…) |

### Sous-domaines (mode `dns`)

| Objectif | Wordlist |
| --- | --- |
| Rapide / classique | `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt` |
| Plus large | `.../DNS/subdomains-top1million-110000.txt` |
| Très large | `.../DNS/bitquark-subdomains-top100000.txt` |

```bash
gobuster dns -d cible.tld -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50
```

### Virtual hosts (mode `vhost`)

Même listes DNS que ci-dessus. Utile quand une IP héberge plusieurs sites :
```bash
gobuster vhost -u http://cible.tld -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -t 40
```

---

## 4. Recettes prêtes à l'emploi

```bash
# 1) Découverte rapide
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirb/common.txt -t 40

# 2) Passe complète avec extensions PHP + sortie fichier
gobuster dir -u http://<IP>/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,bak,zip -t 40 -o gobuster/root.txt

# 3) Creuser un répertoire trouvé (ex. /cgi-bin/) avec des extensions de scripts
gobuster dir -u http://<IP>/cgi-bin/ -w <wordlist> -x sh,cgi,pl -t 40

# 4) HTTPS avec certif auto-signé
gobuster dir -u https://<IP>/ -w <wordlist> -k

# 5) Zone authentifiée (cookie de session)
gobuster dir -u http://<IP>/ -w <wordlist> -c "PHPSESSID=<valeur>"

# 6) Sous-domaines
gobuster dns -d cible.tld -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

> **gobuster vs ffuf/feroxbuster** : gobuster ne fait **pas** de récursivité auto en mode dir.
> Pour explorer en profondeur automatiquement, `feroxbuster` (récursif) ou `ffuf` (filtres fins
> `-fs`/`-fc`/`-fw`, fuzzing de paramètres) sont plus souples. gobuster = simple et rapide.

---

## 5. La méthode : de la découverte à la piste

1. `nmap -sV` / `whatweb` → **quelle techno** ? → choisit les extensions `-x`.
2. Passe rapide (`common.txt`) → repérer les répertoires évidents.
3. Passe large (`directory-list-2.3-medium.txt`) en fond.
4. Pour **chaque** répertoire intéressant → relancer gobuster dedans (extensions adaptées).
5. Reporter dans les notes :

| URL trouvée | Code | Taille | Intérêt |
| --- | --- | --- | --- |
| /cgi-bin/ | 403 | — | scripts ? relancer avec `-x sh,cgi` |
| /backup/ | 200 | 1.2k | fichiers `.bak` à télécharger |

---

## 6. Côté défense / détection

- **Trace** : rafales de **404** (et quelques 200/403) depuis une même IP, User-Agent `gobuster/…`, débit anormal → très visible dans `access.log`.
- **Détection** : règles WAF sur le volume de 404, alerte sur User-Agents d'outils, rate-limiting.
- **Remédiation** : ne pas exposer de répertoires/fichiers sensibles (`.git`, `.bak`, `/backup`, pages d'admin), retirer les fichiers de sauvegarde, `robots.txt` ≠ sécurité (il **révèle** des chemins), page 404 cohérente pour ne pas fuir d'info, WAF + limitation de débit.

---

## 7. À savoir expliquer à l'oral

- **Pourquoi brute-forcer les répertoires ?** Tout n'est pas lié depuis l'accueil (panneaux d'admin, sauvegardes, endpoints API) : le contenu « caché mais accessible » est une source majeure de failles (OWASP A05 mauvaise config, A01 contrôle d'accès).
- **Choisir la wordlist et les extensions selon la techno** : inutile de tester `.aspx` sur du PHP ; on ajuste d'après `nmap -sV`/`whatweb`.
- **Gérer les faux positifs** (soft 404) : filtrage par code et par longueur de réponse.
- **gobuster vs ffuf** : quand passer à ffuf (récursivité, fuzzing de paramètres, filtres fins).
- **`robots.txt`** : ce n'est pas une protection — il liste souvent les chemins que l'admin veut cacher, donc une **piste** pour l'attaquant.
