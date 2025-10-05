# Dojo Kubernetes

Ce projet avait pour but de comprendre le fonctionnement de Kubernetes en partant de la base jusqu’à des concepts plus avancés comme Helm, Redis, les probes et l’autoscaling (HPA).  
On a travaillé sur un cluster local avec **Minikube**.

---

## 0. Préparation de l’environnement

### Outils utilisés
- **Minikube** : création du cluster local  
- **kubectl** : commande principale pour gérer Kubernetes  
- **Docker** : construction et gestion des images  
- **Helm** : déploiement d’applications complexes  
- **k9s** : visualisation du cluster en interface TUI  
- **stern**, **kubens**, **kubectx** : pour les logs et le changement rapide de contextes

### Vérifications
```bash
minikube version
kubectl version --client
docker --version
helm version
```

Tout fonctionnait correctement avant de démarrer.

---

## 1. Premiers pas avec Kubernetes

On a exploré le cluster pour comprendre les ressources de base :

```bash
kubectl get all
kubectl get nodes
kubectl get namespaces
kubectl describe node minikube
kubectl cluster-info
kubectl explain pods
```

Ça nous a permis de comprendre la structure du cluster (Pods, Nodes, Namespaces, Services…).  
On a aussi appris que `kubectl explain` est très utile pour comprendre les manifests YAML.

---

## 2. Premier déploiement : un Pod Nginx

On a commencé simple avec un fichier `pod.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

Commandes utilisées :
```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl logs nginx-pod
kubectl port-forward pod/nginx-pod 9090:80
```

Résultat : le Pod tournait, et on pouvait accéder à nginx sur [http://localhost:9090](http://localhost:9090).

### Difficultés
| Problème | Solution | Ce qu’on a appris |
|-----------|-----------|------------------|
| Port 8080 déjà pris | Utiliser `9090:80` | Le port local peut être changé |
| Nom du pod incorrect | Vérifier avec `kubectl get pods` | Toujours confirmer le nom exact du pod |
| Mauvaise commande minikube | Ajouter `--` avant `-f` | `minikube kubectl -- apply -f` passe les arguments à kubectl |

---

## 3. Déploiement d’un Nginx complet (Deployment + Service)

On est passé à un **Deployment** pour gérer plusieurs replicas et un **Service** pour exposer l’application.

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

Commandes :
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
minikube service nginx-service
```

Et on a pu accéder au service dans le navigateur.

---

## 4. Guestbook avec Helm

On a utilisé **Helm** pour déployer le Guestbook, une vraie application web connectée à Redis.

```bash
helm upgrade --install guestbook ./guestbook
```

Problèmes rencontrés :
- `apiVersion not set` → erreur YAML.  
- Mauvais répertoire de commande → Helm doit être lancé dans le dossier contenant le chart.  
- Utilisation de `{{ .Release.Name }}` et `{{ .Chart.Name }}` pour personnaliser les ressources.

Accès à l’application :  
```bash
minikube service guestbook-service
```

---

## 5. Ajout de Redis

Guestbook utilise Redis comme base de données. On a déployé Redis via Helm :

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis-db bitnami/redis --set auth.enabled=false
```

Mais on a rencontré **beaucoup de problèmes** :  
les images Bitnami ne sont plus disponibles gratuitement depuis 2025 → `ErrImagePull`.

Solution :  
```bash
helm install redis-db oci://registry-1.docker.io/bitnamicharts/redis --version 18.6.1 --set auth.enabled=false
```

Ensuite, on a connecté Guestbook à Redis :

```yaml
env:
  - name: REDIS_HOST
    value: "redis-db"
```

Vérification :
```bash
kubectl exec -it svc/redis-db -- redis-cli ping
PONG
```

Redis fonctionnait bien.

---

## 6. Rendre l’application robuste (Probes)

On a ajouté des probes (`liveness`, `readiness`, `startup`) pour que Kubernetes surveille la santé de l’app.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: {{ .Values.service.port }}
  initialDelaySeconds: 10

readinessProbe:
  httpGet:
    path: /healthz
    port: {{ .Values.service.port }}
  initialDelaySeconds: 5

startupProbe:
  httpGet:
    path: /healthz
    port: {{ .Values.service.port }}
  failureThreshold: 10
```

Quand Redis était coupé, les pods Guestbook passaient en “Not Ready”.  
Quand on relançait Redis, tout redevenait stable automatiquement.

---

## 7. Autoscaling (HPA)

On a ajouté le **Horizontal Pod Autoscaler** pour que Guestbook scale tout seul selon la charge CPU.

Activation du metrics server :
```bash
minikube addons enable metrics-server
```

Création du HPA :
```bash
kubectl autoscale deployment guestbook --cpu-percent=50 --min=3 --max=10
```

Vérification :
```bash
kubectl get hpa
kubectl top pods
```

Pour tester, on a utilisé **k6** :

```bash
sudo snap install k6
```

Script `load-test.js` :
```js
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 100 },
    { duration: '3m', target: 100 },
    { duration: '1m', target: 0 },
  ],
};

export default function () {
  let res = http.get('http://localhost:3000');
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

Pendant le test de charge, le HPA augmentait le nombre de pods.  
Quand la charge redescendait, les pods étaient automatiquement supprimés.

---

## 8. Difficultés principales

| Problème | Cause | Solution |
|-----------|--------|----------|
| Erreurs `ErrImagePull` sur Redis | Images Bitnami limitées | Utilisation du chart OCI |
| Guestbook ne trouvait pas Redis | Mauvaise variable d’environnement | Ajout de `REDIS_HOST` |
| Probes trop rapides | Delays trop courts | Ajustement des temps d’attente |
| Metrics indisponibles | metrics-server non prêt | Attente quelques minutes |
| HPA sans effet | Pas de `requests` CPU définies | Ajout de `resources.requests` dans le manifest |

---

## 9. Conclusion

On a appris énormément de choses :  
- Créer et observer un cluster Kubernetes.  
- Déployer une application réelle avec Helm.  
- Connecter et surveiller une base de données Redis.  
- Gérer la santé de l’app avec des probes.  
- Automatiser la montée en charge avec le HPA.

C’était un projet complet, avec beaucoup d’erreurs et de débogage, mais au final on comprend bien comment Kubernetes gère la vie d’une application du déploiement jusqu’à l’autoscaling.