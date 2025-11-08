# 🚀 Node.js Application Monitoring using Prometheus, Grafana, and Argo CD on GKE

---

## 📘 Overview

This project demonstrates how to **deploy and monitor a Node.js application** on **Google Kubernetes Engine (GKE)** using:

* **Prometheus** and **Grafana** for observability
* **Argo CD** for GitOps-based deployment
* **Helm** for managing Kubernetes charts

The setup enables **real-time metrics visualization** (CPU, memory, and HTTP requests) and **automated GitOps synchronization**.

---

## 🧩 Architecture Diagram

```
Developer → GitHub Repo (YAMLs + Helm)
        ↓
   Argo CD (GitOps)
        ↓
   GKE Cluster
        ↓
   Prometheus + Grafana Stack
        ↓
   Node.js Application Metrics Dashboard
```
![WhatsApp Image 2025-11-08 at 15 14 39_aa5c57f8](https://github.com/user-attachments/assets/7ff06e6a-94f4-4be4-96d9-4ae009952955)

---

## 🧰 Tools & Technologies

| Tool                               | Purpose                                             |
| ---------------------------------- | --------------------------------------------------- |
| **GKE (Google Kubernetes Engine)** | Kubernetes cluster for running workloads            |
| **Helm**                           | Manage Prometheus, Grafana, and Node.js deployments |
| **Argo CD**                        | Continuous Delivery (GitOps model)                  |
| **Prometheus**                     | Metrics collection                                  |
| **Grafana**                        | Visualization of metrics                            |
| **Node.js**                        | Sample web application                              |

---

## ⚙️ Step-by-Step Setup

### 🧱 Step 1 — Create GKE Cluster

```bash
gcloud container clusters create monitoring-cluster \
  --num-nodes=3 \
  --zone=asia-south1-b
```

Get cluster credentials:

```bash
gcloud container clusters get-credentials monitoring-cluster --zone asia-south1-b
```

---

### 📦 Step 2 — Create Namespace for Monitoring

```bash
kubectl create namespace monitoring
```

---

### 📊 Step 3 — Add Helm Repository and Update

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

### 🧠 Step 4 — Install kube-prometheus-stack using Helm

```bash
helm install monitoring-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

Confirm all resources are running:

```bash
kubectl get pods -n monitoring
```

---

### 🖥️ Step 5 — Access Grafana Dashboard

Forward Grafana service:

```bash
kubectl port-forward svc/monitoring-stack-grafana -n monitoring 3000:80
```

Now open in your browser:
👉 **[http://localhost:3000](http://localhost:3000)**

* Username: `admin`
* Password: `prom-operator`

---

### ⚙️ Step 6 — Verify Prometheus is Running

Forward Prometheus service:

```bash
kubectl port-forward svc/monitoring-stack-kube-prom-prometheus -n monitoring 9090:9090
```

Check in browser:
👉 **[http://localhost:9090](http://localhost:9090)**

---

### 🌐 Step 7 — Deploy Node.js Application

**Directory Structure:**

```
Node.js-Application/
├── app/
│   ├── Dockerfile
│   ├── index.js
│   ├── package.json
│   ├── helm/nodejs-chart/
│   └── monitoring/servicemonitor-nodejs.yaml
```

**Apply app manifest:**

```bash
kubectl apply -f app/monitoring/servicemonitor-nodejs.yaml
```

---

### 🔗 Step 8 — Connect Argo CD to GKE

#### Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### Port Forward Argo CD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access at 👉 **[https://localhost:8080](https://localhost:8080)**

---

### 🔐 Step 9 — Login to Argo CD

Get initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Login with:

* Username: `admin`
* Password: *(from above command)*

---

### 🧭 Step 10 — Create Argo CD Application (GUI)

In Argo CD UI → **NEW APP**

| Field                | Value                                                |
| -------------------- | ---------------------------------------------------- |
| **Application Name** | `monitoring-stack`                                   |
| **Project**          | `default`                                            |
| **Repository URL**   | `https://prometheus-community.github.io/helm-charts` |
| **Chart**            | `kube-prometheus-stack`                              |
| **Version**          | `79.1.1`                                             |
| **Namespace**        | `monitoring`                                         |
| **Sync Policy**      | `Automatic`                                          |

Click **Create** ✅
If it says *“spec is different”*, reapply with **Upsert** (Enable `Replace/Upsert` in options).

---

### 📈 Step 11 — Import Grafana Dashboard

1. Go to Grafana → **Dashboards → Import**
2. Enter ID: `1860` (Node Exporter Full)
3. Click **Load**
4. Select Prometheus datasource → **Import**

If “origin not allowed” error occurs, ensure:

* Grafana is accessible via **port-forward**
* Use `localhost:3000` (not external IP)
* Refresh browser after redoing port-forward

---

### 📊 Step 12 — Visualize Metrics

Now open Grafana → Dashboards → **Node Exporter Full**

You’ll see:

* CPU & Memory Usage
* Disk I/O
* Active Processes
* Network Traffic

---

### 📡 Step 13 — Query Application Metrics

In Grafana → **Explore → Prometheus Datasource**

Run this query:

```promql
rate(http_requests_total[1m])
```

You’ll see your Node.js app’s traffic in real-time! 🎯

---

## 🧠 Common Errors & Fixes

| Error                | Cause                         | Fix                                 |
| -------------------- | ----------------------------- | ----------------------------------- |
| `service not found`  | Wrong service name            | Run `kubectl get svc -n monitoring` |
| `origin not allowed` | Invalid Grafana access origin | Use correct localhost port          |
| `spec is different`  | Argo CD already has app       | Use Upsert flag                     |

---

## ✅ Final Verification

| Component   | Status Command                   | Expected            |
| ----------- | -------------------------------- | ------------------- |
| Grafana     | `kubectl get pods -n monitoring` | Running             |
| Prometheus  | `kubectl get svc -n monitoring`  | Port 9090 available |
| Argo CD     | `kubectl get pods -n argocd`     | All pods running    |
| Node.js App | `kubectl get pods -n default`    | Running pod         |
| Dashboard   | Grafana → Dashboards             | Metrics visible     |

---

## 🧾 Summary

✅ **Deployed Node.js app** on GKE
✅ **Configured Prometheus & Grafana** with Helm
✅ **Connected GitOps via Argo CD**
✅ **Imported Grafana dashboard for real-time monitoring**
✅ **Visualized Node.js app metrics using PromQL**

---


*GitHub → Argo CD → GKE → Prometheus/Grafana → User Dashboard*?
It’ll make your GitHub repo look very professional.
