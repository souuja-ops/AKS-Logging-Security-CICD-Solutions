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