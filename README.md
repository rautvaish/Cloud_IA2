# Flask App: Docker + Kubernetes Deployment

A small Flask REST app, containerized with Docker and deployed on Kubernetes (3 replicas behind a NodePort Service).

## Project structure

```
.
├── app.py             # Flask application
├── requirements.txt   # Python dependencies
├── Dockerfile         # Image definition
├── .dockerignore
├── deployment.yaml    # Kubernetes Deployment (3 replicas, probes, resource limits)
├── service.yaml       # Kubernetes Service (NodePort)
├── REPORT.md          # Short technical report
└── DEMO_SCRIPT.md     # Script for the recorded demo
```

Endpoints: `GET /` (message, pod hostname, version) and `GET /health`.

## Prerequisites

- Docker Desktop (or Docker Engine)
- A local Kubernetes cluster: **minikube**, **kind**, or Docker Desktop's built-in Kubernetes
- `kubectl`

## Part A: Docker

```bash
# 1. Build the image
docker build -t flask-k8s-demo:1.0 .

# 2. List images
docker images

# 3. Run a container (host port 5000 -> container port 5000)
docker run -d --name flask-container -p 5000:5000 flask-k8s-demo:1.0

# 4. Verify
docker ps
curl http://localhost:5000/
curl http://localhost:5000/health
docker logs flask-container

# 5. Stop and remove
docker stop flask-container
docker rm flask-container
```

Optional: push to Docker Hub
```bash
docker tag flask-k8s-demo:1.0 YOUR_DOCKERHUB_USERNAME/flask-k8s-demo:1.0
docker login
docker push YOUR_DOCKERHUB_USERNAME/flask-k8s-demo:1.0
```
(then change `image:` in `deployment.yaml` accordingly)

## Part B: Kubernetes

### Make the image available to the cluster

Pick the one that matches your setup:

```bash
# minikube
minikube start
minikube image load flask-k8s-demo:1.0

# kind
kind create cluster
kind load docker-image flask-k8s-demo:1.0

# Docker Desktop Kubernetes: nothing needed, it shares the local Docker images
```

### Deploy

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

kubectl get deployments
kubectl get pods -o wide
kubectl get services
kubectl describe service flask-service
```

### Access the service

```bash
# minikube
minikube service flask-service --url

# kind / Docker Desktop (port-forward works everywhere)
kubectl port-forward service/flask-service 8080:80
curl http://localhost:8080/
```

Run `curl` several times: the `pod_hostname` field changes as the Service load-balances across the 3 pods.

### Demonstrate Kubernetes features

```bash
# Scale
kubectl scale deployment flask-app --replicas=5
kubectl get pods

# Self-healing: delete a pod and watch it get recreated
kubectl delete pod <pod-name>
kubectl get pods -w

# Rolling update: change APP_VERSION in deployment.yaml, then
kubectl apply -f deployment.yaml
kubectl rollout status deployment/flask-app
kubectl rollout history deployment/flask-app

# Logs
kubectl logs deployment/flask-app
```

### Cleanup

```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
```

## Push to GitHub

```bash
git init
git add .
git commit -m "Flask app with Docker and Kubernetes manifests"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

Add a `screenshots/` folder with your evidence before the final push.

## Evidence checklist

- [ ] `docker build` output
- [ ] `docker images` and `docker ps`
- [ ] Browser/curl showing the app at `localhost:5000`
- [ ] `kubectl get deployments,pods,services`
- [ ] App reachable through the Service (different `pod_hostname` values)
- [ ] Scaling / self-healing screenshot
- [ ] Demo video link
