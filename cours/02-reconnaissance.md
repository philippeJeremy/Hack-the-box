# Module 2 — Reconnaissance

> ⚠️ Exercices sur labs autorisés uniquement (art. 323-1). Voir [`index`](index.md).

## Objectif
Transformer « une machine/un réseau inconnu » en une **carte détaillée** : services, versions, points d'entrée possibles. C'est la phase qui décide de tout le reste.

## Pourquoi c'est clé pour le poste
La recon, c'est 80 % d'une mission. Un débutant exploite au hasard ; un pro énumère méthodiquement et *sait* quoi tenter. En pilotage d'équipe, c'est aussi la phase la plus normée (checklists, périmètre) — celle que tu devras cadrer.

## Concepts

**Passive vs active.**
- **Passive** : sans toucher la cible (OSINT, DNS public, moteurs de recherche, fuites). Discrète, légale même hors périmètre pour l'info publique, mais limitée.
- **Active** : on interroge la cible (scans, requêtes). Riche, mais bruyante et **strictement dans le périmètre autorisé**.

**Énumération = le cœur.** Après le scan de ports (module 1), on *creuse chaque service* :
- **Bannières & versions** : une version précise → une vulnérabilité connue (module 4).
- **Contenu caché** : sur un service web, découvrir les répertoires/fichiers non listés (force brute de chemins).
- **Partages & comptes** : SMB, NFS, LDAP livrent souvent des noms d'utilisateurs, des partages ouverts, des politiques.
- **Configuration** : ce qui est mal réglé (accès anonyme, options par défaut) vaut souvent mieux qu'une CVE.

**La règle d'or : énumérer avant d'exploiter.** Chaque service qu'on ouvre pose une question : *qu'est-ce qu'il me révèle ? à quoi il donne accès ?* On ne passe à l'exploitation que quand on a épuisé ce que la recon nous apprend.

**Profondeur.** La première passe est superficielle. La vraie recon est *itérative* : une info trouvée (un nom d'utilisateur, un sous-domaine) relance une nouvelle recherche. On boucle jusqu'à ne plus rien apprendre.

## Méthodologie
1. **Scan de référence** (ports/services/versions) — le point de départ.
2. **Par service, énumérer** : web → contenu caché ; SMB → partages/utilisateurs ; etc.
3. **Collecter les indices** : versions, noms, chemins, identifiants potentiels.
4. **Recouper** : un indice d'un service en éclaire un autre.
5. **Prioriser les pistes** : classer par probabilité de succès × impact.

## Outils & rôle
- **nmap** ([fiche](../outils/nmap.md)) — scan + scripts NSE d'énumération.
- **smb / NetExec** ([fiche](../outils/smb.md)) — énumération SMB : partages, utilisateurs, politiques.
- **gobuster / ffuf** ([fiche](../outils/gobuster.md)) — force brute de chemins et sous-domaines web.
- **burp** ([fiche](../outils/burp.md)) — cartographier une application web.

## Pièges courants
- **Aller trop vite à l'exploit** : l'erreur n°1. Si tu es bloqué, c'est presque toujours que tu as *sous-énuméré*.
- Se contenter de la première passe : la piste gagnante est souvent au 2ᵉ ou 3ᵉ niveau.
- Sortir du périmètre en recon active : une IP qui n'est pas dans le scope ne se scanne pas, point.

## Côté défense (Blue)
- L'énumération SMB/LDAP anonyme se **coupe** (désactiver l'accès null session, durcir les partages).
- La force brute de chemins web génère une **rafale de 404** → détectable, à limiter (rate limiting, WAF).
- Réduire la **surface** : moins de services exposés, moins de bannières verbeuses = moins à énumérer.

## À l'oral
- *« Par quoi tu commences sur une cible ? »* → recon complète, énumérer chaque service, *avant* toute exploitation.
- *« Tu es bloqué sur une box, que fais-tu ? »* → je reprends l'énumération ; un blocage = un service sous-exploré, pas un exploit manquant.
- *« Passive ou active, différence de cadre légal ? »* → l'info publique est libre ; l'active touche la cible et exige une autorisation écrite et un périmètre.

## Pratiquer
Sur chaque machine HTB, impose-toi de **tout énumérer avant d'ouvrir Metasploit**. Cap et Networked récompensent la recon web ; les box Windows récompensent l'énum SMB/LDAP.
