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