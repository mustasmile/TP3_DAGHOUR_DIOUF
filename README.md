# Dojo Kubernetes

L’objectif est de déployer plusieurs applications sur Kubernetes avec **minikube**, et d’apprendre un maximum en documentant chaque étape.

## 0. Préparer l’environnement

### Outils installés
- **minikube** : créer un cluster Kubernetes local  
- **kubectl** : piloter Kubernetes en ligne de commande  
- **Docker** : construire et stocker les images de conteneurs  
- **Helm** : gérer des applications complexes via des charts  
- **k9s** : observer et interagir avec le cluster en TUI  
- **stern** : suivre les logs de plusieurs pods en même temps  
- **kubens** : changer rapidement de namespace  
- **kubectx** : changer rapidement de contexte  

### Vérifications
```bash
minikube version
kubectl version --client
docker --version
helm version
k9s version
```

## 1. Explorer minikube et les bases de Kubernetes

Commandes utilisées :
```bash
kubectl get all 
kubectl get nodes
kubectl get namespaces
kubectl config view
kubectl config current-context
kubectl describe node minikube
kubectl cluster-info
kubectl get events --sort-by='.metadata.creationTimestamp' -A
kubectl get pods --all-namespaces
kubectl get pods -o yaml
kubectl explain pods
```

Apprentissages :
- Découverte des namespaces.  
- Affichage de la configuration du contexte (`minikube`).  
- Utilisation de `kubectl explain` pour comprendre une ressource.  

---

## 2. Déployer un Pod minimal (nginx)

### Fichier `pod.yaml`
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

### Commandes
```bash
minikube kubectl -- apply -f pod.yaml
minikube kubectl -- get pods
minikube kubectl -- logs nginx-pod
minikube kubectl -- port-forward pod/nginx-pod 9090:80
minikube kubectl -- delete -f pod.yaml
```

### Vérifications
- Pod en état `Running`.  
- Logs visibles (`kubectl logs`).  
- Accès à nginx via [http://localhost:9090](http://localhost:9090).  
- Pod supprimé avec succès.  

### Difficultés rencontrées
| Problème | Solution | Nouveau savoir |
|----------|----------|----------------|
| Port 8080 déjà utilisé | Utilisation de `9090:80` avec `port-forward` | On peut mapper sur n’importe quel port local disponible |
| Erreur `NotFound` sur `my-nginx-pod` | Vérifier le vrai nom avec `kubectl get pods` | Le nom du pod doit correspondre à celui du manifest |
| Erreur avec `-f` dans `minikube kubectl` | Utiliser `minikube kubectl -- delete -f pod.yaml` | `--` permet de passer les arguments directement à kubectl |

---

## 3. Déployer nginx (Deployment + Service)

### Fichier `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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

### Fichier `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

### Commandes
```bash
minikube kubectl -- apply -f deployment.yaml
minikube kubectl -- apply -f service.yaml
minikube kubectl -- get deployments
minikube kubectl -- get pods
minikube kubectl -- get svc
minikube service nginx-service
```

## Apprentissages

| Problème | Solution | Nouveau savoir |
|----------|----------|----------------|
| `-f` non reconnu avec minikube | Utiliser `minikube kubectl -- apply -f ...` | Minikube utilise son kubectl intégré |
| Conflit de port avec 8080 | Mapper sur un autre port (`9090:80`) | Kubernetes permet de choisir n’importe quel port local |
| Service non trouvé | Créer `service.yaml` avec un `selector` correct | Le selector doit matcher les labels du Deployment |

---
Nous avons :  
- Préparé l’environnement  
- Exploré Kubernetes avec `kubectl`  
- Déployé un Pod simple (nginx)  
- Transformé en Deployment avec plusieurs replicas  
- Exposé avec un Service et testé avec `minikube service`  
