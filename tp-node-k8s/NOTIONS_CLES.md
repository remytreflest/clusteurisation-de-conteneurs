# Notions clés — Docker + Kubernetes + Minikube

---

## Docker

### Image vs Conteneur
- Une **image** est un modèle figé (comme une classe en POO) : elle contient le code, les dépendances, la configuration.
- Un **conteneur** est une instance en cours d'exécution de cette image (comme un objet instancié).
- Une image ne change jamais. On en crée autant de conteneurs qu'on veut.

### Dockerfile
Fichier de recette pour construire une image. Chaque instruction (`FROM`, `COPY`, `RUN`…) crée une **couche** dans l'image. Les couches sont mises en cache : si une ligne ne change pas, Docker réutilise le cache → build plus rapide.

### Multi-stage build
On utilise plusieurs `FROM` dans un même Dockerfile :
- **Stage 1** (`deps`) : installe les dépendances, utilise des outils de build.
- **Stage 2** (`runner`) : image finale, on ne copie que ce qui est strictement nécessaire depuis le stage 1.

Résultat : l'image finale est légère et ne contient pas les outils de build (réduction de la taille et de la surface d'attaque).

### `npm ci` vs `npm install`
- `npm install` résout les versions et peut modifier `package-lock.json`.
- `npm ci` installe **exactement** ce qui est dans `package-lock.json`, sans le modifier. Reproductible, plus rapide en CI/CD, échoue si le lock est désynchronisé.

### `.dockerignore`
Fonctionne comme `.gitignore` mais pour le **contexte de build Docker**. Tout ce qui est listé n'est pas envoyé au daemon Docker lors du `docker build`. Important d'y mettre `node_modules` (lourd et inutile car réinstallé dans le conteneur).

### Registry
Serveur qui stocke et distribue des images Docker. Exemples :
- **DockerHub** : registry public/privé par défaut.
- `docker push` envoie une image vers le registry.
- `docker pull` la télécharge.
- Une image est identifiée par : `registry/utilisateur/nom:tag` (ex: `docker.io/remytr/tp-docker-hub-app:1.0.0`).

---

## Kubernetes (K8s)

### Pourquoi Kubernetes ?
Docker gère un conteneur sur une machine. Kubernetes gère **des dizaines/centaines de conteneurs sur plusieurs machines**, avec : démarrage automatique, redémarrage en cas de crash, mise à jour sans interruption, répartition de charge.

### Les composants du cluster

| Composant | Rôle |
|---|---|
| **kube-apiserver** | Point d'entrée de toutes les commandes `kubectl`. Cerveau du cluster. |
| **etcd** | Base de données clé-valeur qui stocke tout l'état du cluster. |
| **kube-scheduler** | Décide sur quel nœud placer un nouveau pod. |
| **kube-controller-manager** | Surveille en permanence l'état réel vs l'état désiré et corrige les écarts. |
| **kubelet** | Agent sur chaque nœud, lance et surveille les conteneurs. |

### Pod
Unité de base de Kubernetes. Un pod contient un ou plusieurs conteneurs qui partagent le même réseau et le même stockage. En pratique, un pod = un conteneur la plupart du temps. Les pods sont **éphémères** : ils peuvent être tués et recréés à tout moment.

### Deployment
Objet qui décrit **l'état désiré** : quelle image, combien de replicas, quelle stratégie de mise à jour. Le controller-manager surveille en permanence que cet état est respecté. Si un pod meurt, il en recrée un automatiquement.

### ReplicaSet
Créé automatiquement par le Deployment. Il garantit qu'un nombre précis de pods identiques tournent en permanence. On ne le manipule généralement pas directement.

### Service
Les pods ont des IPs qui changent à chaque recréation. Le **Service** est une adresse stable (IP fixe + DNS interne) qui pointe vers un groupe de pods via des **labels**. Il fait aussi la répartition de charge entre les pods.

Types de Service :
- `ClusterIP` (défaut) : accessible uniquement depuis l'intérieur du cluster.
- `NodePort` : expose un port sur chaque nœud du cluster.
- `LoadBalancer` : crée un load balancer externe (cloud uniquement).

### Namespace
Mécanisme d'isolation logique dans le cluster. Permet de regrouper des ressources (pods, services…) par projet ou environnement. Exemple : `default`, `kube-system`, `tp-minikube`.

### Labels et Selectors
Les labels sont des paires clé/valeur attachées aux ressources (`app: mon-app`). Les selectors permettent de **cibler** des ressources par leurs labels. C'est ce qui relie un Service à ses pods et un Deployment à ses pods.

### imagePullSecrets
Quand l'image Docker est sur un registry **privé**, Kubernetes doit s'authentifier pour la télécharger. On crée un secret de type `docker-registry` qui contient les credentials, puis on le référence dans le Deployment via `imagePullSecrets`. Sans cela : erreur `ImagePullBackOff`.

### Probes (sondes de santé)
Mécanismes par lesquels Kubernetes vérifie qu'un conteneur fonctionne correctement :
- **readinessProbe** : est-ce que le pod est **prêt à recevoir du trafic** ? Tant qu'elle échoue, le Service ne lui envoie pas de requêtes.
- **livenessProbe** : est-ce que le pod est **vivant** ? Si elle échoue trop souvent, Kubernetes redémarre le conteneur.

### Resources (requests & limits)
- **requests** : ressources *garanties* au conteneur. Le scheduler utilise ces valeurs pour choisir le nœud.
- **limits** : ressources *maximales*. Le conteneur est tué (OOM) s'il dépasse la mémoire, ou ralenti s'il dépasse le CPU.

### RollingUpdate
Stratégie de déploiement par défaut. Lors d'une mise à jour, Kubernetes crée les nouveaux pods **avant** de supprimer les anciens → zéro interruption de service. On peut faire un `kubectl rollout undo` pour revenir en arrière.

---

## Minikube

Outil qui crée un cluster Kubernetes **complet en local** dans un conteneur Docker. Il joue le rôle d'un vrai cluster sur une seule machine. Utile pour apprendre et tester sans infrastructure cloud.

### Pourquoi `port-forward` ?
Les Services `ClusterIP` ne sont accessibles que depuis l'intérieur du cluster. Minikube tournant dans un conteneur Docker isolé du réseau hôte, `kubectl port-forward` crée un **tunnel temporaire** entre un port de ta machine et un port du Service dans le cluster.

### `kubectl` et kubeconfig
`kubectl` est le CLI qui communique avec l'API server du cluster. Il sait où se connecter grâce au fichier `~/.kube/config` (le **kubeconfig**), qui contient l'adresse du cluster, les credentials, et le contexte actif. Minikube configure ce fichier automatiquement au démarrage.
