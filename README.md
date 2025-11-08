# 🚀 Node.js Application Deployment on GKE with CI/CD, Monitoring & Ingress

## 🧩 Project Overview

This project demonstrates a **complete DevOps workflow** — deploying a containerized **Node.js application** on **Google Kubernetes Engine (GKE)** with:

* **Cloud Build** for CI/CD automation
* **Artifact Registry** for image storage
* **Helm** for packaging Kubernetes manifests
* **Argo CD** for GitOps-based deployment
* **Prometheus + Grafana** for monitoring
* **Ingress Controller** for external access

---

## ☁️ Architecture Diagram

```
Developer → GitHub → Cloud Build → Artifact Registry → Argo CD → GKE Cluster
                                        ↑                      ↓
                                Prometheus & Grafana <—— Node.js App
                                               ↑
                                          Ingress Access
```
![WhatsApp Image 2025-11-08 at 15 34 34_cc68e5c7](https://github.com/user-attachments/assets/deee466e-02b9-43fd-b8c0-374cf6e16af2)

---


### **STEP 1 — Create GCP Project**

1. Go to the **Google Cloud Console** → Create a new project.
2. Enable these APIs:

   * Kubernetes Engine API
   * Artifact Registry API
   * Cloud Build API

---

### **STEP 2 — Create Artifact Registry**

```bash
gcloud artifacts repositories create hello-world-repo \
  --repository-format=docker \
  --location=asia-south1 \
  --description="Docker repo for Node.js app"
```
![WhatsApp Image 2025-11-04 at 14 59 32_4a2ef9ed](https://github.com/user-attachments/assets/34d44f2b-88db-4cdd-be01-eda4b5ce183d)

---

### **STEP 3 — Create and Prepare GKE Cluster**

```bash
gcloud container clusters create nodejs-cluster \
  --zone=asia-south1-a \
  --num-nodes=2
```

Then connect to the cluster:

```bash
gcloud container clusters get-credentials nodejs-cluster --zone asia-south1-a
```
![WhatsApp Image 2025-11-04 at 15 00 32_106fb7e2](https://github.com/user-attachments/assets/3086c47c-f09a-477b-bb01-35f3dd16539a)
![WhatsApp Image 2025-11-04 at 15 01 28_b7bfbc37](https://github.com/user-attachments/assets/7bb06788-414f-46c9-a62f-116f930efcac)
![WhatsApp Image 2025-11-04 at 15 02 08_5311e246](https://github.com/user-attachments/assets/e5d60862-dc8f-4add-88d7-d151a64c34d8)
![WhatsApp Image 2025-11-04 at 15 03 11_885fd520](https://github.com/user-attachments/assets/35a31509-0071-413c-a9e6-98cbe9a91928)

---

### **STEP 4 — Create the Node.js App**

Project structure:

```
Node.js-Application/
└── app/
    ├── Dockerfile
    ├── package.json
    └── index.js
```

**index.js**

```js
const express = require('express');
const app = express();
const port = 3000;
app.get('/', (req, res) => res.send('Hello from Node.js on GKE!'));
app.listen(port, () => console.log(`App running on port ${port}`));
```

**package.json**

```json
{
  "name": "nodejs-app",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": { "express": "^4.18.2" }
}
```

**Dockerfile**

```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

---

### **STEP 5 — Create Cloud Build Pipeline**

**cloudbuild.yaml**

```yaml
steps:
  # Step 1: Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      [
        'build', '-t', 'asia-south1-docker.pkg.dev/$PROJECT_ID/hello-world-repo/nodejs-app:$COMMIT_SHA',
        '.'
      ]

  # Step 2: Push Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      [
        'push', 'asia-south1-docker.pkg.dev/$PROJECT_ID/hello-world-repo/nodejs-app:$COMMIT_SHA'
      ]

images:
  - 'asia-south1-docker.pkg.dev/$PROJECT_ID/hello-world-repo/nodejs-app:$COMMIT_SHA'
```

Trigger this in **Cloud Build → Triggers** to build on every Git commit.
![WhatsApp Image 2025-11-04 at 14 58 13_efae9bd9](https://github.com/user-attachments/assets/a1332a97-a9f9-4899-9139-26d340c8a08a)
![WhatsApp Image 2025-11-04 at 14 58 51_c51816f7](https://github.com/user-attachments/assets/3cba9813-3ee8-4fea-b86a-3680dc576d9e)



---

### **STEP 6 — Helm Chart for Deployment**

Create:

```
Node.js-Application/app/helm/nodejs-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
```

**Chart.yaml**

```yaml
apiVersion: v2
name: nodejs-chart
version: 0.1.0
```

**values.yaml**

```yaml
replicaCount: 2
image:
  repository: asia-south1-docker.pkg.dev/YOUR_PROJECT_ID/hello-world-repo/nodejs-app
  tag: latest
service:
  type: ClusterIP
  port: 3000
```

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
        - name: nodejs-app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 3000
```

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-service
spec:
  selector:
    app: nodejs-app
  ports:
    - port: 3000
      targetPort: 3000
  type: ClusterIP
```

![WhatsApp Image 2025-11-05 at 12 54 17_5ee0027c](https://github.com/user-attachments/assets/10c75688-e984-4a48-930a-d5e75ef40e20)
![WhatsApp Image 2025-11-05 at 12 52 32_1f8bab2c](https://github.com/user-attachments/assets/c25e7503-ddb6-4598-89a7-37a4f54d2635)

---

### **STEP 7 — Set Up Ingress Controller**

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-controller ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

**Create Ingress Resource**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nodejs-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nodejs-service
                port:
                  number: 3000
```
![WhatsApp Image 2025-11-05 at 12 56 03_656f21d0](https://github.com/user-attachments/assets/7da1d7b4-9f03-4972-bb11-192ddfd1d307)
![WhatsApp Image 2025-11-05 at 12 57 44_8edd42e2](https://github.com/user-attachments/assets/7b741b0a-aefe-4782-9a3e-62805b29dea6)

---

### **STEP 8 — Deploy Using Argo CD**

Install Argo CD:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Port-forward Argo CD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Login at: [https://localhost:8080](https://localhost:8080)
![WhatsApp Image 2025-11-04 at 15 42 27_c84f89e1](https://github.com/user-attachments/assets/e220d945-f2d9-406c-8418-c79c3af9a697)
![WhatsApp Image 2025-11-04 at 15 43 02_1ecad9b3](https://github.com/user-attachments/assets/b48959f3-1b62-43fc-8976-bda4cc3c8fe7)

---

### **STEP 9 — Create Argo CD Application**

In Argo CD GUI → **NEW APP**

| Field            | Value                                                            |
| ---------------- | ---------------------------------------------------------------- |
| Application Name | nodejs-app                                                       |
| Project          | default                                                          |
| Repository URL   | your GitHub repo                                                 |
| Path             | app/helm/nodejs-chart                                            |
| Cluster URL      | [https://kubernetes.default.svc](https://kubernetes.default.svc) |
| Namespace        | default                                                          |
| Sync Policy      | Automatic                                                        |

Click **Create** → **Sync** → ✅ **Deployed**

![WhatsApp Image 2025-11-05 at 12 40 01_42b7cc36](https://github.com/user-attachments/assets/b602119c-7792-473d-8372-f9fd3462f5e4)
![WhatsApp Image 2025-11-05 at 12 08 15_4b188a02](https://github.com/user-attachments/assets/5f881dd5-4370-4a3c-8b27-f8accea729aa)

---

### **STEP 10 — Verify Deployment**

```bash
kubectl get all
kubectl get ingress
```

Visit the external IP from ingress.
![WhatsApp Image 2025-11-05 at 12 08 51_1bad195e](https://github.com/user-attachments/assets/b866d200-3aad-47e6-aec4-15c0b8322dd0)
![WhatsApp Image 2025-11-05 at 12 09 22_d22c4dff](https://github.com/user-attachments/assets/f09649ef-6136-4c51-ae3a-3655600a3a0c)

---

### **STEP 11 — Add Monitoring Stack (Prometheus + Grafana)**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```
![WhatsApp Image 2025-11-05 at 14 03 06_ff618f8c](https://github.com/user-attachments/assets/966a0e09-4a48-4f6b-9fd4-efd5a9b63f1b)
![WhatsApp Image 2025-11-05 at 14 01 49_8caa692e](https://github.com/user-attachments/assets/a8c8f436-da16-410e-8668-4a4754043836)
![WhatsApp Image 2025-11-05 at 16 04 32_97de80ff](https://github.com/user-attachments/assets/66bb5fa4-f605-4c07-89ea-61d6571765b7)

---

### **STEP 12 — Access Grafana Dashboard**

```bash
kubectl port-forward svc/monitoring-stack-grafana -n monitoring 3000:80
```

Open [http://localhost:3000](http://localhost:3000)
Login → `admin / prom-operator`

---

### **STEP 13 — Add Prometheus as Data Source**

1. In Grafana → Connections → Data Sources → Add Prometheus.
2. URL: `http://monitoring-stack-kube-prometheus-sta-prometheus.monitoring.svc.cluster.local:9090`
3. Save & Test ✅

---

### **STEP 14 — Import Node Exporter Dashboard**

Go to Dashboards → Import → enter ID **1860** → Import (Overwrite).
Dashboard: **Node Exporter Full** now available.

---

### **STEP 15 — Add Node.js Service Monitor**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nodejs-servicemonitor
  labels:
    release: monitoring-stack
spec:
  selector:
    matchLabels:
      app: nodejs-app
  endpoints:
    - port: 3000
      interval: 30s
```

Apply:

```bash
kubectl apply -f servicemonitor-nodejs.yaml -n monitoring
```
![WhatsApp Image 2025-11-05 at 16 17 11_c5c8278e](https://github.com/user-attachments/assets/1a9b615c-8d48-4ddc-ab96-3cee0a93ad76)

---

### **STEP 16 — Validate Prometheus Targets**

Open Prometheus UI:

```bash
kubectl port-forward svc/monitoring-stack-kube-prometheus-sta-prometheus -n monitoring 9090:9090
```

Then visit [http://localhost:9090/targets](http://localhost:9090/targets).

---

### **STEP 17 — Grafana Metrics Visualization**

Use query:

```
rate(http_requests_total[1m])
```

You’ll see live Node.js traffic metrics 🎯
![WhatsApp Image 2025-11-05 at 16 47 30_cefdd7c1](https://github.com/user-attachments/assets/53515b11-7562-4034-96e7-8b8473337393)

---

### **STEP 18 — Argo CD + Monitoring Integration**

Add `argocd-servicemonitor.yaml` to monitor Argo CD performance (optional).
![WhatsApp Image 2025-11-05 at 16 17 11_ae12b3eb](https://github.com/user-attachments/assets/a5498a40-f170-40c2-869e-5770c1b62ffa)

---

### **STEP 19 — Test End-to-End CI/CD**

Push a new commit → Cloud Build triggers → builds Docker image → pushes to Artifact Registry → Argo CD auto-syncs → app auto-updated in GKE ✅

---

### **STEP 20 — Clean Up**

```bash
gcloud container clusters delete nodejs-cluster --zone asia-south1-a
```

## 🔗 Technologies Used

* Google Cloud Build
* Google Artifact Registry
* Google Kubernetes Engine
* Argo CD
* Helm
* Prometheus
* Grafana
* Ingress Controller
* Node.js

---

Would you like me to include a **diagram (PNG)** showing the CI/CD + monitoring flow (Cloud Build → Argo CD → GKE → Grafana)?
I can generate it to include at the top of your README.
