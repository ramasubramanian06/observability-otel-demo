# Installation Guide

Step-by-step setup for the Observability project — EKS cluster, OpenTelemetry Demo, and the Prometheus/Grafana monitoring stack.

---

## Prerequisites

Make sure the following tools are installed and configured before starting:

- [ ] **AWS CLI** — [Install guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), then run `aws configure`
- [ ] **eksctl** — [Install guide](https://eksctl.io/installation/)
- [ ] **kubectl** — [Install guide](https://kubernetes.io/docs/tasks/tools/)
- [ ] **Helm** — [Install guide](https://helm.sh/docs/intro/install/)

---

## Step 1: Create the EKS Cluster

### 1.1 Create the cluster (without a nodegroup)

```bash
eksctl create cluster --name=observability \
                       --region=us-east-1 \
                       --zones=us-east-1a,us-east-1b \
                       --without-nodegroup
```

### 1.2 Associate the IAM OIDC provider

```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster observability \
    --approve
```

### 1.3 Create the managed nodegroup

```bash
eksctl create nodegroup --cluster=observability \
                         --region=us-east-1 \
                         --name=observability-ng-private \
                         --node-type=t3.medium \
                         --nodes-min=2 \
                         --nodes-max=3 \
                         --node-volume-size=20 \
                         --managed \
                         --asg-access \
                         --external-dns-access \
                         --full-ecr-access \
                         --appmesh-access \
                         --alb-ingress-access \
                         --node-private-networking
```

### 1.4 Update your kubeconfig

```bash
aws eks update-kubeconfig --name observability
```

---

## Step 2: Install the OpenTelemetry Demo (via Helm)

### 2.1 Add the OpenTelemetry Helm repository

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

### 2.2 Install the chart

```bash
helm install my-otel-demo open-telemetry/opentelemetry-demo
```

### 2.3 Expose the frontend-proxy service

> Replace `default` with your Helm release namespace if different.

```bash
kubectl --namespace default port-forward svc/frontend-proxy 8080:8080
```

---

## Step 3: Install kube-prometheus-stack

### 3.1 Add the Prometheus Community Helm repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 3.2 Create the monitoring namespace

```bash
kubectl create ns monitoring
```

### 3.3 Deploy the chart

```bash
cd day-2

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f ./custom_kube_prometheus_stack.yml
```

---

## Step 4: Verify the Installation

### 4.1 Check all resources in the monitoring namespace

```bash
kubectl get all -n monitoring
```

### 4.2 Access Prometheus UI

```bash
kubectl port-forward service/prometheus-operated -n monitoring 9090:9090
```

> **Note:** If running on an EC2 instance or cloud VM, add `--address 0.0.0.0` to the command above, then access via `<instance-ip>:9090`.

### 4.3 Access Grafana UI

**Default credentials** (set in `custom_kube_prometheus_stack.yml`):

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `prom-operator` |

**If credentials don't work or no custom config was used**, retrieve them from the Kubernetes secret:

```bash
# Get username
kubectl get secret --namespace monitoring monitoring-grafana -o jsonpath='{.data.admin-user}' | base64 -d

# Get password
kubectl get secret --namespace monitoring monitoring-grafana -o jsonpath='{.data.admin-password}' | base64 -d
```

**Port-forward to access Grafana:**

```bash
kubectl port-forward service/monitoring-grafana -n monitoring 8080:80
```

### 4.4 Access Alertmanager UI

```bash
kubectl port-forward service/alertmanager-operated -n monitoring 9093:9093
```

---

## Accessing the Application

With the `frontend-proxy` port-forward running (Step 2.3), the following are available:

| Service | URL |
|---|---|
| Web Store | http://localhost:8080/ |
| Grafana | http://localhost:8080/grafana/ |
| Load Generator UI | http://localhost:8080/loadgen/ |
| Jaeger UI | http://localhost:8080/jaeger/ui/ |
| Flagd Configurator UI | http://localhost:8080/feature |

---

## Cleanup

To avoid ongoing AWS charges, delete the cluster when you're done:

```bash
eksctl delete cluster --name=observability --region=us-east-1
```
