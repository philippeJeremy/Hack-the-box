# Module 3 — Sécurité web

> ⚠️ Exercices sur labs autorisés uniquement (art. 323-1). Voir [`index`](index.md).

## Objectif
Comprendre les **familles de failles web** et leur logique commune, pour les reconnaître vite. Le web est souvent la première porte d'entrée d'un SI.

## Pourquoi c'est clé pour le poste
La majorité des points d'entrée externes sont des applications web. Savoir raisonner « où l'appli fait-elle confiance à tort ? » est un réflexe transférable à tout, y compris aux interfaces d'admin en environnement industriel.

## Concepts

**Le principe unique derrière presque tout : la confiance mal placée.** Une appli web casse quand elle fait confiance à une donnée qu'elle ne devrait pas (entrée utilisateur, identifiant d'objet, en-tête, fichier uploadé). Garde cette lentille : *où l'appli croit-elle l'utilisateur sur parole ?*

**OWASP Top 10 — les familles à connaître :**
- **Contrôle d'accès défaillant (A01)** — l'appli ne vérifie pas *qui* a le droit. Ex. **IDOR** : changer un identifiant dans l'URL pour voir les données d'un autre. La faille la plus courante et souvent la plus grave.
- **Injections (A03)** — une entrée est interprétée comme du code/commande : **SQLi** (base de données), injection de commande OS, LDAP… Cause racine : données et instructions mélangées.
- **XSS** — injecter du script qui s'exécute dans le navigateur d'un autre utilisateur.
- **Mauvaise configuration (A05)** — options par défaut, pages d'admin exposées, verbosité des erreurs.
- **Authentification défaillante (A07)** — mots de passe faibles, sessions mal gérées, absence de MFA.
- **SSRF** — forcer le serveur à faire une requête pour toi (vu au module K8s : atteindre la metadata interne).
- **Composants vulnérables (A06)** — une lib/un CMS avec une CVE connue.

**Upload de fichiers.** Un formulaire d'upload mal filtré = dépôt d'un fichier exécutable → exécution de code côté serveur. À croiser avec l'exploitation (module 4).

**La chaîne web typique** : découverte de contenu (module 2) → identifier une confiance mal placée → obtenir un accès (données, ou exécution) → pivoter vers le système.

## Méthodologie
1. **Cartographier** l'appli : pages, paramètres, points d'entrée (avec un proxy).
2. **Pour chaque entrée, se demander** : qu'est-ce que l'appli en fait ? qui vérifie les droits ?
3. **Tester une famille à la fois**, méthodiquement (accès, injection, upload…).
4. **Comprendre avant d'automatiser** : un outil trouve, mais c'est toi qui interprètes.

## Outils & rôle
- **burp** ([fiche](../outils/burp.md)) — le proxy central : intercepter, rejouer (Repeater), fuzzer (Intruder). Cœur du module.
- **gobuster / ffuf** ([fiche](../outils/gobuster.md)) — découverte de contenu et de paramètres.
- **PortSwigger Web Security Academy** — la meilleure ressource (gratuite) pour *comprendre* chaque faille en labo.

## Pièges courants
- Lancer un scanner automatique sans comprendre : bruit, faux positifs, et on n'apprend rien.
- Négliger la logique métier : l'IDOR ne se trouve pas au scanner, mais en réfléchissant au *sens* des identifiants.
- Oublier que le web est souvent un *tremplin* : l'accès web mène au système, ce n'est pas la fin.

## Côté défense (Blue)
- **Validation & paramétrage** : requêtes préparées (anti-SQLi), échappement de sortie (anti-XSS), contrôle d'accès *côté serveur* systématique (anti-IDOR).
- **WAF** en défense de profondeur, jamais comme seule protection.
- Journaliser les accès anormaux (rafales, paramètres manipulés) ; alerter sur les erreurs 500 en série.

## À l'oral
- *« Explique l'IDOR. »* → l'appli ne vérifie pas que l'objet demandé appartient à l'utilisateur ; je change l'identifiant et j'accède aux données d'autrui. Correctif : contrôle d'accès côté serveur.
- *« Cause racine d'une injection ? »* → données et instructions mélangées. Correctif : séparer (requêtes préparées, échappement).
- *« Un WAF suffit-il ? »* → non, c'est de la défense de profondeur ; la vraie correction est dans le code.

## Pratiquer
**PortSwigger Academy** (Apprentice puis Practitioner) pour la théorie en labo, puis les box web HTB (Cap, Blocky, Networked, Cronos, Node) pour l'enchaînement web → système.
