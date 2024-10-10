# Technical Test 3: Securing AKS with ModSecurity Policies

## Objective
Improve the security of applications exposed via an NGINX Ingress controller on an AKS (Azure Kubernetes Service) cluster by implementing ModSecurity policies. This involves detecting and preventing common attacks such as SQL injection, cross-site scripting (XSS), and more.

---

## Step 1: Install NGINX Ingress Controller

To apply ModSecurity policies, an NGINX Ingress controller must be installed in the AKS cluster and configured to accept ModSecurity rules.

### Create an Ingress Namespace
First, create a namespace for the Ingress controller:
```bash
kubectl create namespace ingress-nginx
```
### Install NGINX Ingress Controller
Use Helm to install the NGINX Ingress controller with ModSecurity support:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.admissionWebhooks.enabled=false
```  
### Verify the Installation
Ensure the NGINX Ingress controller pods are running:
```bash
kubectl get pods -n ingress-nginx
```
## Step 2: Enable ModSecurity on the Ingress Controller
Next, we need to modify the NGINX Ingress controller to enable ModSecurity. This allows us to apply security policies to filter incoming traffic.

### Edit the NGINX ConfigMap
Modify the NGINX controller's configuration to enable ModSecurity:
```bash
kubectl edit configmap nginx-configuration -n ingress-nginx
```
Add or modify the following lines to enable ModSecurity in Detection mode or Blocking mode:
```bash
data:
  enable-modsecurity: "true"
  modsecurity-snippet: |
    SecRuleEngine DetectionOnly    # Use "On" for Blocking mode
```    
* Detection Mode: Detects malicious requests but does not block them.
* Blocking Mode: Actively blocks malicious requests.

### Apply the Configuration Changes
After editing the ConfigMap, the changes will automatically propagate to the NGINX pods. Restart the NGINX controller to apply the updated configuration:
```bash
kubectl rollout restart deployment nginx-ingress-controller -n ingress-nginx
```
## Step 3: Apply ModSecurity Policies
ModSecurity rules are defined in a modsecurity.conf file, which specifies how the Ingress controller should handle different types of attacks.

### Create a ConfigMap for ModSecurity Rules
Create a ConfigMap to store ModSecurity rules that will be referenced by the Ingress controller.
Example modsecurity.conf file to prevent SQL injection, XSS, and other attacks:
```bash
kubectl create configmap modsecurity-config --from-file=modsecurity.conf -n ingress-nginx
```
Example ModSecurity Configuration (modsecurity.conf)
```bash
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess Off
SecRule REQUEST_URI "@rx \.\./" "id:1000,phase:1,t:none,block,msg:'Directory Traversal Attack'"
SecRule ARGS "@rx select.+from" "id:1001,phase:2,t:none,block,msg:'SQL Injection Detected'"
SecRule ARGS "@rx <script>" "id:1002,phase:2,t:none,block,msg:'XSS Attack Detected'"
SecRule REQUEST_HEADERS:User-Agent "@contains sqlmap" "id:1003,phase:1,t:none,block,msg:'SQLMap Detected'"
```
### Reference ModSecurity Config in Ingress
In your Ingress resource, reference the ModSecurity configuration using the nginx.ingress.kubernetes.io/modsecurity-snippet annotation.
Example:
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: your-app-namespace
  annotations:
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/modsecurity-snippet: |
      SecRuleEngine On
      SecRequestBodyAccess On
      SecResponseBodyAccess Off
      Include /etc/nginx/modsecurity/modsecurity.conf
spec:
  rules:
    - host: your-app.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: your-app-service
                port:
                  number: 80
```

### Apply the Ingress Resource
Apply the Ingress configuration with ModSecurity enabled:
```bash
kubectl apply -f secure-ingress.yaml
```
## Step 4: Monitoring and Testing
### Test ModSecurity Configuration
Test the ModSecurity setup by sending requests that would trigger ModSecurity rules, such as SQL injection or XSS payloads, and checking whether the requests are blocked.

Example: SQL injection attack attempt
```bash
curl http://your-app.com?query=SELECT%20*%20FROM%20users
```
### Monitor Logs
Check the NGINX logs to verify that ModSecurity is working as expected:
```bash
kubectl logs -l app.kubernetes.io/name=ingress-nginx -n ingress-nginx
```
Look for ModSecurity messages like "SQL Injection Detected" or "XSS Attack Detected."