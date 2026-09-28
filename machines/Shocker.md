# Shocker — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux (Ubuntu) |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-__-__ |
| **Vecteur** | <à compléter : foothold web → privesc sudo> |
| **CVE** | CVE-2014-6271 (famille « Shellshock ») |
| **Tags** | web · cgi · shellshock · sudo · gtfobins |

> Remplacer `<IP_CIBLE>` par l'IP de ta session (change à chaque reset).
> Mon IP VPN (`tun0`) = `10.10.14.x` · Cible HTB = `10.10.10.x` / `10.129.x.x`.

> 🎯 **Objectif d'apprentissage de cette box** : ta **1re vraie privesc en deux temps**.
> Tu entres comme utilisateur **non privilégié**, puis tu escalades. Résiste au réflexe
> « un exploit = root » des machines précédentes : ici, *foothold* ≠ *root*.

---

## TL;DR

<À remplir en fin de box, en 2-3 phrases : de la recon à root.>

**<recon> → <foothold : quel service, quelle faille, quel utilisateur> → <privesc : quel mécanisme>.**

---

## 1. Reconnaissance

```bash
# Scan complet (le réflexe)
nmap -sC -sV -p- -oA nmap/shocker <IP_CIBLE>
```

Ports notables :

| Port | Service | Version | Piste |
| --- | --- | --- | --- |
| | | | <à remplir d'après ton scan> |

> Deux ports seulement, dont un service web et un accès distant sur un port **inhabituel**.
> Question de méthode : lequel des deux est la surface d'attaque, et lequel te servira
> *après* avoir trouvé des identifiants ?

---

## 2. Énumération web

La page par défaut ne dit rien → il faut **découvrir le contenu caché**.

```bash
# Brute force de répertoires
gobuster dir -u http://<IP_CIBLE>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40
# ou : feroxbuster -u http://<IP_CIBLE>/

# Une fois un répertoire "exécutable" trouvé, chercher les SCRIPTS qui s'y trouvent
# (pense aux extensions de scripts serveur : .sh, .cgi, .pl ...)
gobuster dir -u http://<IP_CIBLE>/<repertoire>/ -w <wordlist> -x sh,cgi,pl
```

> Indice de méthode : un répertoire dont le **nom** évoque l'exécution de scripts côté serveur
> est LA piste. Trouve le script qu'il contient — c'est lui la porte d'entrée.

Trouvé : `______________________`

---

## 3. Accès initial (foothold)

**Faille : Shellshock (CVE-2014-6271).** À comprendre AVANT d'exploiter :

- Bash < 4.3 interprète du code placé **après une définition de fonction** dans une
  variable d'environnement.
- Un script CGI **passe des en-têtes HTTP dans l'environnement** du shell qui l'exécute.
- Donc : injecter la charge dans un en-tête (typiquement `User-Agent` ou `Referer`) →
  le serveur exécute ta commande. C'est une **injection de commande via l'environnement**.

Étapes (à toi de construire la charge) :
```bash
# 1) Vérifier la vulnérabilité : faire exécuter une commande simple (id / ping vers toi)
#    en injectant dans l'en-tête d'une requête vers le script CGI.
#    Outils au choix : curl -H "User-Agent: () { :; }; <cmd>" , nmap http-shellshock, ou Burp Repeater.

# 2) Se mettre à l'écoute
nc -lvnp 4444

# 3) Déclencher un reverse shell (voir ta fiche reverse-shells) via l'en-tête injecté.
```

- **Utilisateur obtenu :** `______` ⚠️ (PAS root — c'est le compte du service web)
- **Stabilisation du shell :**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'   # puis Ctrl+Z, stty raw -echo; fg, export TERM=xterm
```

**Flag user :** répertoire home de l'utilisateur (non publié).

> Rappel méthode : `nmap --script http-shellshock` sait détecter la faille — mais
> **construis la requête à la main au moins une fois** (Burp/curl) pour comprendre l'injection.

---

## 4. Énumération post-accès

Dérouler la checklist `linux-privesc` AVANT d'escalader. Le premier réflexe ici :

```bash
sudo -l          # que puis-je lancer en root SANS mot de passe ?
id ; whoami
# si sudo -l ne donne rien : find / -perm -4000 2>/dev/null ; getcap -r / 2>/dev/null ; crontab -l
```

Résultat de `sudo -l` : `______________________`

---

## 5. Élévation de privilèges

**Piste : abus d'un binaire autorisé en `sudo` (GTFOBins).**

- `sudo -l` révèle un binaire lançable en root sans mot de passe.
- Va voir **[GTFOBins](https://gtfobins.github.io/)** → cherche ce binaire → section **`sudo`**.
- La plupart des interpréteurs (perl, python, ruby, ...) peuvent lancer un shell : appelés
  en root, ils te donnent un **shell root**.

```bash
# Forme générale (adapter au binaire trouvé, cf. GTFOBins)
sudo <binaire> -e 'exec "/bin/sh";'      # exemple pour perl
```

**Résultat attendu :**
```bash
id   # uid=0(root)
```

**Flag root :** `/root/` (non publié).

> Pourquoi ça marche (à savoir dire à l'oral) : `sudo` exécute le binaire **en root** ;
> si ce binaire peut lancer une commande arbitraire (interpréteur, éditeur, pager...),
> l'utilisateur hérite d'un shell root. C'est une **mauvaise configuration de sudoers**,
> pas une CVE.

---

## 6. Remédiation

- **Corriger Shellshock** : mettre Bash à jour (patchs 2014). Ne plus exposer de CGI Bash.
- **Durcir sudoers** : retirer le `NOPASSWD` sur un interpréteur ; appliquer le moindre privilège
  (pas de binaire « shell-capable » en sudo).
- Retirer les scripts CGI inutiles ; WAF / filtrage des en-têtes suspects.

---

## 7. Côté Blue Team — détection

| Étape | Trace / observation | Détection |
| --- | --- | --- |
| Shellshock | En-tête HTTP contenant `() { :; };` dans les logs Apache/access.log | Règle WAF / Sigma sur ce motif ; T1190 (exploit public-facing app) |
| Reverse shell | `www-data`/service web ouvrant une connexion sortante | EDR : process web → `nc`/`bash -i` ; T1059 |
| privesc sudo | `sudo` vers un interpréteur | auditd : `type=USER_CMD` ; alerte sur sudo d'un binaire shell-capable |

---

## 8. Leçons

- 1re box où **foothold ≠ root** : bien séparer les deux étapes dans mes notes.
- Shellshock = injection de commande **via l'environnement** (en-têtes CGI), pas via un champ.
- Réflexe privesc n°1 : **`sudo -l`**, puis GTFOBins.
- <ce que j'ajouterais à une cheatsheet / piège rencontré / temps passé>

---

## Références

- CVE-2014-6271 (Shellshock) — <https://nvd.nist.gov/vuln/detail/CVE-2014-6271>
- GTFOBins — <https://gtfobins.github.io/>
- Fiches liées : [burp](../outils/burp.md) · [reverse-shells](../outils/reverse-shells.md) · [linux-privesc](../outils/linux-privesc.md) · [gtfobins](../outils/gtfobins.md)
