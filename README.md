# KinD Cluster with NGINX Ingress Controller

This project sets up a **Kubernetes cluster using KinD** with:
- 1 Control Plane node
- 3 Worker nodes
- NGINX Ingress Controller
- Sample HTTP server app (Apache) with Ingress routing

---

## 🚀 Prerequisites

- Docker
- [KinD](https://kind.sigs.k8s.io/)
- `kubectl`

---

## ⚙️ Cluster Configuration

### kind-cluster.yaml

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
  - role: worker
  - role: worker
  - role: worker
````

---

## 📦 Cluster Setup

```bash
# Delete existing cluster if any
kind delete cluster --name multi-node-cluster

# Create the cluster
kind create cluster --name multi-node-cluster --config kind-cluster.yaml

# Confirm cluster is ready
kubectl get nodes
```

---

## 🌐 Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/kind/deploy.yaml
```

Wait for the pods to be ready:

```bash
kubectl get pods -n ingress-nginx
```

---

## 🔧 Deploy Sample App

```bash
kubectl create deployment demo --image=httpd --port=80
kubectl expose deployment demo --port=80 --target-port=80 --type=ClusterIP
```

---

## 🚢 Install Argo CD

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
kubectl port-forward svc/argocd-server -n argocd 8090:443
```

Access it at: [https://localhost:8090](https://localhost:8090)

---

### 🔐 Argo CD Login

```bash
# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

Username: `admin`
Password: (from above command)

## ✨ Configure Ingress

Create a file `demo-ingress.yaml`:

```yaml
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

Apply it:

```bash
kubectl apply -f demo-ingress.yaml
```

---

## 🧭 Access the App

### Add host entry:

```bash
sudo echo "127.0.0.1 demo.local" | sudo tee -a /etc/hosts
```

## 🧪 Test URLs

* Argo CD: [https://localhost:30080](https://localhost:30080)
* App via Ingress: [http://demo.local](http://demo.local)

---

## ✅ Cleanup

```bash
kind delete cluster --name multi-node-cluster
```

---

## 🧩 Extras (Optional)

* Add TLS with cert-manager
* Support multiple apps and subdomains

---

## 🛠️ Author

Kamran Javed Ansari
[https://github.com/iemkamran](https://github.com/iemkamran)