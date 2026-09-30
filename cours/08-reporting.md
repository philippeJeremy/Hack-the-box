# Module 8 — Reporting

> ⚠️ Cadre professionnel : voir [`index`](index.md).

## Objectif
Transformer une mission technique en **livrable clair et actionnable** pour deux publics : les décideurs (impact, risque) et les techniques (reproduction, correction). Le rapport *est* le produit d'un pentest.

## Pourquoi c'est clé pour le poste
On ne te paie pas pour « avoir root », mais pour **réduire le risque du client**. Un excellent test mal rapporté ne vaut rien. Et pour un poste de **pilotage**, la qualité et l'homogénéité des rapports de l'équipe, c'est ta responsabilité directe.

## Concepts

**Deux publics, deux niveaux.**
- **Synthèse pour décideurs (executive summary)** : sans jargon. Quel est le risque métier, quelle est sa gravité, que faut-il faire en priorité. Une page, lisible par un dirigeant.
- **Détail technique** : par vulnérabilité, de quoi la comprendre, la rejouer et la corriger.

**Anatomie d'une vulnérabilité bien rapportée :**
- **Titre** clair + **criticité** (échelle type CVSS, mais surtout : impact réel *dans ce contexte*).
- **Description** : la faille et sa cause racine.
- **Preuve (PoC)** : la démonstration reproductible (captures, étapes) — factuelle, sobre.
- **Impact** : ce qu'un attaquant en tire concrètement pour *ce* client.
- **Remédiation** : comment corriger, priorisée et réaliste. C'est la partie la plus importante pour le client.

**Prioriser par le risque, pas par la technique.** Une faille « simple » exposée sur Internet peut être plus grave qu'une chaîne « élégante » interne. Le classement se fait sur **impact × probabilité**, dans le contexte du client.

**Factuel et reproductible.** Chaque affirmation s'appuie sur une preuve. Un technique du client doit pouvoir **rejouer** ta découverte à partir du rapport. Pas d'esbroufe, pas de « j'ai réussi à » sans montrer quoi.

**La traçabilité du test.** Périmètre, dates, méthodologie, limites (ce qui n'a pas pu être testé). Ça protège le client *et* le testeur, et ça rend la mission auditable.

## Méthodologie (elle commence dès la mission)
1. **Prendre des notes au fil de l'eau** : chaque étape, horodatée, avec captures. Un rapport ne se reconstitue pas de mémoire.
2. **Structurer** par vulnérabilité au fur et à mesure.
3. **Rédiger la synthèse** en dernier, une fois l'impact global compris.
4. **Prioriser** les remédiations.
5. **Relire pour le public** : le décideur comprend-il la synthèse ? le technique peut-il rejouer ?

## Lien avec ton dépôt
Tes write-ups de machines suivent déjà la structure d'un rapport (**recon → exploitation → post-exploitation → remédiation → leçons**) : c'est exactement le bon entraînement. Passer du write-up HTB au rapport client, c'est surtout **ajouter la synthèse décideur** et **contextualiser l'impact**.

## Pièges courants
- Rapport **trop technique** sans synthèse : le décideur ne peut pas agir.
- Remédiations **génériques** (« mettez à jour ») au lieu de précises et priorisées.
- Découvertes **non reproductibles** : preuve insuffisante = doute sur le résultat.
- Ton **sensationnaliste** : un rapport est factuel, il n'effraie pas, il informe.

## À l'oral
- *« À quoi sert un pentest ? »* → réduire le risque du client ; le livrable est le rapport, pas le fait d'avoir eu root.
- *« Comment tu priorises tes findings ? »* → impact × probabilité dans le contexte du client, pas par élégance technique.
- *« Que contient un bon rapport ? »* → une synthèse décideur actionnable + un détail technique reproductible avec remédiations priorisées + le cadre (périmètre, dates, limites).

## Pratiquer
Reprends une de tes machines et rédige-la **comme un rapport client** : ajoute une synthèse « pour un dirigeant » et une remédiation priorisée. C'est le meilleur exercice pour franchir le pas write-up → rapport.
