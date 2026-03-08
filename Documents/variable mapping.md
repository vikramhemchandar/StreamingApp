# Kubernetes Helm Variable Mapping Guide

This document traces the complete lifecycle of every single variable used in this project's Helm deployment. It tracks variables from their birth in `values.yaml`, their transformation through intermediate files (like `configmap.yaml`), to their final destination injected into running Kubernetes Pods and Services.

---

## 1. Centralized Control: The `values.yaml` File

The `values.yaml` file acts as the singular source of truth for the entire application. Modifying a variable here automatically updates it across all manifests when running a `helm upgrade`.

### A. Global and Image Variables
These variables control *where* the Docker images are located and *which* version to pull.

*   **`global.registry`**: The AWS ECR domain (e.g., `796786461592.dkr.ecr.ap-south-1.amazonaws.com`).
*   **`images.[service_name].repository`**: The name of the Docker repository for that specific service (e.g., `auth`, `admin`).
*   **`images.[service_name].tag`**: The specific version tag of the Docker image to deploy (e.g., `v1.0.969`).

### B. Configuration Variables (`configMap` block)
These variables control internal ports, URIs, and cloud integrations. They are safe to store in plaintext.

*   **`configMap.AUTH_PORT`**, **`configMap.STREAMING_PORT`**, **`configMap.ADMIN_PORT`**, **`configMap.CHAT_PORT`**: Internal ports Node.js services run on.
*   **`configMap.MONGO_URI`**: The internal connection string for MongoDB.
*   **`configMap.JWT_SECRET`**: A placeholder secret value (deprecated in favor of Kubernetes Secrets).
*   **`configMap.CLIENT_URLS`**: allowed CORS origins.
*   **`configMap.AWS_REGION`**, **`configMap.AWS_S3_BUCKET`**, **`configMap.AWS_CDN_URL`**: Hardcoded data for AWS integration.

### C. Scaling and Resources
*   **`ingress.className`**, **`ingress.host`**: Network settings for the outer traffic layer.
*   **`replicaCount`**: Number of backend pods to run.
*   **`databaseReplicaCount`**: Number of database pods.
*   **`resources.requests`** & **`resources.limits`**: Minimum requested and maximum allowed CPU/Memory usage per pod.

---

## 2. The Bridge: `templates/configmap.yaml`

The backend Node.js code doesn't magically know how to read a Helm `values.yaml` file. We must translate the Helm values into a Kubernetes object using Go templating (`{{ ... }}`).

**How the Mapping Works:**
```yaml
# Inside templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap
data:
  # The LEFT side (AUTH_PORT) is the internal Kubernetes Key.
  # The RIGHT side ({{ .Values... }}) is the data pulled from values.yaml.
  AUTH_PORT: {{ .Values.configMap.AUTH_PORT | quote }}
  MONGO_URI: {{ .Values.configMap.MONGO_URI | quote }}
  AWS_REGION: {{ .Values.configMap.AWS_REGION | quote }}
  # ... (and so on)
```
When Helm deploys, it replaces `{{ .Values.configMap.AUTH_PORT | quote }}` with `"3001"`. The ConfigMap acts as a giant `.env` file waiting inside the cluster.

---

## 3. The Injection: Deployment Files

This is the final destination. Let's look at how variables map inside a backend Deployment file like `templates/admin-deployment-service.yaml`.

### A. Docker Image Mapping
The Pod needs to know exactly what image to pull. We concatenate three variables from `values.yaml` using Helm templating:
```yaml
# Inside admin-deployment-service.yaml (Line 17)
image: "{{ .Values.global.registry }}/{{ .Values.images.admin.repository }}:{{ .Values.images.admin.tag }}"
```
*Flow Result*: `7967864...amazonaws.com/admin:v1.0.969`

### B. Scalability and Resources Mapping
```yaml
replicas: {{ .Values.replicaCount }} # Injects '2'
resources:
  requests:
    cpu: {{ .Values.resources.requests.cpu | quote }}       # Injects "100m"
    memory: {{ .Values.resources.requests.memory | quote }} # Injects "128Mi"
```

### C. Environment Variable Mapping (The Crucial Step!)
We map the data from the ConfigMap directly into the Pod's environment variables (`process.env`). Notice how we remap names:

```yaml
env:
  - name: PORT          # Node.js strictly looks for 'process.env.PORT'
    valueFrom:
      configMapKeyRef:
        name: configmap # Check the bridge file...
        key: ADMIN_PORT # Reach in and grab the '3003' value!
```

---

## 4. Protected Variables (Kubernetes Secrets)

Sensitive keys like AWS Passwords and JWT Tokens should **never** be saved in plaintext in a Git-tracked `values.yaml`. Therefore, they bypass Helm!

**The Flow:**
1. A manual command runs securely on the server:
   `kubectl create secret generic streamingapp-secrets --from-literal=JWT_SECRET="REAL_SECRET"`
2. The Deployment injects it securely without using the ConfigMap:
```yaml
env:
  - name: JWT_SECRET
    valueFrom:
      secretKeyRef:
        name: streamingapp-secrets # Grabs the manually created protected file
        key: JWT_SECRET            # Injects the real token 
```
Variables protected this way: `JWT_SECRET`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`.

---

## 5. Detailed Case Study: How `MONGO_URI` is Formed

It is a common error to use `localhost:27017` in Kubernetes. The database is isolated in its own networking bubble.

**Step 1: The Service Map Formation**
In `database-deployment-service.yaml`, we create a network router for the database:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-service # <--- This becomes the literal internal DNS address!
spec:
  ports:
    - port: 27017        # <--- The open port
```

**Step 2: Declaration in `values.yaml`**
We build the connection string using the exact DNS Name and Port established above:
```yaml
configMap:
  MONGO_URI: "mongodb://database-service:27017/streamingapp"
```

**Step 3: Translation in `configmap.yaml`**
```yaml
MONGO_URI: {{ .Values.configMap.MONGO_URI | quote }}
```

**Step 4: Final Injection in Backend Deployments**
The Node.js Pod mounts it for Mongoose to connect to the database securely inside the isolated cluster network.
```yaml
env:
  - name: MONGO_URI
    valueFrom:
      configMapKeyRef:
        name: configmap
        key: MONGO_URI
```
