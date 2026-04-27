# TP #5 — Environments dev / staging / prod avec Namespaces & RBAC

---

## 1 — Namespaces

Trois namespaces isolent les environnements : `app-dev`, `app-staging`, `app-prod`.  
Chaque namespace porte des labels standard Kubernetes (`environment`, `owner`, `app.kubernetes.io/part-of`) et un niveau PSS différent.

| Namespace | PSS enforce | Objectif |
|---|---|---|
| `app-dev` | `baseline` | Bloque uniquement les configs vraiment risquées |
| `app-staging` | `baseline` + warn `restricted` | Prépare la migration progressive |
| `app-prod` | `restricted` | Niveau le plus strict — refus à l'admission |

---

### VALIDATION

```bash
kubectl get ns --show-labels
```

```
NAME         STATUS   AGE   LABELS
app-dev                Active   106m   app.kubernetes.io/part-of=demo-app,environment=dev,kubernetes.io/metadata.name=app-dev,owner=platform-team,pod-security.kubernetes.io/audit=baseline,pod-security.kubernetes.io/enforce=baseline,pod-security.kubernetes.io/warn=baseline
app-prod               Active   103m   app.kubernetes.io/part-of=demo-app,environment=prod,kubernetes.io/metadata.name=app-prod,owner=platform-team,pod-security.kubernetes.io/audit=restricted,pod-security.kubernetes.io/enforce=restricted,pod-security.kubernetes.io/warn=restricted
app-staging            Active   103m   app.kubernetes.io/part-of=demo-app,environment=staging,kubernetes.io/metadata.name=app-staging,owner=platform-team,pod-security.kubernetes.io/audit=restricted,pod-security.kubernetes.io/enforce=baseline,pod-security.kubernetes.io/warn=restricted
```

---

## 2 — Personas (ServiceAccounts)

Quatre identités simulées, chacune liée à un périmètre précis.

| Persona | ServiceAccount | Namespace de résidence |
|---|---|---|
| `platform-admin` | contexte minikube par défaut | cluster-wide |
| `dev-user` | `app-dev` | app-dev + app-staging |
| `qa-user` | `app-staging` | app-staging (lecture + restart) |
| `prod-deployer` | `app-prod` | app-prod (rolling update CI) |

---

### VALIDATION

```bash
kubectl get serviceaccounts -n app-dev
kubectl get serviceaccounts -n app-staging
kubectl get serviceaccounts -n app-prod
```

```
# app-dev
NAME       SECRETS   AGE
default    0         106m
dev-user   0         95m
PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> kubectl get serviceaccounts -n app-staging
NAME      SECRETS   AGE
default   0         104m
qa-user   0         96m
PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> kubectl get serviceaccounts -n app-prod
NAME            SECRETS   AGE
default         0         104m
prod-deployer   0         96m
```

---

## 3 — Contextes kubectl

Un contexte = triplet **(cluster, user, namespace)**.  
On génère un token par SA puis on crée un contexte nommé pour chaque persona.

```bash
# Générer les tokens (valides 1 an)
DEV_TOKEN=$(kubectl create token dev-user -n app-dev --duration=8760h)
QA_TOKEN=$(kubectl create token qa-user -n app-staging --duration=8760h)
PROD_TOKEN=$(kubectl create token prod-deployer -n app-prod --duration=8760h)

# Enregistrer les credentials
kubectl config set-credentials dev-user --token=$DEV_TOKEN
kubectl config set-credentials qa-user --token=$QA_TOKEN
kubectl config set-credentials prod-deployer --token=$PROD_TOKEN

# Créer les contextes
kubectl config set-context dev-user@app-cluster \
  --cluster=minikube --user=dev-user --namespace=app-dev
kubectl config set-context qa-user@app-cluster \
  --cluster=minikube --user=qa-user --namespace=app-staging
kubectl config set-context prod-deployer@app-cluster \
  --cluster=minikube --user=prod-deployer --namespace=app-prod
```

---

### VALIDATION

```bash
# Avant RoleBindings -> "no" attendu
PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> 
kubectl --context dev-user@app-cluster auth can-i get pods -n app-dev
# no

# Après RoleBindings -> "yes" attendu
PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> 
kubectl --context dev-user@app-cluster auth can-i get pods -n app-dev
# yes
```

---

## 4 — RBAC dev / staging

`dev-user` peut lire, debugger et modifier les déploiements en dev et staging.  
Les secrets sont volontairement exclus (credentials sensibles, trivial à décoder depuis base64).

| Ressource | Permissions dev-user |
|---|---|
| pods, services, configmaps | `get list watch` |
| pods/log, pods/exec | `get create` (debug) |
| deployments, replicasets | `get list watch create update patch delete` |
| **secrets** | **aucun droit** |

Un seul token `dev-user` suffit pour les deux namespaces : le RoleBinding de staging référence `system:serviceaccount:app-dev:dev-user` (cross-namespace binding).

---

### VALIDATION

```bash
kubectl --context dev-user@app-cluster auth can-i create deployment -n app-dev
# yes

kubectl --context dev-user@app-cluster auth can-i get pods/log -n app-staging
# yes

kubectl --context dev-user@app-cluster auth can-i get secrets -n app-dev
# no
```

---

## 5 — RBAC prod (verrouillée)

`dev-user` n'a **aucun droit** en prod (aucun RoleBinding ne le lie).  
`prod-deployer` peut uniquement faire des rolling updates : `get/update/patch` sur les deployments.  
`create` et `delete` sont exclus pour éviter qu'un bug CI supprime un déploiement de prod.

---

### VALIDATION

```bash
kubectl --context dev-user@app-cluster auth can-i get pods -n app-prod
# no

kubectl --context prod-deployer@app-cluster auth can-i patch deployment -n app-prod
# yes

kubectl --context prod-deployer@app-cluster auth can-i delete deployment -n app-prod
# no
```

---

## 6 — Déploiement Kustomize

Structure DRY : une `base/` canonique + des `overlays/` avec uniquement les différences.

```
k8s/
├── base/              # manifest commun
└── overlays/
    ├── dev/           # 1 replica, log_level=debug
    ├── staging/       # 2 replicas, log_level=info
    └── prod/          # 3 replicas, log_level=warn, maxUnavailable=0
```

`maxUnavailable: 0` garantit le zero-downtime : aucun pod existant n'est supprimé avant que son remplaçant soit `Ready`.

---

### VALIDATION

```bash
kubectl apply -k k8s/overlays/dev
kubectl apply -k k8s/overlays/staging
kubectl apply -k k8s/overlays/prod

kubectl get pods -n app-dev      # 1 pod Running
kubectl get pods -n app-staging  # 2 pods Running
kubectl get pods -n app-prod     # 3 pods Running
```

```
PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> kubectl get pods -n app-dev
NAME                       READY   STATUS    RESTARTS   AGE
demo-app-8479bc5b5-vl4bz   1/1     Running   0          17m

PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> kubectl get pods -n app-staging
NAME                        READY   STATUS    RESTARTS   AGE
demo-app-5f45d8f659-gs8jq   1/1     Running   0          17m
demo-app-5f45d8f659-jh5qd   1/1     Running   0          17m

PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> kubectl get pods -n app-prod
NAME                        READY   STATUS    RESTARTS   AGE
demo-app-56cd97bf6b-7ltdv   1/1     Running   0          17m
demo-app-56cd97bf6b-hsvn5   1/1     Running   0          17m
demo-app-56cd97bf6b-sff28   1/1     Running   0          17m
```

---

## 7 — ResourceQuota + LimitRange

**ResourceQuota** : plafonne les totaux par namespace (pods max, CPU total, RAM totale).  
**LimitRange** : fixe les valeurs par défaut par container (évite les containers sans `requests` ou sans `limits`).

| Namespace | pods max | CPU limit | RAM limit |
|---|---|---|---|
| `app-dev` | 10 | 2 | 1 Gi |
| `app-staging` | 20 | 4 | 4 Gi |
| `app-prod` | *(pas de quota, LimitRange seulement)* | — | — |

---

### VALIDATION

```bash
kubectl apply -f k8s/tests/quota-overflow.yaml
```

``` bash
# mrtre@PC-de-Remy MINGW64 ~/source/COURS_DOCKER/tp-5 (main)
# $ kubectl run quota-test --image=busybox:1.36 -n app-dev \
#   --overrides='{"spec":{"containers":[{"name":"quota-test","image":"busybox:1.# 36","resources":{"requests":{"cpu":"3"},"limits":{"cpu":"3"}}}]}}' \
#  -- sleep 3600
# Error from server (Forbidden): pods "quota-test" is forbidden: [maximum cpu usage per Container is 500m, but limi
# t is 3, maximum cpu usage per Pod is 1, but limit is 3]

```

---

## 8 — Pod Security Standards

PSS `restricted` impose 5 contraintes cumulées sur chaque pod.

| Contrainte | Champ YAML requis |
|---|---|
| Non-root | `runAsNonRoot: true` |
| Pas d'escalade | `allowPrivilegeEscalation: false` |
| Pas de capabilities | `capabilities.drop: ["ALL"]` |
| Seccomp | `seccompProfile.type: RuntimeDefault` |
| FS en lecture seule | `readOnlyRootFilesystem: true` |

Notre `base/deployment.yaml` satisfait toutes ces contraintes — les pods peuvent donc tourner en prod.

---

### VALIDATION

```bash
# Tenter de déployer un pod non conforme en prod
kubectl apply -f k8s/tests/bad-pod-pss.yaml -n app-prod
```

``` bash
# PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> kubectl apply -f k8s/tests/bad-pod-pss.yaml -n app-prod 
# Error from server (Forbidden): error when creating "k8s/tests/bad-pod-pss.yaml": pods "bad-pod-pss-test" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "bad-container" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "bad-container" must set securityContext.capabilities.drop=["ALL"]; container "bad-container" must not include "NET_ADMIN" in securityContext.capabilities.add), runAsNonRoot != true (pod or container "bad-container" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "bad-container" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

---

## 9 — NetworkPolicies (prod)

Stratégie **zero-trust réseau** en prod : tout est bloqué par défaut, seuls les flux légitimes sont autorisés explicitement.

```
prod-default-deny          → bloque tout ingress + egress pour tous les pods de app-prod
prod-allow-ingress-controller → autorise uniquement l'ingress-nginx (namespace + pod selector en AND)
prod-allow-egress-dns      → autorise UDP/TCP port 53 vers CoreDNS (sinon DNS cassé)
```

> **Prérequis Minikube** : le CNI par défaut (kindnet) ne supporte pas les NetworkPolicies.  
> Lancer avec : `minikube start --cni=calico --memory=4096`

---

### VALIDATION

```bash
# Pod de test dans app-dev tente d'accéder à app-prod
kubectl run nettest --image=busybox:1.36 --rm -it --restart=Never -n app-dev \
  -- wget -T 3 -O- http://demo-app.app-prod.svc.cluster.local
```

```
wget: bad address 'demo-app.app-prod.svc.cluster.local'
pod "nettest" deleted
pod app-dev/nettest terminated (Error)
```

Trafic bloqué par le `default-deny` de `app-prod`.

---

## 10 — Bonnes pratiques — checklist

Synthèse des garde-fous mis en place dans ce TP.

| Couche | Mécanisme | Objectif |
|---|---|---|
| Identité | RBAC + ServiceAccounts | Principe du moindre privilège |
| Workload | PSS restricted (prod) | Empêche l'escalade de privilèges |
| Ressources | ResourceQuota + LimitRange | Évite les monopolisations de ressources |
| Réseau | NetworkPolicy default-deny | Limite la surface d'attaque latérale |
| Config | Kustomize overlays | Un seul manifeste de référence, 0 duplication |

---

### VALIDATION — vue globale du cluster

```bash
# Vérifier l'état de chaque namespace en une commande
kubectl get pods -A -l app.kubernetes.io/name=demo-app
```

```
NAMESPACE      NAME                        READY   STATUS    RESTARTS
app-dev        demo-app-8479bc5b5-vl4bz    1/1     Running   0
app-staging    demo-app-5f45d8f659-gs8jq   1/1     Running   0
app-staging    demo-app-5f45d8f659-jh5qd   1/1     Running   0
app-prod       demo-app-56cd97bf6b-7ltdv   1/1     Running   0
app-prod       demo-app-56cd97bf6b-hsvn5   1/1     Running   0
app-prod       demo-app-56cd97bf6b-sff28   1/1     Running   0
```

```bash
# Vérifier les rôles créés
kubectl get roles -n app-dev
kubectl get roles -n app-staging
kubectl get roles -n app-prod
```

```
# app-dev
NAME             CREATED AT
developer-role   2026-04-27T13:28:17Z

# app-staging
NAME             CREATED AT
developer-role   2026-04-27T13:28:35Z
qa-role          2026-04-27T13:28:42Z

# app-prod
NAME                 CREATED AT
prod-deployer-role   2026-04-27T13:28:48Z
```

---

## Architecture synthétique

```
Cluster Kubernetes
├── app-dev       [PSS: baseline]   [Quota: 2CPU / 1Gi]
│   └── dev-user  → R/W deployments, debug pods, NO secrets
│
├── app-staging   [PSS: baseline+warn restricted]   [Quota: 4CPU / 4Gi]
│   ├── dev-user  → R/W deployments
│   └── qa-user   → lecture seule + restart pods
│
└── app-prod      [PSS: restricted]   [NetworkPolicy: default-deny]
    ├── dev-user        → AUCUN DROIT
    ├── prod-deployer   → patch/update deployments uniquement
    └── platform-admin  → accès complet (contexte minikube)
```
