
# Project 3: Observability Hub

This project implements a complete **Observability Stack** for the Kubernetes (EKS) cluster created in **Project 1**.  
It leverages the **kube-prometheus-stack** Helm chart to deploy **Prometheus**, **Grafana**, and **Alertmanager**, showcasing **SRE principles** like monitoring, visualization and alerting.

To ensure reliability and flexibility, the **Alertmanager configuration** (with sensitive Slack webhook URLs) is managed via a **separate Kubernetes Secret**, instead of embedding it directly in Helm values.  


---

##  Key Components

| Component | Purpose |
|------------|----------|
| **Prometheus** | Collects time-series metrics from nodes, pods, and workloads. |
| **Grafana** | Provides visual dashboards for cluster performance and alert visibility. |
| **Alertmanager** | Handles alert routing and sends notifications (e.g., Slack). |
| **PrometheusRule** | Defines alert conditions like "Instance Down" or "High CPU Usage". |

---

##  Architecture Overview

This deployment follows a **two-stage architecture**:

1. **Helm Install:** Deploys Prometheus, Grafana, and core services via `prometheus-values.yml`.  
2. **Manual Secret Injection:** Creates a secure Kubernetes Secret (`alertmanager-config.yml`) for Slack webhook configuration.  
3. **Pod Restart:** Restarts the Alertmanager pod to reload the configuration dynamically.

---

##  Project Structure

```bash
/3-Observability-Hub
├── README.md                 #  guide
├── prometheus-values.yaml    # Helm configuration (Prometheus + Grafana)
├── alertmanager-config.yaml  # External Alertmanager config for Slack
├── alerting-rules.yaml       # Custom SRE alert rules
└── test-alert.yaml           # Sample test alert to verify Slack integration
````

---

##  Prerequisites

1. **EKS Cluster** from Project 1 (with EBS CSI Driver enabled).
2. **kubectl** configured for your cluster.

   ```bash
   kubectl get nodes
   ```
3. **Helm v3** installed locally.

   ```bash
   helm version
   ```

---

##  Deployment Steps (Production-Ready)

### **Step 1: Add Helm Repository**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

### **Step 2: Create the Monitoring Namespace**

```bash
kubectl create namespace monitoring
```

---

### **Step 3: Install the Stack Using Helm**

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f prometheus-values.yml
```

At this stage, **Alertmanager will deploy** but without its Slack configuration.

---

### **Step 4: Inject the Alertmanager Configuration**

Create the Slack-enabled configuration as a Kubernetes Secret:

```bash
kubectl create secret generic alertmanager-prometheus-kube-prometheus-alertmanager \
  --from-file=alertmanager.yml=alertmanager-config.yml \
  -n monitoring \
  --dry-run=client -o yml | kubectl apply -f -
```

>  The `--from-file` flag renames `alertmanager-config.yaml` to `alertmanager.yaml` inside the secret, matching what the operator expects.

---

### **Step 5: Restart Alertmanager Pod**

```bash
kubectl delete pod -n monitoring -l app.kubernetes.io/name=alertmanager
```

This ensures the Alertmanager reloads the updated configuration automatically.

---

### **Step 6: Verify Deployment**

```bash
kubectl get pods -n monitoring -w
```

Wait until all pods show `Running` — especially:

```
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2   Running
```

---

##  Verification Steps

### 1. Apply SRE Alert Rules

```bash
kubectl apply -f alerting-rules.yaml -n monitoring
```

### 2. Trigger a Test Alert**

```bash
kubectl apply -f test-alert.yml -n monitoring
```

---

## Accessing the UIs

### **Prometheus**

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

Then open:
 [http://localhost:9090/alerts](http://localhost:9090/alerts)

You should see your `TestSlackAlert` appear and turn **RED (Firing)**.

---

### **Alertmanager**

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
```

Then open:
 [http://localhost:9093](http://localhost:9093)

You’ll see the alert routed to the `ops` receiver (Slack).

---

### **Grafana**

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

Then open:
 [http://localhost:3000](http://localhost:3000)

| Credential   | Value                                            |
| ------------ | ------------------------------------------------ |
| **Username** | `admin`                                          |
| **Password** | `REDACTED` *(or as defined in your values file)* |

## Import Your Dashboard

Once Grafana is running and accessible, you can import the prebuilt dashboard to visualize key cluster metrics.

### Steps to Import:

1. On the left-hand menu, hover over the **Dashboards** icon (four squares).
2. Click on **"Import"**.
3. On the Import page, click the **"Upload dashboard JSON file"** button.
4. Select the `grafana-dashboard.json` file from your project directory.
5. On the next screen, select **"Prometheus"** as the data source.
6. Click **"Import"**.

You will now see a complete dashboard visualizing your cluster's health.  


---




###  Author

**Ajay kumar (ExtraordinaryTechy)**
Cloud & DevOps Engineer 

```

---

