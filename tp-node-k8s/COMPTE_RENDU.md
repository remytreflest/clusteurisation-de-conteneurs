# TP #3 — Déploiement depuis un registry privé DockerHub — Compte-rendu

---

## 1. Mise en place du projet Node.js

```bash
mkdir tp-node-k8s && cd tp-node-k8s
npm init -y
npm i express
```

`server.js` expose un serveur Express sur le port `3000` avec :
- `/` → fichier statique `public/index.html`
- `/healthz` → réponse `200 ok` (utilisée par les probes Kubernetes)

---

## 2. Build Docker

### 2.1 Résultat de `docker build`

Build réussi en **deux stages** :
- `deps` : installe les dépendances via `npm ci`
- `runner` : image finale allégée, copie uniquement les artefacts nécessaires

### 2.2 `docker images | grep tp-docker-hub-app`

```
tp-docker-hub-app   latest   f7a121deb22e   < 1 min ago   138MB
```

### 2.3 Test local

```bash
docker run --rm -p 3000:3000 tp-docker-hub-app
curl http://localhost:3000/healthz   # → ok
curl http://localhost:3000/           # → page HTML
```

---

## 3. Push vers DockerHub (registry privé)

```bash
docker tag tp-docker-hub-app docker.io/remytr/tp-docker-hub-app:1.0.0
docker push docker.io/remytr/tp-docker-hub-app:1.0.0
```

**Résultat :**
```
1.0.0: digest: sha256:1de232d36380a9d921f18956f15d6b8effdaa65b10f0842888a05513cef2a689 size: 2196
```

L'image `remytr/tp-docker-hub-app:1.0.0` est visible sur DockerHub en visibilité **privée**.

---

## 4. Secret Kubernetes pour le registry privé

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=remytr \
  --docker-password=<access-token> \
  --docker-email=remytr@dockerhub.com
```

**Vérification :**
```
$ kubectl get secret regcred
NAME      TYPE                             DATA   AGE
regcred   kubernetes.io/dockerconfigjson   1      5s
```

> Le `--docker-password` utilise un **Access Token DockerHub** (et non le mot de passe du compte), ce qui est la pratique recommandée : le token peut être révoqué sans changer le mot de passe principal.

---

## 5. Déploiement sur Kubernetes

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

---

## 6. Vérification des pods

```
$ kubectl get pods -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
tp-docker-hub-app-79b8b6b7c4-fvvjj   1/1     Running   0          37s   10.244.0.15   minikube   <none>           <none>
tp-docker-hub-app-79b8b6b7c4-m2kjg   1/1     Running   0          37s   10.244.0.16   minikube   <none>           <none>
```

2 replicas `Running`, readiness probe validée (délai initial : 3 s, période : 5 s).

---

## 7. Accès à l'application

```bash
kubectl port-forward svc/tp-docker-hub-svc 8080:80
```

```
$ curl http://localhost:8080/healthz
ok

$ curl http://localhost:8080/
<!doctype html>
<html>
  ...
  <body>
    <h1>Bonjour depuis Node.js (conteneurisé) !</h1>
  </body>
</html>
```

La page s'affiche correctement sur `http://localhost:8080`.

---

## Réponses aux questions

### 3 optimisations mises en place dans le Dockerfile

| # | Optimisation | Pourquoi |
|---|---|---|
| 1 | **Multi-stage build** (`deps` + `runner`) | L'image finale ne contient pas les outils de build (npm cache, etc.), ce qui réduit la taille et la surface d'attaque. |
| 2 | **`npm ci` au lieu de `npm install`** | `npm ci` utilise exactement `package-lock.json` (builds reproductibles), échoue si le lock est désynchronisé, et est plus rapide en CI/CD. |
| 3 | **`NODE_ENV=production`** | Express et de nombreux packages désactivent les comportements de développement (logs détaillés, rechargement, middlewares de debug) et réduisent la consommation mémoire en production. |

**Bonus** : le `.dockerignore` exclut `node_modules`, `.git` et les fichiers Docker eux-mêmes → le contexte de build envoyé au daemon est minimal.

---

### À quoi sert `imagePullSecrets` et comment il a été configuré

**Rôle** : Par défaut, Kubernetes ne connaît pas les credentials d'un registry privé. `imagePullSecrets` référence un secret de type `kubernetes.io/dockerconfigjson` que le kubelet utilise pour s'authentifier auprès du registry au moment du `docker pull`.

**Configuration :**

1. **Création du secret** avec `kubectl create secret docker-registry regcred …` en fournissant le serveur, le username et un Access Token DockerHub.

2. **Référencement dans le Deployment** via le champ `spec.imagePullSecrets` :
   ```yaml
   spec:
     imagePullSecrets:
       - name: regcred
     containers:
       - name: web
         image: docker.io/remytr/tp-docker-hub-app:1.0.0
   ```

Sans ce secret, le pod passerait en `ImagePullBackOff` car DockerHub refuserait le pull de l'image privée avec une erreur `401 Unauthorized`.

---

## Bonus réalisés

### `replicas: 2` — Stabilité

Le Deployment a été configuré avec `replicas: 2` dès le départ. Les deux pods démarrent simultanément et passent en `Running` sans erreur, confirmant la stabilité avec plusieurs instances.

### `resources` — Requests et limits

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

- **requests** : garantie minimale pour le scheduling (le scheduler place le pod sur un nœud avec au moins ces ressources disponibles).
- **limits** : plafond — le container est OOM-killed si il dépasse la mémoire, et throttlé si il dépasse le CPU.

---

## Structure du dépôt

```
tp-node-k8s/
├── server.js
├── package.json
├── package-lock.json
├── Dockerfile
├── .dockerignore
├── public/
│   └── index.html
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── COMPTE_RENDU.md
```
