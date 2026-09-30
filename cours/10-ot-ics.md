# Module 10 — Systèmes industriels / OT

> ⚠️ Exercices sur **simulateurs et labs dédiés** uniquement (GRFICS, Conpot…), jamais sur un environnement de production. En OT, un test mal conduit peut avoir des conséquences physiques. Art. 323-1 + prudence renforcée.

## Objectif
Comprendre ce qui rend les **systèmes industriels (OT)** différents de l'IT, pourquoi les méthodes IT ne s'y appliquent pas telles quelles, et comment on les aborde. C'est le vrai différenciateur d'un profil orienté industrie/défense.

## Pourquoi c'est clé pour le poste
En environnement industriel, l'enjeu final n'est pas la donnée mais le **procédé physique** (une chaîne de production, un système d'un navire…). Peu de pentesters maîtrisent l'OT : c'est rare, donc recherché. Ta double culture (production industrielle + cyber) est ici un atout majeur.

## Concepts

**OT vs IT : la priorité s'inverse.**
- En **IT**, on protège d'abord la **confidentialité** (C-I-A : Confidentialité > Intégrité > Disponibilité).
- En **OT**, c'est l'inverse : la **disponibilité et la sûreté** priment. Un automate qui pilote une vanne ne doit **jamais** s'arrêter. Cette inversion change tout : un scan agressif « normal » en IT peut faire tomber un automate fragile en OT.

**Le vocabulaire (à connaître) :**
- **SCADA** : le système de supervision/contrôle (les écrans de pilotage).
- **PLC / automate** : le petit calculateur qui pilote un équipement physique (capteur → décision → actionneur).
- **HMI** : l'interface homme-machine (l'écran opérateur).
- **Historian** : la base qui archive les données du procédé.

**Le modèle de Purdue.** Le schéma de référence de l'architecture industrielle : des **niveaux** (0 = capteurs/actionneurs, jusqu'aux niveaux entreprise IT en haut), séparés par une **DMZ industrielle**. L'idée de sécurité : l'IT et l'OT ne doivent **pas** se parler directement ; tout passe par des points de contrôle. Beaucoup d'attaques OT commencent par un pied dans l'IT qui « descend » vers l'OT via une segmentation défaillante — d'où l'importance des modules 1 (réseau) et 5 (AD).

**Les protocoles industriels.** Modbus, OPC-UA, S7… Beaucoup ont été conçus **sans sécurité** (pas d'authentification, pas de chiffrement) car pensés pour des réseaux isolés. Les comprendre en lab (lecture de trames) suffit à voir le problème : qui peut parler à un automate peut souvent le commander.

**La contrainte majeure : ne rien casser.** En OT, on privilégie l'analyse **passive** (écoute du trafic) et on évite l'actif intrusif. Un test OT se conçoit avec l'exploitant, souvent sur banc d'essai ou en fenêtre d'arrêt, jamais « à chaud » sans précaution.

## Méthodologie (spécifique OT)
1. **Comprendre le procédé** avant la technique : qu'est-ce qui est piloté, quel est le risque physique ?
2. **Cartographier passivement** : écouter le réseau, identifier automates/protocoles sans perturber.
3. **Analyser la segmentation** IT/OT (le modèle de Purdue est-il respecté ?).
4. **Évaluer** les points faibles de conception (protocoles non authentifiés, accès distants).
5. **Recommander** sans jamais compromettre la disponibilité.

## Outils & labs
- **GRFICS** — simulation d'usine virtuelle complète (procédé + automates + HMI) : idéal pour s'entraîner sans risque.
- **Conpot** — honeypot ICS pour observer protocoles et interactions.
- **TryHackMe** — rooms d'introduction ICS/SCADA.
- **Wireshark** — analyse passive des protocoles industriels.

## Pièges courants
- Appliquer les **réflexes IT** (scan agressif, exploit « pour voir ») : en OT, ça peut arrêter une production ou pire.
- Ignorer le **procédé physique** : sans comprendre ce qui est piloté, on ne mesure pas le risque réel.
- Considérer l'OT comme « de l'IT en retard » : c'est un monde avec ses contraintes propres (sûreté, temps réel, durée de vie des équipements en décennies).

## Côté défense (Blue)
- **Segmentation stricte** IT/OT (modèle de Purdue, DMZ industrielle) : la contre-mesure n°1.
- **Monitoring passif** spécialisé OT (détection d'anomalies sans injecter de trafic).
- **Contrôle des accès distants** (maintenance fournisseurs = vecteur classique).
- Référentiels : **IEC 62443** (norme sécurité des systèmes industriels), **MITRE ATT&CK for ICS**, guides **ANSSI** sur la cybersécurité industrielle.

## À l'oral
- *« Différence fondamentale IT/OT ? »* → la priorité s'inverse : en OT, disponibilité et sûreté avant confidentialité ; un arrêt peut avoir un impact physique.
- *« C'est quoi le modèle de Purdue ? »* → une architecture en niveaux séparant capteurs/automates de l'IT par une DMZ industrielle ; le principe est que l'IT et l'OT ne se parlent pas directement.
- *« Pourquoi on ne scanne pas un réseau OT comme un réseau IT ? »* → des automates fragiles peuvent tomber sous un scan agressif ; on privilégie l'analyse passive et on cadre tout avec l'exploitant.
- *« Pourquoi les protocoles industriels sont un problème ? »* → souvent sans authentification ni chiffrement (conçus pour des réseaux isolés) ; qui accède au réseau peut commander les équipements.

## Pratiquer
**GRFICS** pour un procédé complet en lab, **Conpot** pour les protocoles, la room ICS de TryHackMe pour l'intro. Quand tu attaqueras ce module, crée la fiche `outils/ot-ics.md` pour tes commandes et observations.
