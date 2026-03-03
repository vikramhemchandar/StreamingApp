# Deploying StreamingApp via Helm

This document outlines the structure of the new `streamingapp` Helm chart we created, explains its components line-by-line, and provides instructions on how to deploy it.

## 1. What is Helm?
Helm is essentially a package manager for Kubernetes (like "brew" or "apt"). Instead of managing dozens of individual `kubectl apply -f file.yml` commands, Helm bundles all of your services into a single package called a **"Chart"**. 

Helm also solves the problem of hardcoding. By using templates, you can inject variables (like image tags, port numbers, or passwords) right before deployment using a configuration file called `values.yaml`.

---

## 2. Directory Structure Explained

```text
helm/streamingapp/
├── Chart.yaml        # Metadata about this package (version, name, description)
├── values.yaml       # The master configuration file (all your custom settings go here)
└── templates/        # Directory containing the "blueprints" for your resources
    ├── admin-deployment-service.yaml
    ├── auth-deployment-service.yaml
    ├── chat-deployment-service.yaml
    ├── streaming-deployment-service.yaml
    ├── frontend-deployment-service.yaml
    ├── database-deployment-service.yaml
    ├── database-pvc.yaml
    ├── configmap.yaml
    └── ingress.yaml
```

---

## 3. The Details (Line-by-Line)

### A. `Chart.yaml`
This file is very simple. It just tells Helm what this application is.
```yaml
apiVersion: v2
name: streamingapp       # The name of our chart package
description: A Helm chart for deploying the MERN Streaming Application on Kubernetes
type: application        # Specifies this is an app, not a library
version: 0.1.0           # The version number of THIS chart
appVersion: "1.0.0"      # The overall version of your actual software application
```

### B. `values.yaml`
This is arguably the most important file. Every variable defined in here will be automatically copied and injected into the `.yaml` files in the `templates/` folder when you hit deploy. 

```yaml
global:
  registry: 796786461592.dkr.ecr.ap-south-1.amazonaws.com # Your ECR URL

images:
  auth:
    repository: auth      # The name of the auth image in ECR
    tag: v1.0.36          # The specific tag you want to deploy right now
  # ... (Repeated for streaming, admin, chat, frontend, database)

configMap:
  # All the environment variables your app needs to function
  AUTH_PORT: "3001"
  STREAMING_PORT: "3002"
  # ...
  MONGO_URI: "mongodb://database-service:27017/streamingapp" # Internal K8s DNS to the db

ingress:
  className: "nginx"      # The Ingress Controller we installed
  host: "localhost"       # The domain to route traffic from

replicaCount: 2           # How many pods should run for each microservice
databaseReplicaCount: 1   # We only want 1 database pod running

resources:                # Exactly how much CPU and RAM to limit each pod to
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "250m"
    memory: "256Mi"
```

### C. `templates/` Directory
All the files natively in `k8s/` have been moved here, but notice that we removed the hardcoded tags and replaced them with double curly braces `{{ }}`. 

For example, look at what we did inside `templates/auth-deployment-service.yaml`:
```yaml
spec:
  replicas: {{ .Values.replicaCount }}  # Helm will look at values.yaml and inject "2" here!
...
      containers:
        - name: auth-service
          # Helm combines global.registry + images.auth.repository + images.auth.tag
          # Output: 7967864...amazonaws.com/auth:v1.0.36
          image: "{{ .Values.global.registry }}/{{ .Values.images.auth.repository }}:{{ .Values.images.auth.tag }}" 
```

By using this template system, if you push a brand new update (like `v1.0.99`), you never have to touch the `.yaml` files again! You simply update the tag inside `values.yaml` and run the `helm upgrade` command.

---

## 4. How to Deploy Using Helm

### Step 1: Clean Up Legacy Resources
Before we can use Helm, we should delete the old, manually applied resources so they don't conflict. Don't worry, Helm will recreate them!

```bash
kubectl delete -f k8s/
```

### Step 2: Install via Helm
Navigate to your main project directory, and tell Helm to install the chart you just created. We will name this installation block `my-streaming-release`.

```bash
helm install my-streaming-release helm/streamingapp
```
*Helm will instantly read `values.yaml`, inject the variables into the templates, and deploy all 9 files into the cluster simultaneously!*

### Step 3: Verify the Installation
To prove that Helm successfully deployed everything as a package, run:
```bash
helm list
```

### Upgrading in the Future
When Jenkins builds a new tag in the future, all it has to do is update `values.yaml`, and then apply the change by running an `upgrade` command instead of `install`:

```bash
# Example syntax: helm upgrade <ReleaseName> <PathToChart>
helm upgrade my-streaming-release helm/streamingapp
```

### Deleting the Application
If you ever want to uninstall the entire application (including all pods, services, configmaps, and ingresses), Helm does it in one single line:
```bash
helm uninstall my-streaming-release
```
