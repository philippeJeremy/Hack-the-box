# Codify — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-__-__ |
| **Vecteur** | <à compléter moi-même> |
| **CVE** | — (sandbox escape) |
| **Tags** | web · nodejs · vm2 · sandbox-escape · sudo-script |

> 🎯 **Nouveautés à apprendre (je cherche seul)** :
> 1) L'appli web laisse **exécuter du code Node.js** dans un bac à sable (**vm2**). Les vieilles versions de vm2 ont une **évasion de sandbox** connue → RCE en tant que l'utilisateur du service.
> 2) Fouille post-accès : une **base SQLite** contient un **hash** à casser → 2e utilisateur.
> 3) Privesc : un **script lançable en `sudo`** compare un mot de passe de façon dangereuse (pense `[[ $x == $pattern ]]` → **wildcard/brute char par char**).

> 💡 Concepts à googler : « vm2 sandbox escape POC », « bash [[ ]] wildcard password bruteforce ».

---

## TL;DR
<à remplir en fin de box>

## 1. Reconnaissance
<ports, vhosts — quelle appli ? quelle version de vm2 ?>

## 2. Foothold — évasion vm2
<trouver le POC d'évasion correspondant à la version → RCE → reverse shell>

## 3. Latéral
<où est la base SQLite ? quel hash ? casser → 2e user (réutilisation ?)>

## 4. Privesc
<sudo -l → quel script ? comment la comparaison de mdp fuit-elle les caractères un par un ?>

## 5. Leçons
- <un "bac à sable" JS obsolète = RCE ; toujours noter la VERSION>
- <comparaison bash avec wildcard = fuite exploitable caractère par caractère>

## Références
- Fiches liées : [gobuster](../outils/gobuster.md) · [burp](../outils/burp.md) · [linux-privesc](../outils/linux-privesc.md)
