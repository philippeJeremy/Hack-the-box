# Builder — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-__-__ |
| **Vecteur** | <à compléter moi-même> |
| **CVE** | CVE-2024-23897 (Jenkins) |
| **Tags** | web · jenkins · cve-2024-23897 · file-read · secret-decryption |

> 🎯 **Nouveautés à apprendre (je cherche seul)** :
> 1) **Jenkins** exposé, vulnérable à **CVE-2024-23897** (argument `@file` du CLI → **lecture de fichiers arbitraires** non authentifiée).
> 2) Lire les fichiers internes de Jenkins pour récupérer un **hash** utilisateur (à casser) ou, mieux, les **secrets chiffrés**.
> 3) **Déchiffrer les credentials Jenkins** : il faut **3 fichiers** — `credentials.xml`, `master.key`, `hudson.util.Secret` — pour reconstituer une **clé SSH / un mot de passe** stocké → root.

> 💡 Concepts à googler : « CVE-2024-23897 exploit », « decrypt Jenkins credentials.xml master.key hudson.util.Secret », « jenkins-credential-decryptor ».

---

## TL;DR
<à remplir en fin de box>

## 1. Reconnaissance
<port 8080 Jenkins ? version → vulnérable à CVE-2024-23897 ?>

## 2. Foothold — lecture de fichiers (CVE-2024-23897)
<lire /etc/passwd pour valider, puis le fichier config utilisateur → hash → crack → login Jenkins>

## 3. Privesc — déchiffrer les secrets Jenkins
<récupérer credentials.xml + master.key + hudson.util.Secret via la CVE → déchiffrer → clé SSH root>

## 4. Remédiation
<patcher Jenkins ; ne pas exposer le CLI ; secrets dans un vault externe>

## 5. Leçons
- <une "lecture de fichier" non authentifiée mène au total compromise si on vise les bons fichiers>
- <les secrets Jenkins sont chiffrés mais déchiffrables avec master.key + hudson.util.Secret>

## Références
- Fiches liées : [searchsploit](../outils/searchsploit.md) · [gobuster](../outils/gobuster.md) · [linux-privesc](../outils/linux-privesc.md)
