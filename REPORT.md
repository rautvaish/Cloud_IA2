# Technical Report: Containerizing and Deploying a Flask Application with Docker and Kubernetes

**Name:** ______________  **Roll No:** ______________  **Date:** 01 Nov 2026
**GitHub repository:** ______________  **Demo video:** ______________

## 1. Objective

To containerize a simple Python Flask application using Docker, and deploy it on Kubernetes using Deployment and Service configuration files, demonstrating running pods and an accessible service.

## 2. Application

A Flask app exposing two endpoints:
- `GET /`: returns a JSON message, the serving pod's hostname and the app version.
- `GET /health`: returns `{"status": "ok"}`; used for Kubernetes health probes.

It is served with **gunicorn** inside the container instead of Flask's development server.

## 3. Part A: Docker

### 3.1 Dockerfile explained

| Instruction | Purpose |
|---|---|
| `FROM python:3.12-slim` | Small official base image with Python |
| `WORKDIR /app` | Working directory inside the container |
| `COPY requirements.txt` + `RUN pip install` | Dependencies installed before code is copied, so this layer is cached unless dependencies change |
| `COPY app.py .` | Adds the application code |
| `USER appuser` | Runs as a non-root user (security best practice) |
| `EXPOSE 5000` | Documents the listening port |
| `CMD ["gunicorn", ...]` | Default start command |

### 3.2 Commands

```
docker build -t flask-k8s-demo:1.0 .
docker run -d --name flask-container -p 5000:5000 flask-k8s-demo:1.0
```

*(Insert screenshots: build output, `docker images`, `docker ps`, browser at localhost:5000)*

### 3.3 Image vs. container

- **Image:** a read-only, layered template containing the OS libraries, runtime, dependencies and application code. Built from a Dockerfile; each instruction creates a layer. Images are immutable and portable.
- **Container:** a running instance of an image. Docker adds a thin writable layer on top and uses Linux namespaces (isolation) and cgroups (resource limits) to isolate the process. Many containers can run from one image.
- **Registry:** a store for images (e.g., Docker Hub).
- Containers share the host kernel, so they are lighter and faster to start than virtual machines.

## 4. Part B: Kubernetes

### 4.1 deployment.yaml

- `replicas: 3`: Kubernetes keeps 3 pods running at all times.
- `selector` / `labels`: tie the Deployment to its pods via `app: flask-app`.
- `image: flask-k8s-demo:1.0`, `containerPort: 5000`.
- `readinessProbe`: pod receives traffic only when `/health` responds.
- `livenessProbe`: container is restarted if `/health` stops responding.
- `resources`: requests and limits for CPU and memory.

### 4.2 service.yaml

- Type `NodePort`: exposes the app outside the cluster on port 30007 of the node.
- `selector: app: flask-app`: routes to all matching pods and load-balances across them.
- `port: 80` → `targetPort: 5000`.

### 4.3 Commands

```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get deployments,pods,services
```

*(Insert screenshots: `kubectl get pods -o wide`, `kubectl get svc`, app reached via service showing different pod hostnames, scaling, self-healing)*

### 4.4 Kubernetes architecture

**Control plane (master):**
- **kube-apiserver:** front door of the cluster; all `kubectl` commands go through it.
- **etcd:** key-value store holding the cluster's desired and current state.
- **kube-scheduler:** assigns new pods to suitable nodes.
- **kube-controller-manager:** runs controllers (e.g., ReplicaSet controller) that move actual state toward desired state.

**Worker nodes:**
- **kubelet:** agent that ensures the containers of assigned pods are running and healthy.
- **kube-proxy:** maintains network rules so Services route traffic to pods.
- **Container runtime** (containerd): actually runs the containers.

**Key objects:**
- **Pod:** smallest deployable unit; one or more containers sharing network and storage.
- **ReplicaSet:** maintains a set number of identical pods.
- **Deployment:** manages ReplicaSets; provides rolling updates, rollbacks and scaling.
- **Service:** stable IP/DNS name and load balancing for a changing set of pods.

**Flow:** `kubectl apply` → API server stores the desired state in etcd → scheduler picks a node → kubelet starts the containers → kube-proxy exposes them through the Service.

## 5. Cloud-native concepts demonstrated

- **Containerization:** consistent, portable packaging.
- **Declarative configuration:** desired state in YAML, tracked in Git.
- **Scalability:** `kubectl scale deployment flask-app --replicas=5`.
- **Self-healing:** deleted pods are recreated automatically.
- **Load balancing and service discovery:** via the Service.
- **Rolling updates:** zero-downtime version changes.
- **Health checks:** readiness and liveness probes.
- **Resource management:** requests and limits.

## 6. Results

The application was built into the image `flask-k8s-demo:1.0`, run successfully as a Docker container, then deployed on Kubernetes with 3 replicas. The Service exposed the app, and repeated requests showed different pod hostnames, confirming load balancing. *(Update with your actual observations.)*

## 7. Conclusion

Docker packages the application and its dependencies into a portable image; Kubernetes runs it reliably at scale with self-healing, load balancing and rolling updates, all described by simple YAML files.
