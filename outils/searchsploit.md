# Cheatsheet — searchsploit (Exploit-DB)

Recherche hors ligne dans la base **Exploit-DB**, à partir d'un nom de service et d'une version. Le pont entre l'énumération (nmap `-sV`) et l'exploitation.

> ⚠️ **Cadre.** Un exploit ne se lance que sur tes labs ou un périmètre autorisé par écrit (art. 323-1). **Règle d'or : on ne lance jamais un exploit public à l'aveugle** — on le lit, on le comprend, on l'adapte, on évalue le risque.

---

## 1. Rappels

- **Exploit-DB** : base publique d'exploits (fournie par OffSec). `searchsploit` en est le client local, **hors ligne** (paquet `exploitdb` sur Kali).
- La base locale est dans `/usr/share/exploitdb/` (dossiers `exploits/` et `shellcodes/`).
- Le flux type : `nmap -sV` donne **service + version** → `searchsploit` cherche un exploit correspondant → on lit le code → on adapte.

---

## 2. Recherche

```bash
# Recherche simple (tous les termes doivent matcher)
searchsploit apache 2.4.49

# Un terme précis dans le titre
searchsploit vsftpd 2.3.4
searchsploit "PRTG Network Monitor"

# Exemples réels
searchsploit windows smb remote
searchsploit wordpress plugin
```

| Option | Effet |
| --- | --- |
| `-t` | Chercher uniquement dans le **titre** (moins de bruit) |
| `-e` | Correspondance **exacte** de l'expression |
| `--exclude="terme"` | Exclure des résultats (ex. `--exclude="/dos/"` pour virer les DoS) |
| `-w` | Afficher l'**URL** exploit-db.com au lieu du chemin local |
| `-j` | Sortie **JSON** (pour scripter) |
| `--cve <CVE>` | Filtrer par identifiant CVE |

> Astuce anti-bruit : chercher **large** (`searchsploit apache`), puis affiner avec `-t` et la version. Trop de termes ⇒ 0 résultat ; trop peu ⇒ 300 lignes.

---

## 3. Lire et copier un exploit

```bash
# Afficher le code / les notes d'un exploit dans le terminal
searchsploit -x windows/remote/42315.py
searchsploit -x 42315                      # par EDB-ID

# Copier l'exploit dans le dossier courant (NE PAS éditer l'original)
searchsploit -m 42315
# → 42315.py copié dans ./ ; on travaille sur la copie

# Voir le chemin complet dans la base locale
searchsploit -p 42315
```

Chaque exploit porte un **EDB-ID** (le numéro, ex. `42315`) — c'est la clé pour l'afficher (`-x`) ou le copier (`-m`).

---

## 4. Le réflexe de lecture (obligatoire avant tout lancement)

Avant d'exécuter quoi que ce soit, on lit l'en-tête et le code pour répondre à :

1. **Cible exacte** : quelle version, quel OS, quelle architecture ? (un exploit Win7 x86 ne marche pas sur x64).
2. **Type** : `remote` / `local` / `webapps` / `dos`. Un **DoS** peut faire tomber le service — jamais à l'aveugle, surtout pas en OT.
3. **Ce qu'il fait vraiment** : quelle IP/port il contacte, s'il télécharge un binaire, s'il ouvre un shell — et **où** (LHOST/LPORT à adapter).
4. **Shellcode en dur ?** Beaucoup d'exploits PoC contiennent un shellcode figé (calc.exe, ou pire). Il faut le remplacer par le sien (`msfvenom`).
5. **Dépendances** : Python 2 vs 3, modules, compilation (`gcc exploit.c -o exploit`).

> Un exploit copié-collé et lancé tel quel peut : ne rien faire, planter la cible, ou exécuter un payload inconnu sur **ta** machine. On lit **toujours** avant.

---

## 5. Maintenance de la base

```bash
searchsploit -u                 # mettre à jour la base locale
sudo apt update && sudo apt install exploitdb   # (ré)installer
```

---

## 6. Compléments utiles

```bash
# nmap-vulners / vulscan : corréler versions et CVE pendant le scan
nmap -sV --script vuln 10.0.0.10

# Recherche en ligne quand la base locale ne suffit pas
#   exploit-db.com, github (PoC récents), packetstorm
# Pour les CVE AD/Windows récentes : souvent des repos GitHub, pas Exploit-DB
```

---

## 7. Côté défense

`searchsploit` n'attaque rien en soi (recherche locale) ; ce qui laisse une trace, c'est **l'exploit lancé**. Côté bleu, la parade est en amont :

- **Inventaire et gestion des versions** : la première cause d'intrusion réelle est un **composant obsolète** (OWASP A06). Un service à jour n'a pas d'entrée Exploit-DB exploitable.
- **Veille CVE** : suivre les CVE des produits exposés, prioriser par CVSS + exposition.
- **Détection** : signatures IDS/IPS sur les payloads d'exploits connus, alertes sur crash de service (un PoC raté fait souvent redémarrer le service → Event 7031/7034).

---

## 8. À savoir expliquer à l'oral

- **Le flux version → exploit** : pourquoi `nmap -sV` conditionne tout (sans version fiable, pas de CVE ciblée).
- **Pourquoi on lit un exploit avant de le lancer** : cible/arch à valider, shellcode potentiellement piégé, risque de DoS. C'est un réflexe d'audit responsable et une question classique d'entretien.
- **`remote` vs `local`** : un exploit `local` suppose déjà un accès (souvent pour la privesc), un `remote` s'attaque au service à distance.
- **La limite d'Exploit-DB** : très bien pour les vulnérabilités « classiques » et anciennes ; pour l'AD moderne et les CVE récentes, on va chercher les PoC sur GitHub.
