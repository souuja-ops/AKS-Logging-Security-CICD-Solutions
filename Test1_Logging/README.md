# Technical Test 1: Logging Solution for AKS

## Objective
Set up a logging solution for an AKS (Azure Kubernetes Service) cluster using Promtail, Loki, and Grafana. The goal is to capture logs from all containers running in the AKS cluster and visualize them in Grafana.

---

## Prerequisites
1. AKS Cluster
2. kubectl CLI connected to the AKS cluster
3. Helm CLI installed

---



## Step 1: Install Promtail (Log Shipper)

**Promtail** is used to collect logs from the Kubernetes nodes and ship them to Loki.

### Add Loki Helm Repository
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
### Create Logging Namespace
```bash
kubectl create namespace logging
```
### Install Promtail as a DaemonSet
```bash
helm install promtail grafana/promtail --namespace=logging --set "loki.serviceName=loki"
```
### Promtail Configuration (promtail-config.yaml)
This is an example configuration file for Promtail to collect logs from Kubernetes pods:
```bash
server:
  http_listen_port: 9080

positions:
  filename: /var/log/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```
### Verify Promtail Installation
Ensure Promtail is running as a DaemonSet:
```bash
kubectl get daemonsets -n logging
```

## Step 2: Install Loki (Log Aggregator)

**Loki** is the central component responsible for aggregating logs.

### Install Loki Using Helm
```bash
helm install loki grafana/loki-stack --namespace=logging --set promtail.enabled=false
```
### Loki Storage Configuration
This example configures Loki to use a filesystem for storage:
```bash
storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/boltdb-cache
    shared_store: filesystem

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
```
### Verify Loki Logs
Check the logs of the Loki pod:
```bash
kubectl logs -l app=loki -n logging
```


## Step 3: Install Grafana (Log Visualization)

**Grafana** is used for querying and visualizing logs aggregated by Loki.

### Install Grafana Using Helm
```bash
helm install grafana grafana/grafana --namespace=logging
```
### Expose Grafana Service
```bash
kubectl expose service grafana --type=LoadBalancer --name=grafana-loadbalancer -n logging
```
### Add Loki as a Data Source in Grafana
* Log into Grafana (default credentials: admin/admin).
* Navigate to Configuration -> Data Sources -> Add Data Source.
* Select Loki.
* Set the URL to http://loki:3100/.
* Click Save & Test.

## Step 4: Verify the End-to-End Logging Setup

### Deploy a test application to generate logs:
```bash
kubectl run my-app --image=nginx --namespace=default
```
### Query logs in Grafana:
* Go to Explore in Grafana.
* Use the query:
```bash
{namespace="default", container="my-app"}
```

## Optional: Set Up Alerts in Grafana
### Configure Alerting
* Go to Alerting -> Notification Channels.
* Add a notification channel (e.g., email, Slack).

### Define Alert Rules
Create alert rules based on log queries, for example:

```bash
expr: sum(rate({job="nginx"}[5m])) > 5
for: 5m
labels:
  severity: critical
```
