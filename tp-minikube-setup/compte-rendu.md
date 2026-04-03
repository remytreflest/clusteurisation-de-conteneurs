# TP — Setup Minikube — Compte-rendu

## Réponses aux questions

### Section 2.2 — Contexte kubectl

1. **Nom du contexte courant** : `minikube` (nom par défaut créé automatiquement par `minikube start`)
2. **Nombre de nœuds** : 1 nœud (`minikube`), de rôle `control-plane` et `master`

---

### Section 3.3 — Inspecter le pod

1. **Image exacte utilisée** : `nginx:1.27` (ou `docker.io/library/nginx:1.27` selon le registre résolu)
2. **Évènements confirmant le démarrage** :
   - `Pulling image "nginx:1.27"` — téléchargement de l'image
   - `Successfully pulled image "nginx:1.27"` — image prête
   - `Started container nginx` — conteneur démarré

---

### Section 4.2 — Accéder au service

1. **À quoi sert un Service dans Kubernetes ?**
   Un Service est une abstraction réseau stable qui expose un ou plusieurs pods. Il fournit une IP virtuelle (ClusterIP) et un DNS interne constants, même si les pods sous-jacents sont recréés ou remplacés. Il assure aussi la répartition de charge entre les replicas.

2. **Pourquoi le Service n'expose-t-il pas automatiquement une IP publique en local ?**
   Un Service de type `ClusterIP` est interne au cluster : son IP n'est routable que depuis l'intérieur. Minikube tourne dans une VM ou un conteneur Docker isolé du réseau hôte ; `minikube service --url` crée un tunnel pour contourner cette isolation.

---

### Section 5.2 — Rollout / mise à jour d'image

1. **Que se passe-t-il au niveau des pods lors d'un changement d'image ?**
   Kubernetes applique une stratégie `RollingUpdate` par défaut : il crée un nouveau pod avec la nouvelle image, attend qu'il soit `Running`, puis supprime l'ancien. À aucun moment le service n'est totalement indisponible.

2. **Qu'est-ce qu'un "rollout" et pourquoi est-ce utile ?**
   Un rollout est le processus de déploiement progressif d'une nouvelle version d'un Deployment. C'est utile car cela permet de mettre à jour sans interruption de service (`zero downtime`), et d'effectuer un retour arrière (`kubectl rollout undo`) en cas de problème.

---

## Mini compte-rendu

**Commandes de vérification utilisées :**

```bash
kubectl get nodes                          # vérifier le nœud minikube
kubectl get pods -n tp-minikube            # état des pods
kubectl get svc -n tp-minikube             # état du service
kubectl describe pod -n tp-minikube -l app=web-nginx  # détails du pod
kubectl rollout status -n tp-minikube deployment/web-nginx
```

**Exposition du service :**
Le service est de type `ClusterIP` (interne au cluster). Pour y accéder depuis le navigateur, j'ai utilisé :
```bash
minikube service web-nginx-svc -n tp-minikube --url
```
Cette commande crée un tunnel entre la machine hôte et le cluster Minikube, et retourne une URL `http://127.0.0.1:<port>` accessible localement.

**Difficultés rencontrées :**

1. **Minikube qui ne démarre pas avec le driver Docker** : Docker Desktop n'était pas démarré. Solution : lancer Docker Desktop avant `minikube start --driver=docker`.

2. **Labels incohérents entre selector et template** : Le pod n'était pas sélectionné par le Deployment car le label `app` dans `spec.selector.matchLabels` ne correspondait pas à celui de `spec.template.metadata.labels`. Solution : s'assurer que les trois niveaux de labels (`metadata.labels`, `selector.matchLabels`, `template.metadata.labels`) sont identiques.
