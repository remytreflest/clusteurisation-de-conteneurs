# Compte Rendu — TP #4 : Déploiement Node.js + MySQL sur Kubernetes

## Actions réalisées

### Partie 1 — Conteneurisation

**Build de l'image :**
```bash
docker build -t tp-app-mysql:1.0.0 .
```

**Problème rencontré — Push vers le registry Minikube :**

La commande `docker push localhost:5000/tp-app-mysql:1.0.0` via le port-forward échouait systématiquement avec `context deadline exceeded`. Ce problème est connu sur Windows avec Docker Desktop : le daemon Docker ne peut pas établir la connexion TCP avec le port-forward kubectl dans certaines configurations réseau.

**Solution appliquée — `minikube image load` via tar :**

```bash
# Build avec un tag simple
docker build -t tp-app-mysql:1.0.0 .

# Export en tar puis chargement dans minikube (contourne le problème du registry)
docker save tp-app-mysql:1.0.0 -o /tmp/tp-app-mysql.tar
minikube image load /tmp/tp-app-mysql.tar --overwrite
```

L'image est disponible dans minikube sous `docker.io/library/tp-app-mysql:1.0.0`.

**`minikube image load` propage l'image sur tous les nœuds du cluster automatiquement.** Vérification :

```bash
minikube ssh "docker images tp-app-mysql"
# REPOSITORY     TAG       IMAGE ID       SIZE
# tp-app-mysql   1.0.0     592fae4dbef9   178MB

minikube ssh -n minikube-m02 "docker images tp-app-mysql"
# REPOSITORY     TAG       IMAGE ID       SIZE
# tp-app-mysql   1.0.0     592fae4dbef9   178MB  ← même IMAGE ID
```

C'est indispensable avec `imagePullPolicy: Never` : si un pod est schedulé sur `minikube-m02` et que l'image n'y est pas, Kubernetes retournerait une erreur `ErrImageNeverPull` et le pod ne démarrerait jamais.

Le manifest `app-deployment.yaml` utilise donc :
```yaml
image: tp-app-mysql:1.0.0
imagePullPolicy: Never
```

---

### Partie 2 — Namespace

```bash
kubectl create namespace tp-app-mysql
kubectl config set-context --current --namespace=tp-app-mysql
```

---

### Partie 3 à 10 — Application des manifests

Tous les fichiers YAML ont été créés et appliqués dans cet ordre :

```bash
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f app-configmap.yaml
kubectl apply -f app-deployment.yaml
kubectl apply -f app-service.yaml
kubectl apply -f app-ingress.yaml
```

Activation du metrics-server pour le HPA :
```bash
minikube addons enable metrics-server
kubectl apply -f app-hpa.yaml
```

---

### Problème majeur — Réseau cross-nœud cassé

**Symptôme :** L'endpoint `/` retournait `{"error":"connect EHOSTUNREACH 10.244.1.6:3306"}`. Le pod `node-app` sur le nœud `minikube` ne pouvait pas joindre le pod MySQL sur `minikube-m02`.

**Diagnostic :** La route dans les pods du nœud `minikube` est :
```
10.244.0.0/16 dev eth0 scope link src 10.244.0.37
```

Avec `scope link` sur un `/16`, le pod pense que toutes les adresses `10.244.x.x` sont sur le même segment L2 et tente un ARP direct pour `10.244.1.6`. Cet ARP ne reçoit pas de réponse car le pod est sur un autre nœud (un autre container Docker). Il s'agit d'un bug connu de kindnet avec le driver Docker en multi-nœuds sur Minikube.

**Solution appliquée — Proxy ARP sur le nœud :**

```bash
minikube ssh "sudo sysctl -w net.ipv4.conf.bridge.proxy_arp=1"
minikube ssh "sudo sysctl -w net.ipv4.conf.all.proxy_arp=1"
```

Le proxy ARP permet au bridge du nœud `minikube` de répondre aux requêtes ARP pour les IPs des autres nœuds avec sa propre MAC, ce qui force le trafic à passer par le routage L3. La connectivité cross-nœud est ainsi rétablie.

> **Note :** Ce correctif est temporaire (perdu au redémarrage du nœud). En production, cette situation ne se produit pas car les drivers hypervisor (VirtualBox, Hyper-V, KVM) gèrent correctement le réseau multi-nœuds.

---

### Partie 8 — Ajout de l'entrée hosts (Ingress)

L'entrée `192.168.49.2 tp-app-mysql.local` a été ajoutée dans `C:\Windows\System32\drivers\etc\hosts`.

**Limitation du driver Docker sur Windows :** L'IP `192.168.49.2` n'est pas directement routable depuis l'hôte Windows. L'Ingress est testé via port-forward sur l'ingress controller :

```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8081:80

curl -H "Host: tp-app-mysql.local" http://localhost:8081/
# → {"message":"Connexion MySQL OK","time":"2026-04-01T10:10:16.000Z"}

curl -H "Host: tp-app-mysql.local" http://localhost:8081/health
# → {"status":"ok"}
```

**Alternative — `minikube tunnel` (accès navigateur direct) :**

Le service `ingress-nginx-controller` étant en `NodePort` par défaut, `minikube tunnel` ne l'expose pas automatiquement. Il faut d'abord le passer en `LoadBalancer` :

```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec":{"type":"LoadBalancer"}}'
```

`minikube tunnel` lui attribue alors l'IP externe `127.0.0.1` :

```
NAME                       TYPE           EXTERNAL-IP   PORT(S)
ingress-nginx-controller   LoadBalancer   127.0.0.1     80:30477/TCP,443:31478/TCP
```

L'entrée hosts doit être mise à jour pour pointer vers `127.0.0.1` (au lieu de `192.168.49.2`). Dans un PowerShell **administrateur** :

```powershell
(Get-Content "C:\Windows\System32\drivers\etc\hosts") `
  -replace "192.168.49.2 tp-app-mysql.local", "127.0.0.1 tp-app-mysql.local" `
  | Set-Content "C:\Windows\System32\drivers\etc\hosts"
```

Lancer le tunnel dans un terminal PowerShell **administrateur dédié** (à laisser ouvert) :

```powershell
minikube tunnel
# → "Tunnel de démarrage pour le service node-app-ingress."
```

`http://tp-app-mysql.local` est alors accessible directement depuis le navigateur.

---

## Vérification finale

### État des ressources

```
NAME                            READY   STATUS    NODE
pod/mysql-759c94f598-47hp9      1/1     Running   minikube-m02
pod/node-app-5c85f656b9-8p4dt   1/1     Running   minikube
pod/node-app-5c85f656b9-9767x   1/1     Running   minikube-m02

service/mysql              ClusterIP   None           3306/TCP
service/node-app-service   NodePort    10.102.88.45   80:30080/TCP

deployment.apps/mysql      1/1
deployment.apps/node-app   2/2

horizontalpodautoscaler.autoscaling/node-app-hpa
  Targets: 16%/70% memory, 1%/50% CPU — Replicas: 2/6
```

### Tests réussis

```bash
# Via port-forward (Git Bash / WSL)
kubectl port-forward -n tp-app-mysql service/node-app-service 8080:80

curl http://localhost:8080/health
# → {"status":"ok"}

curl http://localhost:8080/
# → {"message":"Connexion MySQL OK","time":"2026-04-01T10:05:27.000Z"}
```

### Répartition des pods sur les nœuds

Grâce à la règle `podAntiAffinity`, les deux réplicas `node-app` sont sur des nœuds différents :
- `node-app-...-8p4dt` → `minikube` (control-plane)
- `node-app-...-9767x` → `minikube-m02` (worker)

### PVC

```
mysql-pvc   Bound   pvc-5847c723-...   1Gi   RWO   standard
```

---

## Structure des fichiers

```
tp-4/
├── index.js                  # Application Node.js
├── package.json
├── Dockerfile
├── mysql-secret.yaml         # Secret (credentials MySQL)
├── mysql-pvc.yaml            # PersistentVolumeClaim 1Gi
├── mysql-deployment.yaml     # Deployment MySQL (1 replica)
├── mysql-service.yaml        # Service headless (clusterIP: None)
├── app-configmap.yaml        # ConfigMap (DB_HOST, DB_PORT, PORT)
├── app-deployment.yaml       # Deployment node-app (2 replicas + anti-affinity)
├── app-service.yaml          # NodePort :30080
├── app-ingress.yaml          # Ingress → tp-app-mysql.local
└── app-hpa.yaml              # HPA 2-6 replicas (CPU 50%, RAM 70%)
```

---

## Réponses aux questions

**1. Secret vs ConfigMap ?**

Un `Secret` stocke des données sensibles (mots de passe, tokens, clés API) encodées en base64. Un `ConfigMap` stocke de la configuration non sensible en clair. Utiliser Secret pour les credentials MySQL, ConfigMap pour les paramètres de connexion non sensibles (host, port, nom de base).

**2. Pourquoi MySQL avec replicas: 1 ?**

MySQL est une base de données avec état. Avoir plusieurs replicas nécessite un mécanisme de réplication (MySQL Group Replication, Galera, etc.) et une gestion des conflits d'écriture. Sans cela, plusieurs instances écrivant sur le même PVC provoqueraient une corruption des données. Le bon outil pour des bases répliquées est le `StatefulSet`.

**3. Suppression du PVC et recréation du pod MySQL ?**

Si on supprime le PVC, les données sont perdues (le PV est libéré). À la recréation, un nouveau PVC vide est créé — la base repart de zéro. Conclusion : le PVC est le seul garant de la persistance. Sans lui, chaque redémarrage de pod perd toutes les données.

**4. Rôle de podAntiAffinity ?**

`podAntiAffinity` empêche (ou décourage) deux pods portant le même label d'être planifiés sur le même nœud (`topologyKey: kubernetes.io/hostname`). Sans cette règle, Kubernetes pourrait placer les 2 réplicas `node-app` sur le même nœud, créant un Single Point of Failure : si ce nœud tombe, l'application est totalement indisponible.

**5. ClusterIP vs NodePort vs LoadBalancer ?**

| Type | Accessible depuis | Usage |
|------|-------------------|-------|
| ClusterIP | Intérieur du cluster uniquement | Communication interne entre services |
| NodePort | Extérieur, via IP:port du nœud | Développement/test local |
| LoadBalancer | Extérieur, via IP dédiée | Production (cloud provider requis) |

**6. Pourquoi imagePullPolicy: Never ici ?**

L'image est chargée directement dans minikube (non publiée sur un registry public). Sans `Never`, Kubernetes essaierait de la télécharger depuis Docker Hub et échouerait. En production, l'image est dans un registry (ECR, GCR, ACR…) et Kubernetes peut toujours la télécharger — `Never` n'est pas approprié.

**7. HPA vs VPA ?**

| | HPA | VPA |
|--|-----|-----|
| Scale | Horizontal (nombre de pods) | Vertical (CPU/RAM par pod) |
| Besoin | Charge variable, app stateless | App difficile à scaler horizontalement |
| Conflit | Ne pas utiliser avec replicas fixes élevés | Ne pas utiliser avec HPA sur les mêmes métriques |

**8. Pourquoi le scale down est plus lent ?**

Pour éviter les oscillations (flapping) : si la charge redescend brièvement, réduire immédiatement les pods puis les recréer quelques secondes plus tard est coûteux et risqué. Le délai de scale down par défaut est 5 minutes (`--horizontal-pod-autoscaler-downscale-stabilization`), ce qui lisse les pics temporaires.

**9. StatefulSet vs Deployment pour une base de données répliquée ?**

Le `StatefulSet` garantit :
- Une **identité stable** par pod (nom prévisible : `mysql-0`, `mysql-1`) utilisée pour la réplication
- Un **PVC dédié par pod** (chaque instance a son propre stockage)
- Un **démarrage/arrêt ordonné** (important pour les primaires/réplicas)

Le `Deployment` ne garantit rien de tout cela — les pods sont interchangeables et partagent le même PVC, ce qui est incompatible avec une réplication MySQL correcte.

---

## Points de vigilance

- Le correctif `proxy_arp` est perdu au redémarrage de minikube → à ré-appliquer si besoin
- L'IP minikube (`192.168.49.2`) n'est pas routable directement depuis Windows avec le driver Docker — utiliser port-forward ou `minikube tunnel`
- Le driver Docker de Minikube sur Windows présente des limitations réseau en multi-nœuds non présentes avec les drivers hyperviseurs
