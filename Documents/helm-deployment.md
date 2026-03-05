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
    ├── frontend-nginx-configmap.yaml
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

### D. Advanced Backend Deployments (Secrets)
In standard Kubernetes, credentials might be stored in a `ConfigMap`. However, for production deployments, all backend services (`auth`, `admin`, `streaming`, `chat`) have been specifically upgraded to pull their sensitive credentials securely from a Kubernetes `Secret` via `secretKeyRef`.

**Example from `auth-deployment-service.yaml`:**
```yaml
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: streamingapp-secrets   # The secure vault created via kubectl
                  key: AWS_ACCESS_KEY_ID       # The specific key to extract
```
*Why this matters:* Secrets are base64 encoded and can be encrypted at rest in the Kubernetes etcd database, preventing anyone with read-access to the cluster from easily stealing your AWS or JWT keys.

### E. Advanced Frontend Architecture (InitContainers & Nginx)
The frontend deployment in this chart contains two highly advanced Kubernetes architectural patterns that solve common Single Page Application (SPA) caching and Cross-Domain API issues.

#### 1. The Nginx ConfigMap (`frontend-nginx-configmap.yaml`)
React router relies on client-side routing. By default, Nginx returns a `404 Not Found` if a user directly visits `http://localhost/admin` because the physical `/admin/index.html` file doesn't exist.
```yaml
      location / {
          try_files $uri $uri/ /index.html;
      }
```
*Line-by-Line Explanation:*
- `location /`: For every single request hitting this container...
- `try_files $uri $uri/`: First, try to serve the exact physical file they asked for (e.g., an image or css file).
- `/index.html`: If a physical file is NOT found, silently redirect the traffic to `index.html` without changing the URL. This hands control to React so it can render the `Admin` dashboard.

#### 2. The Hot-Patching InitContainer (`frontend-deployment-service.yaml`)
When the AWS CI/CD pipeline broke, we were stuck with old React containers that had hardcoded `localhost:3001` API URLs baked into their Javascript. This broke the app on CloudFront. To fix it without rebuilding the Docker image, we injected an `initContainer`.

```yaml
      initContainers:
        - name: patch-react-bundle
          image: "7967864...amazonaws.com/frontend:v1.0.969"
          command: ["sh", "-c"]
          args:
            - |
              cp -a /usr/share/nginx/html/* /patched-html/
              for JS_FILE in /patched-html/static/js/main.*.js; do ...
              sed -i 's|http://localhost:3001/api|/api/auth|g' "$JS_FILE"
          volumeMounts:
            - name: static-html
              mountPath: /patched-html
```

*Line-by-Line Explanation:*
- `initContainers:`: This specifies a specialized container that runs and **must successfully complete** *before* the main Nginx container boots up.
- `cp -a /usr/share/nginx/html/* /patched-html/`: Because standard Docker images are read-only, it copies the React files out of the default directory into a highly-accessible `emptyDir` memory volume.
- `for JS_FILE in /patched-html/static/js/main.*.js;`: Loops through the minified React bundles.
- `sed -i 's|http://localhost:3001/api|/api/auth|g' "$JS_FILE"`: Uses a text-editor script (`sed`) to find the broken absolute URL (`http://localhost:3001/api`) and dynamically replace it with a clean relative path (`/api/auth`).
- `volumeMounts:`: By attaching the `static-html` volume to both the `initContainer` (to edit the files) and the main `frontend` container (to host the files), the Nginx server seamlessly boots up and serves the freshly hot-patched Javascript.

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

**Dynamically Injecting Secrets via Pipeline (Without values.yaml):**
If you have sensitive CI/CD secrets (like AWS keys) injected directly inside Jenkins or GitHub actions, you do NOT want to store them in your `values.yaml`. Instead, use the `--set` argument to overwrite or inject them directly into the cluster at deploy-time:
```bash
helm upgrade streamingapp helm/streamingapp \
  --set secrets.AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID} \
  --set secrets.AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
```
*How it works*: The `--set` flag intercepts Helm as it processes `values.yaml` and replaces whatever paths you specify. In this case, injecting your AWS IAM keys directly into the template securely.

### Deleting the Application
If you ever want to uninstall the entire application (including all pods, services, configmaps, and ingresses), Helm does it in one single line:
```bash
helm uninstall my-streaming-release
```
