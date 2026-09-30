# Module 6 — Élévation de privilèges

> ⚠️ Exercices sur labs autorisés uniquement (art. 323-1). Voir [`index`](index.md).

## Objectif
Passer d'un accès **utilisateur limité** (obtenu au module 4) à **root/SYSTEM** sur la machine, en énumérant méthodiquement les mauvaises configurations locales.

## Pourquoi c'est clé pour le poste
Un premier shell est presque toujours faible. La privesc est ce qui transforme un « pied dans la porte » en contrôle réel. C'est aussi un révélateur d'hygiène : les chemins de privesc sont des erreurs de configuration que tu devras savoir *expliquer et corriger* en rapport.

## Concepts

**Le principe : une privesc = une confiance mal placée locale.** Le système exécute quelque chose avec plus de droits que l'utilisateur, et cet utilisateur peut l'influencer. On cherche systématiquement ces points.

**Linux — les grandes familles :**
- **sudo mal configuré** : le droit d'exécuter un binaire en root qui permet de sortir (cf. **GTFOBins**).
- **SUID/SGID** : un binaire qui s'exécute avec les droits du propriétaire (souvent root) ; certains permettent une évasion.
- **Capabilities** : droits fins (ex. `cap_setuid`) posés sur un binaire → équivalent d'un SUID ciblé.
- **cron** : une tâche planifiée root qui exécute un script *modifiable* par l'utilisateur.
- **PATH / bibliothèques** : détourner ce que charge un programme privilégié.
- **Noyau** : un kernel non patché avec un exploit connu (dernier recours, risque de crash).

**Windows — les grandes familles :**
- **Jetons / `SeImpersonate`** (famille « Potato ») : abuser d'un privilège pour usurper SYSTEM.
- **Services mal configurés** : binaire de service modifiable, chemin non protégé (unquoted path), permissions faibles.
- **Tâches planifiées** exécutées en privilégié et modifiables.
- **Creds qui traînent** : dans des fichiers, le registre, des sauvegardes.
- **Noyau** : exploit MS non patché.

**Le réflexe central : ÉNUMÉRER avant d'escalader.** On ne devine pas une privesc, on la *trouve* en passant une checklist. Les outils (winPEAS/linPEAS) automatisent la collecte ; ton travail est de **lire leur sortie** intelligemment.

**Lire PEAS.** Un PEAS crache des centaines de lignes en couleurs. La compétence n'est pas de le lancer, c'est de repérer le **signal** (une ligne surlignée = un vecteur probable) dans le bruit, et de vérifier avant d'agir.

## Méthodologie
1. **Stabiliser** le shell (module 4) et savoir *qui* on est (`id` / `whoami /priv`).
2. **Énumérer** : lancer la checklist (manuelle) + PEAS (auto).
3. **Trier** les vecteurs par simplicité et fiabilité.
4. **Vérifier** un vecteur avant de l'exploiter (comprendre *pourquoi* il marche).
5. **Escalader**, confirmer root/SYSTEM, **documenter** la cause (c'est le futur point de remédiation).

## Outils & rôle
- **linux-privesc** ([fiche](../outils/linux-privesc.md)) — checklist Linux.
- **windows-privesc** ([fiche](../outils/windows-privesc.md)) — checklist Windows.
- **gtfobins** ([fiche](../outils/gtfobins.md)) — binaires détournables (sudo/SUID/capabilities).
- **lire-peas** ([fiche](../outils/lire-peas.md)) — méthode de lecture de winPEAS/linPEAS.

## Pièges courants
- **Foncer sur l'exploit noyau** en premier : risqué (crash) et souvent inutile si une mauvaise config plus propre existe.
- Lancer PEAS et se noyer : sans méthode de lecture, on rate le vecteur évident.
- Ne pas noter la **cause racine** : sans elle, pas de remédiation dans le rapport.

## Côté défense (Blue)
- **Hygiène de configuration** : sudo au strict nécessaire, pas de SUID inutile, services/tâches aux bonnes permissions, pas de creds en clair.
- **Patch** du noyau et des composants.
- Surveiller les **exécutions anormales** (un processus privilégié lancé par un binaire inattendu) via Sysmon/auditd (module 9).
- Principe du **moindre privilège** partout : la meilleure privesc est celle qui n'existe pas.

## À l'oral
- *« Ta démarche de privesc ? »* → énumération méthodique d'abord (checklist + PEAS), tri des vecteurs, vérification, puis escalade — jamais à l'aveugle.
- *« GTFOBins, c'est quoi ? »* → un référentiel de binaires qui, lancés en sudo/SUID, permettent de sortir vers un shell privilégié. Correctif : ne pas donner ces binaires en sudo.
- *« SeImpersonate, pourquoi c'est dangereux ? »* → ce privilège permet d'usurper des jetons, jusqu'à SYSTEM (famille Potato). On le retrouve souvent sur des comptes de service web.

## Pratiquer
Ta **phase 1-2** : Shocker, Nibbles, Bashed (Linux) ; Devel, Optimum, Bounty (Windows). Impose-toi de **dérouler la checklist avant** de chercher l'exploit, et d'écrire la cause racine de chaque escalade.
