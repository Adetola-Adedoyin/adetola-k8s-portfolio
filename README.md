# Kubernetes Portfolio Project

A comprehensive guide to deploying a portfolio application on Kubernetes using Kind, Helm, ArgoCD, Prometheus, and Grafana for a complete GitOps and monitoring setup.

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Phase 1: Local Kubernetes Setup with Kind](#phase-1-local-kubernetes-setup-with-kind)
- [Phase 2: Application Deployment](#phase-2-application-deployment)
- [Phase 3: GitOps with Helm and ArgoCD](#phase-3-gitops-with-helm-and-argocd)
- [Phase 4: Monitoring with Prometheus and Grafana](#phase-4-monitoring-with-prometheus-and-grafana)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Ubuntu/Linux system
- Basic knowledge of Kubernetes, Docker, and YAML
- GitHub account for GitOps workflow

## Phase 1: Local Kubernetes Setup with Kind

### Step 1: Install Docker

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker

# Add user to docker group to run without sudo
sudo usermod -aG docker $USER
newgrp docker
```

### Step 2: Install Kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Step 3: Create Local Cluster

```bash
kind create cluster
```

Expected output:
```
Creating cluster "kind" ...
✓ Ensuring node image ...
✓ Preparing nodes ...
✓ Writing configuration ...
✓ Starting control-plane ...
✓ Installing CNI ...
✓ Installing StorageClass ...
Set kubectl context to "kind-kind"
```

### Step 4: Verify Cluster

```bash
kubectl config current-context
kubectl get nodes
```

Expected output:
```
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   Xs    v1.XX.X
```

## Phase 2: Application Deployment

### Step 1: Create Deployment Configuration

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adetola-portfolio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: adetola-portfolio
  template:
    metadata:
      labels:
        app: adetola-portfolio
    spec:
      containers:
      - name: adetola-container
        image: teeboss/adetola-portfolio:v6
        ports:
        - containerPort: 80
```

### Step 2: Create Service Configuration

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: adetola-service
spec:
  selector:
    app: adetola-portfolio
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

### Step 3: Deploy Application

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Check status
kubectl get pods
kubectl get svc
```

### Step 4: Access Application

Since Kind runs in Docker, use port forwarding:

```bash
kubectl port-forward service/adetola-service 8080:80
```

Then visit: http://localhost:8080

## Phase 3: GitOps with Helm and ArgoCD

### Step 1: Install Helm

```bash
sudo apt update
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version
```

### Step 2: Create Helm Chart

```bash
helm create adetola-portfolio
```

This creates the following structure:
```
adetola-portfolio/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ...
```

### Step 3: Configure Helm Chart

Edit `values.yaml`:

```yaml
image:
  repository: teeboss/adetola-portfolio
  pullPolicy: IfNotPresent
  tag: v6

service:
  type: NodePort
  port: 80
  nodePort: 30080
```

### Step 4: Deploy with Helm

```bash
helm install adetola-release ./adetola-portfolio

# Verify deployment
kubectl get pods
kubectl get svc
```

### Step 5: Install ArgoCD

```bash
# Create namespace and install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 6: Access ArgoCD UI

#### Option 1: Port Forward (Recommended for local)
```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```
Visit: https://localhost:8081

#### Option 2: NodePort
```bash
kubectl patch svc argocd-server -n argocd -p '{
  "spec": {
    "type": "NodePort"
  }
}'

kubectl get svc argocd-server -n argocd
```

### Step 7: Get ArgoCD Credentials

```bash
# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode
```

- Username: `admin`
- Password: (from command above)

### Step 8: Setup GitOps Repository

1. Create a GitHub repository (e.g., `adetola-k8s-portfolio`)
2. Push your Helm chart to the repository:

```
adetola-k8s-portfolio/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

### Step 9: Create ArgoCD Application

In the ArgoCD UI:

1. Click **+ NEW APP**
2. Fill in the form:

| Field | Value |
|-------|-------|
| Application Name | adetola-portfolio |
| Project | default |
| Sync Policy | Manual |
| Repository URL | https://github.com/YOUR-USERNAME/adetola-k8s-portfolio.git |
| Revision | HEAD |
| Path | . (or folder containing Chart.yaml) |
| Cluster URL | https://kubernetes.default.svc |
| Namespace | default |

3. Click **Create**
4. Click **SYNC** to deploy

### Step 10: Access Application via Port Forward

```bash
kubectl port-forward service/adetola-release-adetola-portfolio 8081:80
```

Visit: http://localhost:8081

## Phase 4: Monitoring with Prometheus and Grafana

### Step 1: Install Prometheus and Grafana

```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install monitor prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

### Step 2: Verify Installation

```bash
kubectl get pods -n monitoring
```

You should see pods for:
- Prometheus
- Grafana
- Kube-state-metrics
- Node-exporter

### Step 3: Access Grafana

```bash
# Get Grafana admin password
kubectl --namespace monitoring get secrets monitor-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Port forward Grafana
export POD_NAME=$(kubectl --namespace monitoring get pod -l \
  "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitor" -o name)
kubectl --namespace monitoring port-forward $POD_NAME 3000
```

Visit: http://localhost:3000

- Username: `admin`
- Password: (from command above)

### Step 4: Explore Dashboards

In Grafana, navigate to **Dashboards → Browse** to explore:
- Kubernetes / Nodes
- Kubernetes / Workloads
- Pod metrics
- Memory/CPU usage
- Deployment status

## Troubleshooting

### Common Issues and Solutions

1. **Pod not starting**: Check logs with `kubectl logs <pod-name>`
2. **Service not accessible**: Verify port forwarding and service configuration
3. **ArgoCD sync issues**: Check repository URL and path configuration
4. **Grafana dashboard empty**: Ensure Prometheus is scraping metrics correctly

### Useful Commands

```bash
# Check cluster status
kubectl cluster-info

# Watch pod status in real-time
kubectl get pods -w

# Check service endpoints
kubectl get endpoints

# View application logs
kubectl logs -f deployment/adetola-portfolio

# Delete resources
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml

# Uninstall Helm release
helm uninstall adetola-release
```

## Project Architecture

This project demonstrates:

1. **Containerization**: Application containerized with Docker
2. **Orchestration**: Deployed on Kubernetes using Kind
3. **Package Management**: Helm charts for templating and versioning
4. **GitOps**: ArgoCD for continuous deployment from Git repository
5. **Monitoring**: Prometheus for metrics collection and Grafana for visualization
6. **Local Development**: Complete setup running locally for development and testing

## Next Steps

1. Set up automatic sync in ArgoCD for true GitOps workflow
2. Configure alerting rules in Prometheus
3. Add custom Grafana dashboards for application-specific metrics
4. Implement CI/CD pipeline for image building and chart updates
5. Add ingress controller for external access
6. Configure persistent storage for stateful applications

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Update documentation
5. Submit a pull request

## License

This project is open source and available under the [MIT License](LICENSE).
