# Module 12 — Conteneurs & Kubernetes

> ⚠️ Exercices sur lab auto-hébergé jetable (Kubernetes Goat…) uniquement (art. 323-1). Fiche opérationnelle + déroulé Goat : [`kubernetes`](../outils/kubernetes.md).

## Objectif
Comprendre comment un cluster Kubernetes fait confiance (ou pas), et comment on remonte la chaîne **pod → nœud → cluster**. C'est la couche cloud/moderne des SI.

## Pourquoi c'est clé pour le poste
Les applications d'entreprise migrent massivement vers les conteneurs. Un pentester qui sait auditer un cluster (RBAC, isolation, secrets) couvre un angle que beaucoup ignorent encore.

## Concepts

**L'échelle de rebond.** Le modèle mental clé : **pod** (le conteneur) → **node** (la VM qui l'héberge) → **namespace** (cloisonnement logique) → **cluster** (le tout). Tout le jeu consiste à remonter cette échelle : d'un pod compromis jusqu'au contrôle du cluster.

**Le token de service-account : ton identité.** Par défaut, chaque pod reçoit un **token** monté dans son système de fichiers. Compromettre un pod = hériter de cette identité. La toute première question est donc : *que me permet ce token ?* → c'est le **RBAC**.

**Le RBAC, cœur du sujet.** Kubernetes contrôle qui peut faire quoi via des rôles. La commande réflexe (`auth can-i --list`) répond à « qu'est-ce que mon identité actuelle autorise ? ». Les mauvaises configs classiques : un service-account autorisé à **lire les secrets** ou à **créer des pods** dans un périmètre trop large.

**base64 n'est pas du chiffrement.** Les « secrets » Kubernetes sont encodés, pas chiffrés par défaut. Qui peut les lire les lit en clair → et ils contiennent souvent les creds vers une base de données, un registre, un autre service → rebond.

**L'évasion de conteneur.** Le point le plus grave : si on peut créer un **pod privilégié** (ou monter le système de fichiers du nœud), on sort du conteneur vers le nœud hôte avec les droits root. C'est l'équivalent K8s d'une privesc SUID (module 6) : une confiance mal placée (un champ de sécurité manquant) qui donne bien plus que prévu.

**Le namespace n'est pas une frontière de sécurité.** C'est un cloisonnement *logique*, pas un pare-feu. Sans **NetworkPolicies**, les pods de namespaces différents se parlent. D'où le mouvement latéral (module 7) une fois sur le nœud.

**SSRF → metadata.** Un pod hérite du réseau du nœud ; un SSRF (module 3) peut atteindre l'endpoint de metadata interne et en tirer des creds du nœud → sortie du périmètre K8s vers le cloud.

## Méthodologie
1. **Découvrir** : API server, kubelet, exposition (kube-hunter).
2. Depuis un pod : **lire le token**, puis `auth can-i --list`.
3. **Récolter** les secrets accessibles → creds de rebond.
4. **S'évader** si un pod privilégié est créable → nœud.
5. **Bouger latéralement** (tokens des autres pods, namespaces) → cluster.

## Outils & rôle
- **kubernetes** ([fiche](../outils/kubernetes.md)) — commandes + déroulé guidé Kubernetes Goat.
- kube-hunter / kube-bench / kubescape / Trivy / Peirates (détaillés dans la fiche).

## Pièges courants
- Oublier la **question RBAC** et partir chercher un exploit : le chemin est presque toujours dans les droits.
- Croire un **namespace** étanche : sans NetworkPolicy, il ne l'est pas.
- Prendre un **secret base64** pour une protection.

## Côté défense (Blue)
- **Pod Security Admission** `restricted` (refuse les pods privilégiés), `automountServiceAccountToken: false` par défaut.
- **RBAC minimal**, **NetworkPolicies** entre namespaces, **etcd chiffré** au repos.
- **Audit logging** de l'API server, **Falco** en runtime (détecte shell/montage anormal).
- Scan d'images (**Trivy**) en CI.

## À l'oral
- *« Par quoi tu commences dans un cluster ? »* → le RBAC : `auth can-i --list` sur l'identité du pod.
- *« Pourquoi un pod privilégié est-il critique ? »* → il permet de sortir vers le nœud en root, puis vers le cluster via les tokens des autres pods.
- *« Un namespace isole-t-il vraiment ? »* → non par défaut ; il faut des NetworkPolicies.
- *« Les secrets K8s sont-ils sûrs ? »* → base64 ≠ chiffré ; il faut chiffrer etcd et restreindre la lecture.

## Pratiquer
C'est ta **phase 8** : **Kubernetes Goat** (le « GOAD du K8s »), puis Bust-a-Kube / CloudGoat. Déroulé pas à pas dans la fiche `outils/kubernetes.md` ; livrable dans `machines/KubernetesGoat.md`.
