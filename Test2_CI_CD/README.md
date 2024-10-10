# Technical Test 2: CI/CD Pipeline Using Tekton for AKS

## Objective
Set up a CI/CD pipeline using Tekton to deploy an application to an AKS (Azure Kubernetes Service) cluster. This pipeline will automate building, testing, and deploying the application to AKS.

---

## Prerequisites
1. **AKS Cluster**: A running AKS cluster with kubectl configured.
2. **Tekton**: Tekton CLI (`tkn`) installed on your local machine.
3. **kubectl**: The Kubernetes CLI (`kubectl`) connected to the AKS cluster.
4. **Helm**: Helm CLI installed to simplify chart installations.

---

## Step 1: Install Tekton Pipelines on AKS

### Create a Tekton Namespace
It's a good practice to create a namespace for Tekton resources:
```bash
kubectl create namespace tekton-pipelines
```
### Install Tekton Pipelines
To install Tekton, run the following command:
```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

### Verify Installation
Ensure Tekton is installed and running:
```bash
kubectl get pods --namespace tekton-pipelines
```
## Step 2: Define the Tekton Pipeline
The Tekton pipeline will consist of tasks such as cloning the repository, building the application, and deploying it to AKS.

## Create a GitHub Secret (Optional)
If your source code is in a private GitHub repository, create a secret for authentication:
```bash
kubectl create secret generic github-secret \
  --from-literal=username=<your-username> \
  --from-literal=password=<your-token> \
  --namespace=tekton-pipelines
```  
## Pipeline Configuration (pipeline.yaml)
Below is an example of the pipeline configuration for Tekton to build and deploy to AKS:
```bash
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: aks-deployment-pipeline
  namespace: tekton-pipelines
spec:
  tasks:
    - name: fetch-repository
      taskRef:
        name: git-clone
        kind: ClusterTask
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: "https://github.com/your-repo/your-app.git"
        - name: revision
          value: "main"

    - name: build-and-push
      taskRef:
        name: kaniko
        kind: ClusterTask
      params:
        - name: IMAGE
          value: "your-registry/your-app:latest"
      workspaces:
        - name: source
          workspace: shared-workspace

    - name: deploy-to-aks
      taskRef:
        name: kubectl-deploy
        kind: ClusterTask
      params:
        - name: manifests
          value: "k8s/deployment.yaml"
      workspaces:
        - name: source
          workspace: shared-workspace

workspaces:
  - name: shared-workspace
```  
## Tasks Overview:
* fetch-repository: This task clones the source code from the Git repository.
* build-and-push: Builds the application using Kaniko and pushes the Docker image to a container registry.
* deploy-to-aks: Deploys the built image to the AKS cluster using kubectl.

### Step 3: Apply the Pipeline and Resources

## Apply the Tekton Pipeline
Run the following command to apply the pipeline to your AKS cluster:
```bash
kubectl apply -f pipeline.yaml -n tekton-pipelines
``` 

## Create PipelineRun (pipeline-run.yaml)
A PipelineRun triggers the pipeline execution. 
```bash
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: aks-deployment-pipeline-run
  namespace: tekton-pipelines
spec:
  pipelineRef:
    name: aks-deployment-pipeline
  workspaces:
    - name: shared-workspace
      persistentVolumeClaim:
        claimName: tekton-pvc
  serviceAccountName: pipeline-service-account
```  
## Apply the PipelineRun
Apply the PipelineRun to start the pipeline:
```bash
kubectl apply -f pipeline-run.yaml -n tekton-pipelines
```
### Step 4: Monitoring and Verifying the Pipeline
## Monitor Pipeline Execution
Use Tekton CLI (tkn) to monitor the pipeline run:
```bash
tkn pipelinerun logs aks-deployment-pipeline-run -f -n tekton-pipelines
```
This will display the logs of each task in real-time.

## Verify Deployment on AKS
Once the pipeline completes, check if the application is running on AKS:
```bash
kubectl get pods -n <your-app-namespace>
```
### Step 5: Clean Up
After testing clean up the Tekton resources:
```bash
kubectl delete pipelinerun aks-deployment-pipeline-run -n tekton-pipelines
kubectl delete pipeline aks-deployment-pipeline -n tekton-pipelines
kubectl delete namespace tekton-pipelines
```