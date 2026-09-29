# Kubernetes — cheatsheet

Pentest de cluster : surface d'API, RBAC, tokens de service-account, secrets, pods privilégiés, évasion vers le nœud. Prolonge [`linux-privesc`](linux-privesc.md) (l'évasion de conteneur retombe sur du Linux) et [`nmap`](nmap.md).

> ⚠️ **Rappel légal.** Uniquement sur ton **lab auto-hébergé** (Kubernetes Goat, Bust-a-Kube, cluster kind/minikube perso) ou plateformes autorisées (CloudGoat, Academy HTB). Hors cadre contractuel écrit → infraction, **art. 323-1**.

**Plan** : commandes → recettes → côté défense → à savoir expliquer à l'oral.

---

## 1. Surface d'attaque

| Composant | Risque type |
| --- | --- |
| **API server** (6443) | exposition, auth anonyme, RBAC trop large |
| **kubelet** (10250) | endpoints exposés → exec dans les pods |
| **etcd** (2379) | secrets/état du cluster en clair si non chiffré |
| **Service-account token** | monté par défaut dans chaque pod (`/var/run/secrets/...`) |
| **Pod privilégié / hostPath / hostPID** | évasion vers le nœud hôte |
| **Secrets** | base64 ≠ chiffré ; RBAC de lecture souvent trop large |

---

## 2. Commandes — découverte & énumération

```bash
# Recon externe (lab)
kube-hunter --remote <ip>            # découverte de surface
nmap -p 6443,10250,2379,8443 <ip>

# Depuis un pod compromis — le token monté par défaut
cat /var/run/secrets/kubernetes.io/serviceaccount/token
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
env | grep KUBERNETES                # adresse de l'API server

# Ce que ce token me permet (la question centrale du RBAC)
kubectl auth can-i --list
kubectl auth can-i create pods
kubectl get pods,secrets,nodes -A 2>/dev/null

# Posture de sécurité (aussi un outil Blue)
kube-bench                           # CIS benchmark
kubescape scan
trivy image <image>                  # vulns des images
```

---

## 3. Recettes (lab)

**a. Token SA → énumération RBAC.** Depuis un pod, récupérer le token monté, puis `kubectl auth can-i --list` : c'est le point de départ. Un SA autorisé à `create pods` ou `get secrets` dans un namespace large = accès disproportionné.

**b. Lecture de secrets.** Si le RBAC autorise `get secrets` : `kubectl get secret <n> -o jsonpath='{.data}'` puis décodage base64. Souvent on y trouve des creds vers d'autres services (BDD, registry) → rebond.

**c. Pod privilégié → évasion vers le nœud.** Un droit de créer un pod `privileged` / `hostPath: /` / `hostPID` permet de monter le système de fichiers du nœud et de sortir du conteneur. C'est l'équivalent K8s d'une privesc SUID : à dérouler pas à pas sur **Kubernetes Goat** (scénario dédié), en documentant *pourquoi* le champ de sécurité manquant l'autorise.

**d. kubelet exposé.** Endpoint kubelet accessible non authentifié → exécution de commandes dans les pods du nœud. À tester sur lab uniquement.

**e. Outillage post-expl.** **Peirates** automate l'énum token → escalade → évasion : bon pour comprendre la chaîne, à faire une fois à la main d'abord.

> *À compléter avec tes captures Kubernetes Goat (scénario → chaîne → preuve).*

---

## 4. Côté défense / détection

| Attaque | Contrôle / trace |
| --- | --- |
| Token SA abusé | audit log API server (`verb`, `user`, `sourceIP`) |
| Pod privilégié créé | **Pod Security Admission** (restricted) le refuse ; alerte sur `securityContext.privileged` |
| Lecture de secrets | audit log `get secret` ; RBAC au moindre privilège |
| kubelet exposé | `--anonymous-auth=false`, NetworkPolicy |
| Images vulnérables | Trivy en CI, admission control (signature) |

Durcissement : RBAC minimal, `automountServiceAccountToken: false` par défaut, Pod Security Admission `restricted`, NetworkPolicies entre namespaces, etcd chiffré au repos, audit logging activé.

---

## 5. À savoir expliquer à l'oral

- « La question n°1 dans un cluster, c'est **le RBAC** : `kubectl auth can-i --list` avant tout. »
- Pourquoi un **pod privilégié = compromission du nœud** (et souvent du cluster de proche en proche).
- base64 n'est **pas** du chiffrement : un secret K8s lisible est un secret exposé.
- Le lien avec le reste : un secret K8s contient souvent les creds qui mènent à l'annuaire ou à la BDD → K8s est un tremplin, pas une fin.
- Réflexe défensif : Pod Security Admission `restricted` + `automountServiceAccountToken: false`.
