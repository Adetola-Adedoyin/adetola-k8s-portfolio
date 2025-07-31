# Kubernetes Portfolio Project: `adetola-k8s-portfolio`


## 🗂️ Table of Contents

1. [Architecture Diagram](#architecture-diagram)
2. [Phase 1 – Local Kubernetes Cluster Setup](#phase-1)
3. [Phase 2 – GitOps Deployment with ArgoCD](#phase-2)
4. [Phase 3 – Monitoring with Prometheus and Grafana](#phase-3)
5. [Challenges Faced](#challenges-faced)
6. [Future Improvements](#future-improvements)
7. [Appendix](#appendix)

---


## ⚙️ Phase 1: Local Kubernetes Cluster Setup

### Tools:

* Docker
* kind (Kubernetes in Docker)
* kubectl

### Steps:

```bash
# Install Docker
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker

# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create a kind cluster
kind create cluster --name adetola-cluster

# Confirm cluster is running
kubectl get nodes
```

---

## 🚀 Phase 2: GitOps Deployment with ArgoCD

### Tools:

* Helm
* ArgoCD
* GitHub: [adetola-k8s-portfolio](https://github.com/Adetola-Adedoyin/adetola-k8s-portfolio)

### Steps:

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Create Helm chart
helm create adetola-portfolio

# Edit values.yaml to include Docker image hosted on Docker Hub:
# image:
#   repository: adetola/<image-name>
#   tag: latest

# Deploy the Helm chart locally (for testing)
helm install adetola-release ./adetola-portfolio

# Install ArgoCD in 'argocd' namespace
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Patch ArgoCD service to expose it via NodePort
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'

# Get ArgoCD admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Access ArgoCD dashboard at http://<NODE-IP>:<ARGOCD-NODEPORT>

# Log in using the CLI (optional)
argocd login <NODE-IP>:<ARGOCD-NODEPORT> --username admin --password <PASSWORD>

# Connect GitHub repo
argocd repo add https://github.com/Adetola-Adedoyin/adetola-k8s-portfolio.git

# Create Application using CLI or ArgoCD UI and sync
```

---

## 📊 Phase 3: Monitoring with Prometheus & Grafana

### Tools:

* Prometheus Operator (`kube-prometheus-stack`)
* Grafana

### Steps:

```bash
# Add and update Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus + Grafana stack
helm install monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# Get Grafana admin password
kubectl get secret monitor-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 -d && echo

# Port-forward Grafana for access
export POD_NAME=$(kubectl get pods -n monitoring \
  -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitor" -o name)
kubectl port-forward -n monitoring $POD_NAME 3000
```

### Grafana Panel Setup:

* Log into Grafana at [http://localhost:3000](http://localhost:3000).
* Use admin credentials from the previous step.
* Add a new panel and select metrics (e.g., `node_cpu_seconds_total`).
* Customize and save dashboard.

---

## ⚠️ Challenges Faced

* ArgoCD failed syncing due to immutable Deployment label. Fixed by deleting and re-creating the Deployment.
* Grafana initially showed 0 CPU usage — due to local environment and metric selection.
* Issues with port-forwarding and accessing services were resolved by checking pod names carefully and applying correct port-forward commands.

---

## 💡 Future Improvements

* [ ] Move cluster to cloud (e.g., AWS EKS)
* [ ] Add Ingress controller and TLS cert-manager
* [ ] Expand monitoring to application-specific metrics
* [ ] Implement authentication for the portfolio app
* [ ] Add CI/CD pipeline integration (e.g., GitHub Actions)

---

## 📎 Appendix

* Node IP: `172.18.0.2`
* Portfolio App NodePort: `30080`
* ArgoCD Dashboard NodePort: `30970`
* Grafana Dashboard Port: `3000` (via port-forward)
* GitHub Repo: [adetola-k8s-portfolio](https://github.com/Adetola-Adedoyin/adetola-k8s-portfolio)


