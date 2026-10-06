# Usage — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-__-__ |
| **Vecteur** | <à compléter moi-même> |
| **CVE** | — |
| **Tags** | web · sqli · laravel · file-upload · 7z-wildcard |

> 🎯 **Nouveautés à apprendre (je cherche seul)** :
> 1) **SQLi** sur la page de reset de mot de passe (souvent **blind** → `sqlmap`) → dumper le hash de l'admin → le casser.
> 2) Panel **Laravel-admin** : un **upload d'avatar** mal filtré permet d'uploader un **webshell PHP** → RCE.
> 3) Latéral : **réutilisation de mot de passe** (fichier de conf / `.monit`) vers un autre user.
> 4) Privesc : un binaire `sudo` utilise **7z** en root → l'option de **wildcard `@fichier` / `-i!`** permet de **lire des fichiers root** (ex. clé SSH) via les messages d'erreur.

> 💡 Concepts à googler : « sqlmap blind reset form », « laravel-admin file upload RCE », « 7z sudo arbitrary file read wildcard ».

---

## TL;DR
<à remplir en fin de box>

## 1. Reconnaissance
<vhosts (admin.usage.htb ?) — quelle techno ? Laravel ?>

## 2. Foothold — SQLi → admin → upload
<où est la SQLi ? sqlmap pour dumper → crack → login admin → upload webshell>

## 3. Latéral
<quel fichier contient un mdp réutilisable ? vers quel user ?>

## 4. Privesc — 7z en sudo
<sudo -l → le binaire 7z ; comment lire /root/.ssh/id_rsa via le wildcard ?>

## 5. Leçons
- <blind SQLi : automatiser avec sqlmap, puis crack>
- <upload mal filtré = RCE ; vérifier l'extension réellement acceptée>
- <7z en root + wildcard = lecture de fichiers arbitraires (fuite par erreur)>

## Références
- Fiches liées : [burp](../outils/burp.md) · [gobuster](../outils/gobuster.md) · [linux-privesc](../outils/linux-privesc.md) · [hashcat](../outils/hashcat.md)
