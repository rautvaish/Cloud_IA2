# Running the Docker + Kubernetes Project on an AWS EC2 Instance

## 1. Launch the instance

| Setting | Value |
|---|---|
| AMI | Ubuntu Server 24.04 LTS (or 22.04) |
| Instance type | **t3.medium** (2 vCPU, 4 GB). t2.micro/t3.micro (1 GB) is too small for Kubernetes |
| Storage | 20 GB gp3 |
| Key pair | Create/download a `.pem` file |

### Security group (inbound rules)

| Port | Source | Purpose |
|---|---|---|
| 22 | My IP | SSH |
| 5000 | My IP (or 0.0.0.0/0 for demo) | Docker container test |
| 30007 | My IP (or 0.0.0.0/0 for demo) | Kubernetes NodePort Service |
| 8080 | My IP | Only if you use `kubectl port-forward` (Option B) |

Prefer "My IP" over 0.0.0.0/0 so strangers can't reach your demo.

## 2. Connect

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

## 3. Install Docker and get the code

```bash
sudo apt update
sudo apt install -y docker.io git curl
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker          # or log out and back in

git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
```

## 4. Part A: Docker on EC2

```bash
docker build -t flask-k8s-demo:1.0 .
docker images
docker run -d --name flask-container -p 5000:5000 flask-k8s-demo:1.0
docker ps
curl http://localhost:5000/
```

From your laptop's browser: `http://<EC2-PUBLIC-IP>:5000/`

Take screenshots with the EC2 console (instance details/public IP) visible as evidence, then clean up:

```bash
docker stop flask-container && docker rm flask-container
```

## 5. Part B: Kubernetes on EC2

Choose **one** option.

### Option A (recommended): k3s, lightweight single-node Kubernetes

NodePort is reachable directly on the EC2 public IP, so no port-forward is needed.

```bash
# Install k3s
curl -sfL https://get.k3s.io | sh -

# Use kubectl without sudo
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

kubectl get nodes          # should show Ready

# k3s uses containerd, so import the Docker image into it
docker save flask-k8s-demo:1.0 | sudo k3s ctr images import -
sudo k3s ctr images ls | grep flask

# Deploy
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get deployments
kubectl get pods -o wide
kubectl get svc
```

Open in your browser: `http://<EC2-PUBLIC-IP>:30007/`. Refresh several times; `pod_hostname` changes between the 3 pods.

### Option B: minikube (Docker driver)

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

minikube start --driver=docker
minikube image load flask-k8s-demo:1.0

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods,svc
```

With minikube, the NodePort lives on minikube's internal IP, not the EC2 public IP. Use port-forward instead:

```bash
kubectl port-forward --address 0.0.0.0 service/flask-service 8080:80
```

Open `http://<EC2-PUBLIC-IP>:8080/` (port 8080 must be open in the security group). Keep this terminal running, and use a second SSH session for other commands.

## 6. Demonstrate Kubernetes features

```bash
kubectl scale deployment flask-app --replicas=5
kubectl get pods

kubectl delete pod <pod-name>      # self-healing
kubectl get pods -w

# rolling update: edit APP_VERSION in deployment.yaml to "2.0", then
kubectl apply -f deployment.yaml
kubectl rollout status deployment/flask-app

kubectl logs deployment/flask-app
kubectl describe service flask-service
```

## 7. Troubleshooting

| Problem | Fix |
|---|---|
| `permission denied` on docker | Run `newgrp docker` or log out/in after `usermod` |
| Pods show `ErrImagePull` / `ImagePullBackOff` | Image not loaded into the cluster: re-run the `k3s ctr images import` or `minikube image load` step; `imagePullPolicy: IfNotPresent` is already set |
| Browser can't connect | Check security group port (5000 / 30007 / 8080) and use the **public** IP over `http://` (not https) |
| Pods `Pending` | Not enough memory: use t3.medium; check `kubectl describe pod <name>` |
| minikube fails to start | Needs at least 2 CPUs and about 2 GB free; t3.medium is enough |
| Public IP changed | It changes on stop/start unless you attach an Elastic IP |

## 8. Cleanup (avoid charges)

```bash
kubectl delete -f service.yaml -f deployment.yaml
# k3s only: sudo /usr/local/bin/k3s-uninstall.sh
# minikube only: minikube delete
```

Then **stop or terminate** the EC2 instance in the AWS console when you're done recording.

## 9. Extra evidence worth capturing on EC2

- EC2 console showing the running instance, type and public IP
- Security group inbound rules
- Terminal output of `kubectl get nodes`, `get pods -o wide`, `get svc`
- Browser tab showing `http://<EC2-PUBLIC-IP>:30007/` with the URL bar visible
