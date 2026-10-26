# GreenHorn — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-10-26 |
| **Vecteur** | Gitea source leak → hash admin Pluck cracké → Pluck RCE (CVE-2024-9405) → su junior (réutilisation mdp) → PDF pixelisé dépixelisé (Depix) → root |
| **CVE** | CVE-2024-9405 (Pluck CMS 4.7.18) |
| **Tags** | web · gitea · pluck · git-leak · rce · password-reuse · depix |

> 🎯 **Nouveautés apprises** :
> 1) **Gitea exposé** (:3000) → code source lisible → hash/mdp admin.
> 2) **Pluck CMS** : install de module = upload d'un zip PHP → RCE (CVE-2024-9405).
> 3) **Depix** : un mot de passe **pixelisé** dans un PDF est **réversible** → récupération du mdp root.

> ⚠️ **Note défense perso** : je ne mets PAS le code brut du reverse shell ici — Windows Defender le détecte (`Trojan:PHP/RevWebshell`) et **supprime le fichier**. Je garde une **référence**. Version complète = dossier exclu de l'AV (sinon : garder les payloads sur la Kali).

---

## TL;DR
1. nmap → 80 (Pluck) + **3000 (Gitea)**.
2. Gitea : dépôt source lisible → **hash admin Pluck** → cracké → `iloveyou1`.
3. Login Pluck → install de module (CVE-2024-9405) → **reverse shell PHP** → www-data.
4. `su junior` (mdp réutilisé `iloveyou1`) → user.txt + `Using OpenVAS.pdf`.
5. PDF → **mot de passe root pixelisé** → **Depix** (rasteriser + recadrer) → `su root`.

## 1. Reconnaissance
```bash
nmap -sC -sV -p- 10.129.231.80
# 22 ssh · 80 nginx (greenhorn.htb, Pluck CMS) · 3000 Gitea (Golang net/http)
# /etc/hosts : 10.129.x.x greenhorn.htb
```
> 2 surfaces : le **site Pluck** (80) et un **Gitea** (3000) qui héberge le code source du site.

## 2. Foothold — Gitea (source) → Pluck RCE
- Sur **Gitea :3000**, le dépôt du site est **public** → on lit le code source → **hash admin de Pluck** (`login.php` / fichier de conf).
- Crack du hash → mot de passe admin Pluck : **`iloveyou1`**.
- Login Pluck → `http://greenhorn.htb/admin.php?action=managemodules` (fonction **install de module** = upload d'un zip contenant du PHP).
- **CVE-2024-9405** (Pluck 4.7.18) : uploader un **reverse shell PHP** via l'install de module.
  - Payload : **php-reverse-shell (pentestmonkey)** — réf : `github.com/pentestmonkey/php-reverse-shell`. Régler `$ip = <MON_IP>` / `$port = 5555`. *(code gardé sur la Kali, pas dans cette note — AV.)*
  - `nc -lvnp 5555`, déclencher le module → shell **www-data**.

```bash
# réutilisation de mot de passe :
www-data@greenhorn:/$ su junior          # mdp : iloveyou1
junior@greenhorn:~$ ls
 user.txt  'Using OpenVAS.pdf'
```

## 3. Privesc — dépixelisation (Depix)
Le mot de passe **root** est **pixelisé** dans `Using OpenVAS.pdf` (home de junior). Exfiltration :
```bash
# box (junior) :
python3 -m http.server 8000
# Kali :
wget 'http://10.129.231.80:8000/Using OpenVAS.pdf'
```

### ⚠️ Pièges Depix (ce qui m'a bloqué longtemps)
1. Depix lit une **image PNG**, **pas un PDF** → `UnidentifiedImageError` si on lui donne le `.pdf`.
2. `pdfimages -all` ne sort qu'un **bandeau 420×15** (pas le mot de passe).
3. Pointer `images/Using_OpenVAS.pdf` (fourni dans le repo Depix) dépixelise **l'exemple de Depix** → on obtient toujours « **side from side the other side** » (texte de démo), jamais le mdp.
4. ✅ **La bonne méthode = RASTERISER la page** puis **recadrer** la zone floutée :
```bash
pdftoppm -png -r 300 'Using OpenVAS.pdf' page       # -> page-1.png (la page entière rendue)
# repérer les coords de la barre pixelisée dans un visualiseur, puis :
convert page-1.png -crop LARGEURxHAUTEUR+X+Y crop.png   # garder SEULEMENT le mot de passe flouté
python3 depix.py -p crop.png \
  -s images/searchimages/debruinseq_notepad_Windows10_spaced.png \
  -o output.png
```
→ ouvrir `output.png` : le mot de passe root se lit à l'œil.
```bash
su root        # mot de passe lu sur output.png  -> root.txt
```

## 4. Remédiation
- Ne pas exposer **Gitea / le dépôt source** (secrets dans le code).
- **Pluck** : patcher (CVE-2024-9405), restreindre l'install de modules aux admins de confiance.
- Ne **jamais pixeliser** un secret pour le masquer : c'est **réversible** (Depix) → rectangle plein opaque.
- Pas de **réutilisation de mot de passe** (admin Pluck → junior).

## 5. Leçons
- **Code source exposé = secrets exposés** : fouiller tout Git/Gitea accessible.
- Un mot de passe **pixelisé n'est pas protégé** → Depix le reconstruit.
- **Depix** : viser l'image **rasterisée + recadrée** du VRAI document — jamais le PDF brut, jamais l'exemple du repo.
- **Antivirus** : exclure le dossier de notes pentest (les payloads type reverse shell sont flaggés `RevWebshell` et le fichier est supprimé) ; ne lancer les outils offensifs que depuis la **VM Kali**.

## Références
- Fiches liées : [gobuster](../outils/gobuster.md) · [linux-privesc](../outils/linux-privesc.md) · [hashcat](../outils/hashcat.md) · [commandes-utiles](../outils/commandes-utiles.md)
