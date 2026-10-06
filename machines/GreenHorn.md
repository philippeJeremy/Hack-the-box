# GreenHorn — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-__-__ |
| **Vecteur** | <à compléter moi-même> |
| **CVE** | — |
| **Tags** | web · cms · git-leak · rce · depix |

> 🎯 **Nouveautés à apprendre (je cherche seul)** :
> 1) Un **dépôt Git** (Gitea) exposé laisse lire le **code source** de l'appli → secrets / hash dans les fichiers.
> 2) Le CMS (**Pluck**) a une **CVE d'upload/RCE** authentifié → shell (le mdp vient du code source leaké).
> 3) Privesc original : un mot de passe fuit sous forme d'**image pixelisée** (PDF/screenshot) → à **dépixeliser** avec **Depix** pour le reconstituer.

> 💡 Concepts à googler : « Gitea source disclosure », « Pluck CMS RCE CVE », « Depix pixelated password recovery ».

---

## TL;DR
<à remplir en fin de box>

## 1. Reconnaissance
<ports, vhosts — Gitea ? Pluck CMS ? versions>

## 2. Foothold
<lire le code via Git → trouver le hash/mdp admin → login CMS → CVE RCE>

## 3. Privesc — dépixelisation
<où est l'image/PDF du mdp ? extraire l'image → Depix → mot de passe root>

## 4. Remédiation
<ne pas exposer .git / Gitea public ; ne jamais "flouter" un secret (réversible)>

## 5. Leçons
- <fuite de code source = fuite de secrets ; toujours fouiller un Git exposé>
- <un mot de passe pixelisé n'est PAS protégé → Depix le reconstruit>

## Références
- Fiches liées : [gobuster](../outils/gobuster.md) · [searchsploit](../outils/searchsploit.md) · [linux-privesc](../outils/linux-privesc.md) · [hashcat](../outils/hashcat.md)
