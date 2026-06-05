# TD Kubernetes - Mini Todo (k3d)

Application "Mini Todo" deployee sur un cluster k3d.
Il y a un backend Node.js (l'API, qui stocke les todos en memoire) et un
frontend nginx en 2 exemplaires qui parle au backend.

## Architecture (en gros)

```
Navigateur (localhost:8080)
   -> Service frontend (LoadBalancer, port 80)
      -> 2 pods nginx (config via la ConfigMap frontend-config)
         -> Service backend (ClusterIP, port 3000)
            -> 1 pod Node.js (token admin via le Secret backend-secret)
```

Le backend garde les todos en memoire : si le pod backend redemarre, on perd
les todos. C'est voulu pour le TD.

## Contenu du depot

```
manifests/
  02-configmap.yaml         ConfigMap frontend-config
  03-deployment-back.yaml   Deployment backend (1 replica)
  04-deployment-front.yaml  Deployment frontend (2 replicas)
  05-service-back.yaml      Service ClusterIP (fourni)
  06-service-front.yaml     Service LoadBalancer (fourni)
screenshots/
  app-v1.png
  app-v2.png
  kubectl-get-all.png
README.md
```

## Les commandes que j'ai faites (dans l'ordre)

### 1. Le cluster

```powershell
k3d cluster create td-k8s `
  --servers 1 --agents 2 `
  --port "8080:80@loadbalancer" `
  --k3s-arg "--disable=traefik@server:0"

kubectl get nodes
```

J'ai aussi importe les images dans le cluster pour eviter les pulls lents :

```powershell
docker pull stephanparichon/epsi-k8s-bff:1.0
docker pull stephanparichon/epsi-k8s-front:1.0
docker pull stephanparichon/epsi-k8s-front:2.0

k3d image import stephanparichon/epsi-k8s-bff:1.0 stephanparichon/epsi-k8s-front:1.0 stephanparichon/epsi-k8s-front:2.0 --cluster td-k8s
```

### 2. Le Secret (en imperatif)

Le token n'est pas dans les fichiers, je le cree en ligne de commande :

```powershell
kubectl create secret generic backend-secret --from-literal=admin-token=s3cr3t-token-td
```

### 3. Appliquer les manifests

Attention a l'ordre : le Secret avant le backend, et le Service backend avant
le frontend (sinon le DNS du backend n'existe pas encore au demarrage des pods).

```powershell
kubectl apply -f manifests/02-configmap.yaml
kubectl apply -f manifests/05-service-back.yaml
kubectl apply -f manifests/03-deployment-back.yaml
kubectl apply -f manifests/06-service-front.yaml
kubectl apply -f manifests/04-deployment-front.yaml
```

### 4. Verifier que tout tourne

```powershell
kubectl get all
```

Tous les pods doivent etre Running et Ready (backend 1/1, frontend 2/2).
Voir la capture `screenshots/kubectl-get-all.png`.

Ensuite j'ouvre http://localhost:8080 et je teste en ajoutant / supprimant des
todos. Capture de l'app en v1 : `screenshots/app-v1.png`.

### 5. Rolling update v1 vers v2

J'ai change l'image dans `04-deployment-front.yaml` (front:1.0 -> front:2.0),
puis :

```powershell
kubectl apply -f manifests/04-deployment-front.yaml
kubectl rollout status deployment/frontend
```

Avec maxUnavailable: 0 et maxSurge: 1, la mise a jour se fait sans coupure.
Apres refresh la page passe en theme violet avec le badge "v2.0 - NEW".
Capture : `screenshots/app-v2.png`.

### 6. Tester le Secret avec curl

Sans le bon token, l'endpoint admin doit refuser (401). Avec le bon token, il
vide les todos (200).

```powershell
# Sans token -> 401
curl.exe -i -X POST http://localhost:8080/api/admin/clear

# Mauvais token -> 401
curl.exe -i -X POST http://localhost:8080/api/admin/clear -H "X-Admin-Token: mauvais"

# Bon token -> 200, les todos sont supprimes
curl.exe -i -X POST http://localhost:8080/api/admin/clear -H "X-Admin-Token: s3cr3t-token-td"
```

Apres le dernier appel, en rafraichissant le navigateur la liste est vide.

## Question : si je supprime un pod frontend a la main, il se passe quoi ?

Kubernetes en recree un tout de suite. Le Deployment frontend demande 2 replicas,
donc il y a un ReplicaSet derriere qui surveille en permanence le nombre de pods.
Des qu'il en manque un, le ReplicaSet en relance un nouveau (avec un autre nom)
pour revenir a 2. C'est le principe de self-healing : on declare l'etat voulu et
le cluster s'arrange pour le maintenir. Comme il reste 2 replicas, le site reste
accessible pendant que le remplacant demarre.

Test :

```powershell
kubectl get pods -l app=frontend
kubectl delete pod <nom-du-pod>
kubectl get pods -l app=frontend
```
