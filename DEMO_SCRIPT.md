# Demo Video Script (about 8-10 minutes, screen recording)

Record with OBS, Zoom/Teams "record", or Win+G / QuickTime. Keep a terminal and browser visible.

## 1. Introduction (30 s)
- Your name, the topic, the app: a Flask app with `/` and `/health`.
- Show the GitHub repo and the file list.

## 2. Docker (3 min)
1. Open `Dockerfile`; explain each line briefly (base image, WORKDIR, caching of dependencies, non-root user, CMD).
2. `docker build -t flask-k8s-demo:1.0 .`
3. `docker images`
4. `docker run -d --name flask-container -p 5000:5000 flask-k8s-demo:1.0`
5. `docker ps`, then open `http://localhost:5000` in the browser.
6. `docker logs flask-container`
7. Say: image = read-only template; container = running instance; registry stores images.
8. `docker stop flask-container && docker rm flask-container`

## 3. Kubernetes (4 min)
1. Show `kubectl get nodes` (cluster is running) and load the image into it if needed.
2. Open `deployment.yaml`: replicas, labels/selector, image, probes, resources.
3. Open `service.yaml`: NodePort, selector, port/targetPort.
4. `kubectl apply -f deployment.yaml` and `kubectl apply -f service.yaml`
5. `kubectl get deployments`, `kubectl get pods -o wide`, `kubectl get svc`
6. Access the app (`minikube service flask-service --url` or `kubectl port-forward service/flask-service 8080:80`); refresh or run `curl` several times to show different `pod_hostname` values.
7. Self-healing: `kubectl delete pod <name>` then `kubectl get pods`; a new pod appears.
8. Scaling: `kubectl scale deployment flask-app --replicas=5`, then `kubectl get pods`.

## 4. Architecture explanation (1-2 min)
- Control plane: API server, etcd, scheduler, controller manager.
- Worker node: kubelet, kube-proxy, container runtime.
- Objects: Pod, ReplicaSet, Deployment, Service.
- Walk through what happened when you ran `kubectl apply`.

## 5. Wrap-up (30 s)
- Cloud-native concepts shown: containerization, declarative config, scaling, self-healing, load balancing.
- Cleanup: `kubectl delete -f service.yaml -f deployment.yaml`
