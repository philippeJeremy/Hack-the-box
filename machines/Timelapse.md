# Timelapse — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows (contrôleur de domaine) |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-__-__ |
| **Vecteur** | <à compléter moi-même> |
| **CVE** | — |
| **Tags** | active-directory · pfx · certificat · laps |

> 🎯 **Nouveautés à apprendre (je cherche les commandes seul — cf. `outils/ad-memo.md`)** :
> 1) Une archive protégée + un **certificat `.pfx`** protégé par mot de passe → à **cracker** (pense `*2john`).
> 2) Un `.pfx` sert à s'authentifier en **WinRM par certificat** (pas par mot de passe).
> 3) **LAPS** : le mot de passe de l'admin local est stocké **dans l'AD**, lisible par certains comptes → le lire.

---

## TL;DR
<à remplir en fin de box>

## 1. Reconnaissance
<ports, partages — que trouve-t-on en anonyme ?>

## 2. Foothold
<quel fichier récupéré ? comment casser le .pfx ? comment l'utiliser pour se connecter ?>

## 3. Privesc
<quel compte peut lire LAPS ? où est le mdp admin local dans l'AD ?>

## 4. Remédiation
<à remplir>

## 5. Leçons
- <pfx : authentification par certificat, crackage>
- <LAPS : mdp admin local centralisé dans l'AD>

## Références
- Fiches liées : [smb](../outils/smb.md) · [ad-memo](../outils/ad-memo.md) · [hashcat](../outils/hashcat.md)
