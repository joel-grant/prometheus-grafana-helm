# Prometheus + Grafana Monitoring Stack

This repository contains a customized Helm chart that wraps the excellent community-maintained Prometheus monitoring charts with opinionated defaults for production Kubernetes monitoring.

## What This Repository Provides

This is **not a custom implementation** - it's a thin wrapper around the battle-tested [Prometheus Community Helm Charts](https://github.com/prometheus-community/helm-charts), specifically:

- **[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)** - The complete Prometheus monitoring solution
- **[prometheus-postgres-exporter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-postgres-exporter)** - PostgreSQL database monitoring

### What Gets Installed

When you deploy this chart, you get a complete monitoring stack:

**Core Components (from kube-prometheus-stack):**
- **Prometheus Server** - Metrics collection and storage with 30-day retention
- **Grafana** - Visualization with 50+ pre-built Kubernetes dashboards
- **Alertmanager** - Alert routing and notification management
- **Node Exporter** - Hardware and OS metrics from all cluster nodes
- **Kube State Metrics** - Kubernetes object metrics (pods, deployments, services, etc.)
- **Prometheus Operator** - Manages Prometheus instances and ServiceMonitors

**Additional Components:**
- **PostgreSQL Exporter** - Monitors Digital Ocean (or any) PostgreSQL databases
- **Persistent Storage** - Retains metrics and Grafana dashboards across pod restarts
- **Service Discovery** - Automatically discovers applications with Prometheus annotations

## Why Use Community Charts?

✅ **Maintained by experts** - The Prometheus community actively maintains these charts  
✅ **Battle-tested** - Used by thousands of organizations worldwide  
✅ **Regular updates** - Security patches, bug fixes, and new features  
✅ **Best practices** - Follows Kubernetes and Prometheus recommendations  
✅ **Pre-built dashboards** - Comprehensive Grafana dashboards included  
✅ **Sensible defaults** - Works out of the box with minimal configuration  

## Repository Purpose

This is both a **demonstration/learning repository** and a **production-ready configuration** that I use in my own Kubernetes clusters. The customizations focus on:

- **Persistent storage** - Default charts use ephemeral storage
- **Extended retention** - 30 days vs default 10 days  
- **Database monitoring** - PostgreSQL exporter for external databases
- **Production-ready values** - Separate configurations for dev/staging/production

## Credits

Full credit goes to the [Prometheus Community](https://github.com/prometheus-community/helm-charts) for creating and maintaining these excellent charts. This repository simply provides:
- Curated configuration values
- Documentation for common use cases
- Production and development value examples

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- Digital Ocean PostgreSQL database (optional)

## Installation

### 1. Add Helm Repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. Update Dependencies

```bash
helm dependency update
```

### 3. Install the Chart

#### Basic Installation
```bash
helm install prometheus-grafana . \
  --namespace monitoring \
  --create-namespace
```

#### With PostgreSQL Monitoring
```bash
# Create PostgreSQL secret first
kubectl create secret generic postgres-secret -n monitoring \
  --from-literal=DATA_SOURCE_NAME="postgresql://user:pass@your-host.db.ondigitalocean.com:25060/defaultdb?sslmode=require"

# Install with PostgreSQL monitoring
helm install prometheus-grafana . \
  --namespace monitoring \
  --create-namespace \
  --set postgresql.enabled=true
```

### 4. Access Your Monitoring Stack

#### Grafana Dashboard
```bash
kubectl port-forward -n monitoring svc/prometheus-grafana-grafana 3000:80
```
- URL: http://localhost:3000
- Username: `admin`
- Password: `admin` (change this in production!)

#### Prometheus Server
```bash
kubectl port-forward -n monitoring svc/prometheus-grafana-kube-prometheus-prometheus 9090:9090
```
- URL: http://localhost:9090

#### Alertmanager
```bash
kubectl port-forward -n monitoring svc/prometheus-grafana-kube-prometheus-alertmanager 9093:9093
```
- URL: http://localhost:9093

## Configuration

### Application Monitoring

To monitor your applications, add these annotations to your pods/services:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

### Storage Configuration

Customize storage in `values.yaml`:

```yaml
kube-prometheus-stack:
  prometheus:
    prometheusSpec:
      storageSpec:
        volumeClaimTemplate:
          spec:
            storageClassName: "fast-ssd"  # Your storage class
            resources:
              requests:
                storage: 100Gi
```

### Resource Limits

Adjust resources based on your cluster size:

```yaml
kube-prometheus-stack:
  prometheus:
    prometheusSpec:
      resources:
        limits:
          cpu: 4000m
          memory: 8Gi
        requests:
          cpu: 1000m
          memory: 4Gi
```

### Ingress Setup

Enable ingress for external access:

```yaml
kube-prometheus-stack:
  grafana:
    ingress:
      enabled: true
      ingressClassName: nginx
      hosts:
        - grafana.yourdomain.com
      tls:
        - secretName: grafana-tls
          hosts:
            - grafana.yourdomain.com
```

## Digital Ocean PostgreSQL Setup

1. Create connection secret:
```bash
kubectl create secret generic postgres-secret -n monitoring \
  --from-literal=DATA_SOURCE_NAME="postgresql://username:password@host:25060/database?sslmode=require"
```

2. Enable PostgreSQL monitoring:
```yaml
postgresql:
  enabled: true
```

## Environment-Specific Configurations

### Production
- Use persistent storage
- Enable ingress with TLS
- Configure proper alerting
- Set appropriate resource limits
- Enable monitoring across all namespaces

### Development  
- Use smaller storage
- NodePort services for easy access
- Reduced resource limits
- Monitor fewer namespaces

## Monitoring Namespaces

The community chart automatically monitors all namespaces by default and includes comprehensive service discovery. Applications in any namespace can be monitored by adding the standard Prometheus annotations:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

The monitoring stack will discover and scrape these applications automatically - no additional configuration needed.

## What You Get Out of the Box

The kube-prometheus-stack community chart provides an incredible amount of functionality:

- **50+ Pre-built Grafana Dashboards** including:
  - Kubernetes cluster overview and capacity planning
  - Node-level CPU, memory, disk, and network metrics
  - Pod and container resource usage
  - Kubernetes API server and etcd metrics
  - Ingress controller and networking metrics
  - Persistent volume and storage metrics

- **Pre-configured Alerting Rules** for:
  - Node resource exhaustion (CPU, memory, disk)
  - Kubernetes component health (API server, kubelet, etc.)
  - Pod crash loops and failed deployments
  - Certificate expiration warnings
  - And many more production-ready alerts

- **Automatic Service Discovery** for:
  - All Kubernetes system components
  - Applications with `prometheus.io/scrape: "true"` annotations
  - ServiceMonitor and PodMonitor custom resources

- **PostgreSQL Database Monitoring** (when enabled):
  - Connection and query performance metrics
  - Database size and table statistics
  - Lock and transaction metrics
  - Custom queries for Digital Ocean managed databases

All of this comes from the upstream community charts - this repository just adds the configuration to enable persistence and extend retention periods.

## Upgrading

```bash
helm repo update
helm upgrade prometheus-grafana . --namespace monitoring
```

## Uninstall

```bash
helm uninstall prometheus-grafana --namespace monitoring
```

## Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n monitoring
```

### View Prometheus Targets
1. Port-forward to Prometheus: `kubectl port-forward -n monitoring svc/prometheus-grafana-kube-prometheus-prometheus 9090:9090`
2. Open: http://localhost:9090/targets

### Check PostgreSQL Connection
```bash
kubectl logs -n monitoring deployment/prometheus-grafana-prometheus-postgres-exporter
```

### Grafana Login Issues
The default credentials are `admin/admin`. Reset the password:
```bash
kubectl get secret -n monitoring prometheus-grafana-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
