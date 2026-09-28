# Cheatsheet — hashcat

Cassage d'empreintes (hashes) hors ligne, accéléré par GPU. Complément direct des attaques AD et système : on casse ici ce qu'on récupère ailleurs.

> ⚠️ **Cadre.** Uniquement sur des empreintes issues de tes labs ou d'une mission autorisée. Le cassage de mots de passe d'autrui hors périmètre est illégal.

---

## 1. Anatomie d'une commande

```
hashcat -m <mode> -a <attaque> [options] <hashes> <wordlist/masque>
```

Exemple de référence :
```bash
hashcat -m 1000 -a 0 hashes.txt rockyou.txt -r best64.rule -O
```
> NTLM · attaque par dictionnaire · règles best64 · noyau optimisé.

---

## 2. Les modes d'attaque (`-a`)

| `-a` | Nom | Principe |
| --- | --- | --- |
| `0` | Straight (dictionnaire) | Une wordlist, éventuellement + règles |
| `1` | Combination | Concatène deux wordlists |
| `3` | Brute-force / masque | Explore un espace défini par un masque |
| `6` | Hybride wordlist + masque | mot + suffixe (`mot?d?d`) |
| `7` | Hybride masque + wordlist | préfixe + mot |

---

## 3. Les modes de hash (`-m`) — les plus utiles

| `-m` | Type | D'où il vient |
| --- | --- | --- |
| `0` | MD5 | Web, applis |
| `100` | SHA1 | Web, applis |
| `1400` | SHA-256 | Web, applis |
| `1000` | **NTLM** | SAM, NTDS, hashdump (Windows) |
| `5600` | **NetNTLMv2** | Responder (LLMNR/NBT-NS) |
| `13100` | **Kerberoasting** (TGS-REP) | GetUserSPNs |
| `18200` | **AS-REP roasting** | GetNPUsers |
| `1800` | sha512crypt `$6$` | `/etc/shadow` Linux moderne |
| `500` | md5crypt `$1$` | vieux Linux/Cisco |
| `3200` | bcrypt `$2*$` | applis web modernes (lent !) |
| `2100` | DCC2 (mscash2) | cache d'ouverture de session Windows |
| `22000` | WPA-PBKDF2/PMKID | Wi-Fi |
| `13400` | KeePass | fichiers `.kdbx` |
| `7500` | Kerberos AS-REQ (etype 23) | pré-auth |

```bash
# Retrouver un mode à partir d'un exemple
hashcat --example-hashes | less
hashcat -m 1000 --example-hashes
```
> Aide à l'identification : `hashid`, `hash-identifier`, ou le format du préfixe (`$6$`, `$2b$`…).

---

## 4. Attaque par dictionnaire (`-a 0`)

```bash
hashcat -m 1000 hashes.txt rockyou.txt
hashcat -m 1000 hashes.txt rockyou.txt -r rules/best64.rule
hashcat -m 1000 hashes.txt wl1.txt wl2.txt              # plusieurs wordlists
```

---

## 5. Les règles (rules) — le vrai gain

Une règle transforme chaque mot (majuscule, chiffres, `!` final, l33t…). C'est ce qui casse « Printemps2026! » à partir de « printemps ».

```bash
hashcat -m 1000 hashes.txt rockyou.txt -r rules/best64.rule
hashcat -m 1000 hashes.txt rockyou.txt -r rules/d3ad0ne.rule
hashcat -m 1000 hashes.txt rockyou.txt -r r1.rule -r r2.rule   # cumul
```

Règles fournies (dans `/usr/share/hashcat/rules/`) : `best64.rule` (rapide, bon défaut), `rockyou-30000.rule`, `d3ad0ne.rule`, `dive.rule` (énorme). Externe très utile : **OneRuleToRuleThemAll**.

---

## 6. Attaque par masque (`-a 3`)

Jeux de caractères :

| Symbole | Contenu |
| --- | --- |
| `?l` | a-z |
| `?u` | A-Z |
| `?d` | 0-9 |
| `?s` | symboles |
| `?a` | tout (l+u+d+s) |
| `?b` | 0x00-0xff |

```bash
# 8 caractères tout venant
hashcat -m 1000 hashes.txt -a 3 ?a?a?a?a?a?a?a?a
# Motif « Majuscule + 5 minuscules + 2 chiffres »
hashcat -m 1000 hashes.txt -a 3 ?u?l?l?l?l?l?d?d
# Jeu personnalisé (-1) : maj/min puis 3 libres + année
hashcat -m 1000 hashes.txt -a 3 -1 ?u?l ?1?1?1?1?d?d?d?d
--increment --increment-min 6 --increment-max 8        # longueur variable
```

---

## 7. Attaque hybride (`-a 6` / `-a 7`)

```bash
# mot du dico + 3 chiffres
hashcat -m 1000 hashes.txt -a 6 rockyou.txt ?d?d?d
# préfixe année + mot du dico
hashcat -m 1000 hashes.txt -a 7 ?d?d?d?d rockyou.txt
```

---

## 8. Options fréquentes

| Option | Effet |
| --- | --- |
| `-O` | Noyau optimisé (plus rapide, limite la longueur des mots) |
| `-w 3` | Profil de charge (1 bas → 4 max) |
| `--status` `--status-timer 10` | Affichage d'avancement régulier |
| `-o cassés.txt` | Fichier de sortie |
| `--outfile-format 2` | Format de sortie (2 = mot seul) |
| `--username` | Les hashes ont un `user:hash` en tête (cas NTDS/SAM) |
| `-p :` | Séparateur de champ personnalisé |
| `--show` | Afficher les hashes **déjà** cassés (depuis le potfile) |
| `--left` | Afficher ceux **non** cassés |
| `--session nom` | Nommer la session |
| `--restore` | Reprendre une session interrompue |
| `-i` | Mode incrémental de longueur |

Le **potfile** (`~/.local/share/hashcat/hashcat.potfile`) mémorise tout ce qui est cassé.

---

## 9. Recettes par contexte

```bash
# NTLM issus d'un dump SAM/NTDS (format user:rid:lm:ntlm:::)
hashcat -m 1000 --username ntds.txt rockyou.txt -r best64.rule -O

# NetNTLMv2 (Responder)
hashcat -m 5600 hashes.txt rockyou.txt -r best64.rule

# Kerberoasting (TGS)
hashcat -m 13100 kerb.txt rockyou.txt -r OneRuleToRuleThemAll.rule

# AS-REP roasting
hashcat -m 18200 asrep.txt rockyou.txt -r best64.rule

# Linux /etc/shadow ($6$…) — d'abord isoler le champ hash
hashcat -m 1800 shadow_hashes.txt rockyou.txt

# KeePass
keepass2john base.kdbx > kp.hash   # puis retirer le "nom:" en tête si besoin
hashcat -m 13400 kp.hash rockyou.txt
```

> `*2john` (John the Ripper) sert souvent à **extraire** le hash d'un fichier (zip, kdbx, ssh…) avant de le passer à hashcat.

---

## 10. Wordlists et préparation

- **rockyou.txt** : `/usr/share/wordlists/rockyou.txt` (décompresser le `.gz`). Le défaut universel.
- **SecLists** : collections thématiques (`/usr/share/seclists/Passwords/`).
- **Génération ciblée** :
  ```bash
  cewl https://cible.tld -m 6 -w custom.txt      # mots du site de la cible
  # + règles pour dériver les variantes
  ```
- Nettoyage/tri : `sort -u`, `awk 'length>=8'`.

---

## 11. Performance & bonnes pratiques

- **GPU** indispensable pour le brute-force sérieux ; `-O` + `-w 3/4` en lab.
- **Benchmark** : `hashcat -b -m 1000`.
- Ordre efficace : dictionnaire + règles → hybride → masque ciblé → brute-force en dernier.
- bcrypt/argon2/scrypt sont **lents par conception** : viser le dictionnaire + règles, pas le brute-force.

---

## 12. Côté défense — ce que le cassage démontre

Le fait qu'un hash tombe est un **constat de rapport** : mot de passe faible, hash rapide/non salé, RC4 sur Kerberos.

- **Empreintes robustes** : bcrypt/argon2 côté applis ; jamais de MD5/SHA1 non salé.
- **Windows/AD** : MDP longs, **gMSA** (Kerberoasting), AES au lieu de RC4, MFA.
- **Politique** : longueur ≥ 14, passphrases, bannir les mots courants, détecter la réutilisation.
- Un mot de passe « Saison+Année+! » cassé en secondes = argument concret dans la restitution.

---

## 13. À savoir expliquer à l'oral

- **Différence dictionnaire / règles / masque / hybride** et quand choisir quoi.
- **Pourquoi un bon hachage est lent et salé** (contre le cassage GPU et les rainbow tables).
- **Les 4 modes AD à connaître par cœur** : 1000 (NTLM), 5600 (NetNTLMv2), 13100 (Kerberoasting), 18200 (AS-REP).
- **hachage vs chiffrement** : le hachage est à sens unique ; on ne « déchiffre » pas un hash, on le compare.
