# TP #5 — Environments dev/staging/prod avec Namespaces & RBAC

## Pré-requis

- Minikube avec l'addon `ingress` activé (pour les NetworkPolicies ingress-nginx)
- Kustomize installé (`brew install kustomize` ou inclus dans `kubectl` >= 1.14)
- Minikube démarré avec un CNI supportant les NetworkPolicies

```bash
# Démarrer Minikube avec Calico (CNI supportant NetworkPolicies)
minikube start --cni=calico --memory=4096

# Ou si Minikube est déjà lancé, activer l'addon ingress
minikube addons enable ingress
```

---

## Installation — étape par étape

### 1. Créer les namespaces

```bash
kubectl apply -f k8s/namespaces/app-dev.yaml
kubectl apply -f k8s/namespaces/app-staging.yaml
kubectl apply -f k8s/namespaces/app-prod.yaml

# Vérification
kubectl get ns --show-labels | grep app-
```

**Résultat attendu :**
```
app-dev       Active   ...  environment=dev,owner=platform-team,...
app-staging   Active   ...  environment=staging,owner=platform-team,...
app-prod      Active   ...  environment=prod,owner=platform-team,...
```

### 2. Appliquer RBAC

```bash
kubectl apply -f k8s/rbac/serviceaccounts.yaml
kubectl apply -f k8s/rbac/dev-developer-role.yaml
kubectl apply -f k8s/rbac/dev-developer-binding.yaml
kubectl apply -f k8s/rbac/staging-developer-role.yaml
kubectl apply -f k8s/rbac/staging-developer-binding.yaml
kubectl apply -f k8s/rbac/staging-qa-role.yaml
kubectl apply -f k8s/rbac/staging-qa-binding.yaml
kubectl apply -f k8s/rbac/prod-deployer-role.yaml
kubectl apply -f k8s/rbac/prod-deployer-binding.yaml
```

### 3. Créer les contextes kubectl (simulation utilisateurs)

```bash
# Générer les tokens (valides 1 an)
DEV_TOKEN=$(kubectl create token dev-user -n app-dev --duration=8760h)

eyJhbGciOiJSUzI1NiIsImtpZCI6ImFhQno4dTdLTndNUWtHaXVzSjU5UHRNUXhVVXl2cmxLNngwcVViU0c5elUifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODA4ODM0MDA1LCJpYXQiOjE3NzcyOTgwMDUsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJhcHAtZGV2Iiwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRldi11c2VyIiwidWlkIjoiYzE1MjdiZjgtYmJkMC00ODRlLWE0MTEtYzEzMzBjODkyMDJlIn19LCJuYmYiOjE3NzcyOTgwMDUsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDphcHAtZGV2OmRldi11c2VyIn0.mzJanK5vo13BE62WxEf5lZ-aHoiv56iQu1-TdefH_13f6sE57ibc2z_S2aiAGlB364OrLxkV519VeSyPRmWdgJ0BNRZki9ipVyk8WAm-6SUbSMNcDWuGxhnZP1BP0B0Nc-P-iceIvhQn5GrmAxi4ODWHPNeDzCvPa_KdyiqBkZ47c-GIY67KKMY-L8IMMltuMwQ1hIqvbDgOUuzx4jVWm8wHMY9soG_bjQavvuZqXTejqk0zxzyrcRCVSrHZ1C_Ehft6PRBwHYTxiAgB9KFErbpQ8PBNCH5EcciKqA-eZauRJ9NZKIcYF66S_3vHwu6Xdx9GCmx68YGuWN9V6LL06Q

QA_TOKEN=$(kubectl create token qa-user -n app-staging --duration=8760h)

eyJhbGciOiJSUzI1NiIsImtpZCI6ImFhQno4dTdLTndNUWtHaXVzSjU5UHRNUXhVVXl2cmxLNngwcVViU0c5elUifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODA4ODM0MDIyLCJpYXQiOjE3NzcyOTgwMjIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJhcHAtc3RhZ2luZyIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJxYS11c2VyIiwidWlkIjoiNmFiZWVmNTYtZDEyNS00ZjFjLTkyMzktZDQzZjQxMGE0NzRkIn19LCJuYmYiOjE3NzcyOTgwMjIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDphcHAtc3RhZ2luZzpxYS11c2VyIn0.UiGRjTvNqMeXDj_Tl6t6tlmhQWDRoh3le3mgdMTuxdmMt6Y2pQXB1v2CYbIJAt9eG0eDw_PhKM-jQnMXsHGk5UsMOfZDyeiKltQ6wdX1ghoNdqO6GhmXOrtpUDTSnKqpFefmYJu7tD8iJt0_P7v0-Umhy1d3xR_F2kEquPI0YrQLU55uVwCO3J-seHmms9gN2HHaneLSb_GrF8Mgd9eOhNCr5n-mS878qc8ui1fcm9FfpQkaDd6-0uccT7hwo7EgEXnaBqw4vmm-N74OyzPuvyFSqGcYXvPvu3JPqCOm6HeSjiiELh_j-Pxe-1-wzv4bl3ZB_sW8a0a7r6NlvI0bJg

PROD_TOKEN=$(kubectl create token prod-deployer -n app-prod --duration=8760h)

eyJhbGciOiJSUzI1NiIsImtpZCI6ImFhQno4dTdLTndNUWtHaXVzSjU5UHRNUXhVVXl2cmxLNngwcVViU0c5elUifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODA4ODM0MDM1LCJpYXQiOjE3NzcyOTgwMzUsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJhcHAtcHJvZCIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJwcm9kLWRlcGxveWVyIiwidWlkIjoiZmZkMTQxZDMtZjg5MS00NmI5LWJiMmEtMjcwN2M0ZWQ4MDliIn19LCJuYmYiOjE3NzcyOTgwMzUsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDphcHAtcHJvZDpwcm9kLWRlcGxveWVyIn0.iwvbD8e8y46rs2j7jaxXBpGKaZFOPOmobTPGx0C5m9Z6qz2cZDxg7jkibx6afGj3BDFb3xY6Z84mSubFIbU3WLDAW4JjK6pI1mkkxu1Nfk30M_KiCSFYaPQjT7YJq6S5JOHhiQ02wbuiBKiJ6tZgDxD9lHdCHV6zR8rqgTFJiuHf-9RpAJjyRoGey4mJKGBRUFUvMqTw8vBDj75NQHO6uk7mkmvHPxrP5gQ965up0vDYO9_mwZuQTPJ7SOLBJg3PuPO8jcBfuky0Pw6TutAsvm4dzY5mSMSmQogRTsEra8N66sADE5bcn7gmqzODyiE4aoNgpUVNHwhuipv7fGvANg

# Cluster actuel (minikube)
CLUSTER_SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

https://127.0.0.1:54345

CA_DATA=$(kubectl config view --minify --flatten -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')

# Enregistrer les credentials
kubectl config set-credentials dev-user --token="eyJhbGciOiJSUzI1NiIsImtpZCI6ImFhQno4dTdLTndNUWtHaXVzSjU5UHRNUXhVVXl2cmxLNngwcVViU0c5elUifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODA4ODM0MDA1LCJpYXQiOjE3NzcyOTgwMDUsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJhcHAtZGV2Iiwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRldi11c2VyIiwidWlkIjoiYzE1MjdiZjgtYmJkMC00ODRlLWE0MTEtYzEzMzBjODkyMDJlIn19LCJuYmYiOjE3NzcyOTgwMDUsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDphcHAtZGV2OmRldi11c2VyIn0.mzJanK5vo13BE62WxEf5lZ-aHoiv56iQu1-TdefH_13f6sE57ibc2z_S2aiAGlB364OrLxkV519VeSyPRmWdgJ0BNRZki9ipVyk8WAm-6SUbSMNcDWuGxhnZP1BP0B0Nc-P-iceIvhQn5GrmAxi4ODWHPNeDzCvPa_KdyiqBkZ47c-GIY67KKMY-L8IMMltuMwQ1hIqvbDgOUuzx4jVWm8wHMY9soG_bjQavvuZqXTejqk0zxzyrcRCVSrHZ1C_Ehft6PRBwHYTxiAgB9KFErbpQ8PBNCH5EcciKqA-eZauRJ9NZKIcYF66S_3vHwu6Xdx9GCmx68YGuWN9V6LL06Q"
kubectl config set-credentials qa-user --token="eyJhbGciOiJSUzI1NiIsImtpZCI6ImFhQno4dTdLTndNUWtHaXVzSjU5UHRNUXhVVXl2cmxLNngwcVViU0c5elUifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODA4ODM0MDIyLCJpYXQiOjE3NzcyOTgwMjIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJhcHAtc3RhZ2luZyIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJxYS11c2VyIiwidWlkIjoiNmFiZWVmNTYtZDEyNS00ZjFjLTkyMzktZDQzZjQxMGE0NzRkIn19LCJuYmYiOjE3NzcyOTgwMjIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDphcHAtc3RhZ2luZzpxYS11c2VyIn0.UiGRjTvNqMeXDj_Tl6t6tlmhQWDRoh3le3mgdMTuxdmMt6Y2pQXB1v2CYbIJAt9eG0eDw_PhKM-jQnMXsHGk5UsMOfZDyeiKltQ6wdX1ghoNdqO6GhmXOrtpUDTSnKqpFefmYJu7tD8iJt0_P7v0-Umhy1d3xR_F2kEquPI0YrQLU55uVwCO3J-seHmms9gN2HHaneLSb_GrF8Mgd9eOhNCr5n-mS878qc8ui1fcm9FfpQkaDd6-0uccT7hwo7EgEXnaBqw4vmm-N74OyzPuvyFSqGcYXvPvu3JPqCOm6HeSjiiELh_j-Pxe-1-wzv4bl3ZB_sW8a0a7r6NlvI0bJg"
kubectl config set-credentials prod-deployer --token="eyJhbGciOiJSUzI1NiIsImtpZCI6ImFhQno4dTdLTndNUWtHaXVzSjU5UHRNUXhVVXl2cmxLNngwcVViU0c5elUifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODA4ODM0MDM1LCJpYXQiOjE3NzcyOTgwMzUsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJhcHAtcHJvZCIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJwcm9kLWRlcGxveWVyIiwidWlkIjoiZmZkMTQxZDMtZjg5MS00NmI5LWJiMmEtMjcwN2M0ZWQ4MDliIn19LCJuYmYiOjE3NzcyOTgwMzUsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDphcHAtcHJvZDpwcm9kLWRlcGxveWVyIn0.iwvbD8e8y46rs2j7jaxXBpGKaZFOPOmobTPGx0C5m9Z6qz2cZDxg7jkibx6afGj3BDFb3xY6Z84mSubFIbU3WLDAW4JjK6pI1mkkxu1Nfk30M_KiCSFYaPQjT7YJq6S5JOHhiQ02wbuiBKiJ6tZgDxD9lHdCHV6zR8rqgTFJiuHf-9RpAJjyRoGey4mJKGBRUFUvMqTw8vBDj75NQHO6uk7mkmvHPxrP5gQ965up0vDYO9_mwZuQTPJ7SOLBJg3PuPO8jcBfuky0Pw6TutAsvm4dzY5mSMSmQogRTsEra8N66sADE5bcn7gmqzODyiE4aoNgpUVNHwhuipv7fGvANg"

# Créer les contextes
kubectl config set-context dev-user@app-cluster --cluster=minikube --user=dev-user --namespace=app-dev

kubectl config set-context qa-user@app-cluster --cluster=minikube --user=qa-user --namespace=app-staging

kubectl config set-context prod-deployer@app-cluster --cluster=minikube --user=prod-deployer --namespace=app-prod
```

### 4. Appliquer les quotas et LimitRanges

```bash
kubectl apply -f k8s/quotas/dev-resourcequota.yaml
kubectl apply -f k8s/quotas/dev-limitrange.yaml
kubectl apply -f k8s/quotas/staging-resourcequota.yaml
kubectl apply -f k8s/quotas/staging-limitrange.yaml
kubectl apply -f k8s/quotas/prod-limitrange.yaml
```

### 5. Déployer l'application (Kustomize)

```bash
# Dev
kubectl apply -k k8s/overlays/dev

PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> kubectl apply -k k8s/overlays/dev
configmap/demo-app-config created
service/demo-app created
deployment.apps/demo-app created

# Staging
kubectl apply -k k8s/overlays/staging

PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> kubectl apply -k k8s/overlays/staging
configmap/demo-app-config created
service/demo-app created
deployment.apps/demo-app created

# Prod
kubectl apply -k k8s/overlays/prod

PS C:\Users\mrtre\source\COURS_DOCKER\tp-5> kubectl apply -k k8s/overlays/prod
configmap/demo-app-config created
service/demo-app created
deployment.apps/demo-app created
```

### 6. Appliquer les NetworkPolicies (prod)

```bash
kubectl apply -f k8s/network/prod-default-deny.yaml
kubectl apply -f k8s/network/prod-allow-ingress-controller.yaml
kubectl apply -f k8s/network/prod-allow-egress-dns.yaml
```

---

## Commandes de test (preuves)

### Test namespaces + labels

```bash
kubectl get ns --show-labels | grep app-
```

### Test RBAC — dev-user en dev/staging (doit répondre "yes")

```bash
kubectl --context dev-user@app-cluster auth can-i create deployment -n app-dev
# => yes

kubectl --context dev-user@app-cluster auth can-i get pods/log -n app-staging
# => yes
```

### Test RBAC — dev-user BLOQUÉ en prod (doit répondre "no")

```bash
kubectl --context dev-user@app-cluster auth can-i get pods -n app-prod
# => no

kubectl --context dev-user@app-cluster auth can-i create deployment -n app-prod
# => no
```

### Test RBAC — prod-deployer en prod (doit répondre "yes")

```bash
kubectl --context prod-deployer@app-cluster auth can-i patch deployment -n app-prod
# => yes

kubectl --context prod-deployer@app-cluster auth can-i delete deployment -n app-prod
# => no  (delete interdit)
```

### Test app — vérifier l'environnement dans la réponse HTTP

```bash
# Port-forward vers chaque env
kubectl port-forward svc/demo-app 8080:80 -n app-dev &
curl http://localhost:8080
# => env=dev | log_level=debug | demo-app v1

kubectl port-forward svc/demo-app 8081:80 -n app-staging &
curl http://localhost:8081
# => env=staging | log_level=info | demo-app v1

kubectl port-forward svc/demo-app 8082:80 -n app-prod &
curl http://localhost:8082
# => env=prod | log_level=warn | demo-app v1
```

### Test PSS — pod non conforme rejeté en prod

```bash
kubectl apply -f k8s/tests/bad-pod-pss.yaml -n app-prod
# => Error : pods "bad-pod-pss-test" is forbidden:
#    violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false, ...

kubectl apply -f k8s/tests/bad-pod-pss.yaml -n app-dev
# => pod/bad-pod-pss-test created  (baseline tolère)
kubectl delete pod bad-pod-pss-test -n app-dev
```

### Test quota — dépassement rejeté en dev

```bash
kubectl apply -f k8s/tests/quota-overflow.yaml
# => Error : exceeded quota: dev-quota, requested: limits.cpu=3,
#    used: limits.cpu=0, limited: limits.cpu=2
```

### Test NetworkPolicy — isolation prod

```bash
# Lancer un pod de test dans app-dev
kubectl run nettest --image=busybox:1.36 --rm -it --restart=Never -n app-dev \
  -- wget -T 3 http://demo-app.app-prod.svc.cluster.local
# => wget: download timed out  (bloqué par default-deny)
```

---

## Tableau RBAC — Qui a le droit de faire quoi

```
+------------------+-----------------------------+------------------------------+------------------+
| Action           | app-dev                     | app-staging                  | app-prod         |
+------------------+-----------------------------+------------------------------+------------------+
| get/list pods    | dev-user [YES]              | dev-user [YES], qa-user [YES]| prod-deployer [YES] |
| pods/log         | dev-user [YES]              | dev-user [YES], qa-user [YES]| prod-deployer [YES] |
| pods/exec        | dev-user [YES]              | dev-user [YES]               | PERSONNE         |
| create deploy    | dev-user [YES]              | dev-user [YES]               | PERSONNE         |
| update/patch dep | dev-user [YES]              | dev-user [YES], qa-user[YES]*| prod-deployer [YES] |
| delete deploy    | dev-user [YES]              | dev-user [YES]               | PERSONNE         |
| get/list secrets | PERSONNE                    | PERSONNE                     | PERSONNE         |
| create service   | dev-user [YES]              | dev-user [YES]               | PERSONNE         |
| rollout restart  | dev-user [YES]              | dev-user [YES], qa-user [YES]| PERSONNE         |
+------------------+-----------------------------+------------------------------+------------------+
* qa-user : patch uniquement (pas create/delete)
```

**Résumé des personas :**
- **platform-admin** : admin cluster complet (vous — contexte minikube par défaut)
- **dev-user** : déploiement libre en dev + staging, aucun droit en prod
- **qa-user** : lecture + logs + rollout restart en staging uniquement
- **prod-deployer** : rolling update uniquement en prod (typiquement le CI/CD)

---

## Bonnes pratiques — checklist (mémo)

### Séparation des environnements
- [x] Un namespace par environnement (app-dev, app-staging, app-prod)
- [x] Labels standards (`environment`, `owner`, `app.kubernetes.io/part-of`) pour le filtrage et la facturation
- [x] Jamais de ressources multi-env dans le même namespace

### RBAC — principe du moindre privilège
- [x] Roles namespace-scoped (pas ClusterRole sauf nécessité)
- [x] Un RoleBinding par persona par namespace — pas de wildcard `*` sur les verbes
- [x] Jamais `get/list` secrets pour les devs (exfiltration involontaire)
- [x] prod-deployer limité à `patch/update` deployment (pas `create/delete`)
- [x] Rotation régulière des tokens de service account

### Secrets — stratégie
- [ ] Utiliser External Secrets Operator ou Sealed Secrets (pas de secrets en clair dans Git)
- [x] Aucun accès secrets pour dev-user et qa-user
- [ ] Secrets injectés par le pipeline CI uniquement
- [ ] Audit des accès secrets avec `kubectl get events` ou un SIEM

### Quotas et LimitRanges
- [x] ResourceQuota sur chaque namespace (CPU, RAM, nombre de pods)
- [x] LimitRange avec `defaultRequest` pour que les pods sans requests reçoivent des valeurs raisonnables
- [x] `min` dans LimitRange : interdit les requests=0 (anti-pattern scheduler)
- [x] Quotas progressifs : dev < staging <= prod

### Pod Security Standards
- [x] `baseline` en dev/staging : bloque les configs vraiment dangereuses (privileged, hostPID...)
- [x] `restricted` en prod : runAsNonRoot, allowPrivilegeEscalation:false, capabilities drop ALL, seccompProfile
- [x] `warn` et `audit` activés en staging pour préparer la migration vers restricted
- [x] Tester les manifests avec `--dry-run=server` avant d'appliquer en prod

### NetworkPolicies
- [x] Default deny ingress + egress en prod (zero trust réseau)
- [x] Autoriser explicitement : DNS (port 53), ingress controller, accès DB si besoin
- [x] Aucune NetworkPolicy en dev (trop contraignant pour le debug)
- [ ] Documenter chaque NetworkPolicy avec le flux qu'elle autorise

### Stratégie de release + rollback
- [x] RollingUpdate avec `maxUnavailable: 0` (zero downtime)
- [ ] Utiliser `kubectl rollout history` pour auditer les déploiements
- [ ] Rollback : `kubectl rollout undo deployment/demo-app -n app-prod`
- [ ] Intégrer un health check dans le pipeline CI avant de promouvoir staging -> prod

### Politique images
- [ ] Registry privé (Harbor, ECR, GCR) — jamais Docker Hub en prod sans scan
- [ ] Scan de vulnérabilités obligatoire (Trivy, Snyk) avant push
- [ ] Signature d'image (cosign + Sigstore) + Admission Controller (Kyverno/OPA) pour vérifier
- [ ] `imagePullPolicy: Always` en staging/prod (jamais `:latest` avec `IfNotPresent`)
- [ ] Tag immuable : `image:sha256-<digest>` ou `image:<semver>` — pas `:latest`
