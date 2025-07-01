# KinD Cluster with NGINX Ingress, Argo CD GitOps & Monitoring Stack

This guide sets up a complete **KinD-based Kubernetes lab** with:
- ✅ 1 Control Plane + 3 Worker Nodes
- ✅ NGINX Ingress Controller
- ✅ Argo CD for GitOps-based app deployment
- ✅ Sample App with Ingress (`httpd` via `demo.local`)
- ✅ **Monitoring Stack** (Prometheus, Grafana, Node Exporter, Kube State Metrics)

---

## 🚀 Prerequisites

- Docker
- KinD
- `kubectl`
- `git` (optional: GitHub for your app repo)

---

## ⚙️ Step 1: Create KinD Cluster

Create a file `kind-cluster.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
      - containerPort: 443
        hostPort: 443
      - containerPort: 30080
        hostPort: 30080  # Argo CD UI
  - role: worker
  - role: worker
  - role: worker
```

Create the cluster:

```bash
kind create cluster --name multi-node-cluster --config kind-cluster.yaml
kubectl get nodes
```

---

## 🌐 Step 2: Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

---

## 🚢 Step 3: Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for Argo CD to be ready:

```bash
kubectl get pods -n argocd
```

---

### 🧷 Optional: Port-forward Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 30080:443
```

Access it at: [https://localhost:30080](https://localhost:30080)

---

### 🔐 Argo CD Login

```bash
# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

Username: `admin`  
Password: (from above command)

---

## 🚀 Step 4: Deploy Sample App via Argo CD

Create the app:

```yaml
# app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/iemkamran/gitops-app
    path: ./ # or path to your Helm/Kustomize folder
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply:

```bash
kubectl apply -f app.yaml
```

---

## 🌍 Step 5: Ingress to Sample App

```yaml
# demo-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo
                port:
                  number: 80
```

Apply and update hosts:

```bash
kubectl apply -f demo-ingress.yaml
echo "127.0.0.1 demo.local" | sudo tee -a /etc/hosts
```

---

## 📈 Step 6: Deploy Monitoring Stack (Prometheus + Grafana)

This monitoring stack is from:  
🔗 [iemkamran/monitoring-stack-k8s](https://github.com/iemkamran/monitoring-stack-k8s)

### 📦 Features
- Prometheus (metrics collection)
- Grafana (dashboard visualization)
- Node Exporter
- Kube State Metrics
- Pre-provisioned dashboards

### 📥 Clone the monitoring repo

```bash
git clone https://github.com/iemkamran/monitoring-stack-k8s.git
cd monitoring-stack-k8s
```

### 🚀 Deploy via Helm

```bash
helm upgrade --install monitoring ./prometheus-helm
```

Wait for all pods:

```bash
kubectl get pods -n monitoring
```

### 🌐 Access Grafana

```bash
kubectl port-forward -n monitoring svc/grafana 3001:80
```

Then open: [http://localhost:3001](http://localhost:3001)  
Login: `admin` / `admin`

> Dashboards like Node Exporter, K8s Cluster, Prometheus Overview are automatically provisioned.

---

## 🧪 Test URLs

* Argo CD: [https://localhost:30080](https://localhost:30080)
* App via Ingress: [http://demo.local](http://demo.local)
* Grafana: [http://localhost:3001](http://localhost:3001)

---

## 🧹 Cleanup

```bash
kind delete cluster --name multi-node-cluster
```

---

## 🛠️ Author

Kamran Javed Ansari  
[https://github.com/iemkamran](https://github.com/iemkamran)