# Kubernetes Troubleshooting Guide: Streaming Application

This document outlines the step-by-step troubleshooting process used to resolve issues where multiple microservices (Admin, Chat, Streaming, Auth, and Frontend) were struggling to reach the `1/1 READY` state and were stuck in `CrashLoopBackOff`. 

---

## 🛑 Issue 1: ConfigMap Key Duplication (The "Poisoning" Bug)

### **Symptoms**
When inspecting the logs for the backend pods using `kubectl logs pod-name`, the Node.js services were starting on the incorrect port (e.g., all starting on 3004) and attempting to connect to MongoDB on `localhost` instead of the Kubernetes database service.

### **Root Cause**
In Kubernetes, a `ConfigMap` is parsed as a YAML dictionary. In the original `configmap.yml`, keys like `PORT` and `MONGO_URI` were defined multiple times (once for each microservice). In YAML, duplicate keys override previous ones, meaning whichever value was at the **bottom of the file** (which belonged to the Chat service) was applied to **all** services.

As a result, every service was injected with `PORT=3004` and `MONGO_URI=mongodb://localhost:27017/streamingapp`. Note: Inside a Kubernetes pod, `localhost` refers to the container itself, meaning it was looking for a database inside its own application container, which failed.

### **The Fix**
1. **In `configmap.yml`**: Renamed the duplicate keys to be globally unique (e.g., `PORT` -> `AUTH_PORT`, `STREAMING_PORT`, etc.). Consolidated `MONGO_URI` to a single shared key pointing to the correct Kubernetes service (`database-service`), not `localhost`.
2. **In all Backend Deployment YAMLs**: Updated the `configMapKeyRef` to pull the specific unique port keys instead of the generic `PORT` key.

### **Code Changes**
**Old `configmap.yml`:**
```yaml
data:
  # Auth Service
  PORT: "3001"
  MONGO_URI: "mongodb://mongo:27017/streamingapp"
  ...
  # Admin Service
  PORT: "3003"
  MONGO_URI: "mongodb://localhost:27017/streamingapp"
  ...
  # Chat Service
  PORT: "3004"
  MONGO_URI: "mongodb://localhost:27017/streamingapp"
```

**New `configmap.yml`:**
```yaml
data:
  # Unique Port configurations
  AUTH_PORT: "3001"
  STREAMING_PORT: "3002"
  ADMIN_PORT: "3003"
  CHAT_PORT: "3004"
  
  # Shared backend configurations
  MONGO_URI: "mongodb://database-service:27017/streamingapp"
```

**Old `admin-deployment-service.yml`:**
```yaml
            - name: PORT
              valueFrom:
                configMapKeyRef:
                  name: configmap
                  key: PORT        # <-- Pulled the overwritten value "3004"
```

**New `admin-deployment-service.yml`:**
```yaml
            - name: PORT
              valueFrom:
                configMapKeyRef:
                  name: configmap
                  key: ADMIN_PORT  # <-- Correctly pulls "3003"
```

---

## 🛑 Issue 2: Incorrect Health Check Probe Paths (The 404 Trap)

### **Symptoms**
After fixing the ConfigMap, the Pods were successfully starting up on the correct ports (verified via `kubectl logs`). However, they remained `0/1 READY` and Kubernetes aggressively killed and restarted them (`CrashLoopBackOff`). 

Checking `kubectl get events` revealed:
`Readiness probe failed: HTTP probe failed with statuscode: 404`

### **Root Cause**
The Kubernetes `livenessProbe` and `readinessProbe` were configured to ping the endpoint `path: /health`. Kubernetes faithfully sent HTTP GET requests to `/health`, but the backend developers wrote the health endpoint in Express.js as `/api/health`. Because the route didn't exist at `/health`, Node.js returned a `404 Not Found`. Kubernetes interpreted this 404 as "the application is dead/frozen" and terminated the pod.

### **The Fix**
Updated the `path` for both probes in `admin-deployment-service.yml`, `chat-deployment-service.yml`, and `streaming-deployment-service.yml` to match the actual route written in the Node.js code (`/api/health`).

### **Code Changes (Admin, Chat, and Streaming Deployments)**
**Old Configuration:**
```yaml
          livenessProbe:
            httpGet:
              path: /health
              port: 3003
```

**New Configuration:**
```yaml
          livenessProbe:
            httpGet:
              path: /api/health
              port: 3003
```

---

## 🛑 Issue 3: Frontend Nginx Port Misconfiguration

### **Symptoms**
The frontend pod was immediately terminating with health probe connection refused errors (e.g., `dial tcp IP:3000: connect: connection refused`).

### **Root Cause**
The frontend React app was built and served using an `nginx` Alpine image inside the Dockerfile. By default, Nginx listens on port `80`. However, the Kubernetes deployment was configured as if the application was running a local Node server on port `3000`. `containerPort: 3000` was specified, and the probes were pinging port `3000`. Ultimately, there was no responsive service listening on port `3000` inside the container.

### **The Fix**
1. Changed `containerPort` to `80`.
2. Pointed the health probes to ping port `80` at the root path `/` (checking if the Nginx home page loads).
3. Adjusted the Kubernetes Service mapping so that the `targetPort` points to the container's real port of `80`.

### **Code Changes (`frontend-deployment-service.yml`)**
**Old Configuration:**
```yaml
        - name: frontend
          image: vikramhemchandar/streamingapp:frontendv1.0
          ports:
            - containerPort: 3000
...
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
...
---
  ports:
    - port: 3000
      targetPort: 3000
```


**New Configuration:**
```yaml
        - name: frontend
          image: vikramhemchandar/streamingapp:frontendv1.0
          ports:
            - containerPort: 80
...
          livenessProbe:
            httpGet:
              path: /
              port: 80
...
---
  ports:
    - port: 3000
      targetPort: 80
```

---

## 🛠️ General Troubleshooting Flow for Future Reference
If your pods are ever stubbornly failing, follow this exact order of operations:

1. **Check Pod Status**: 
   ```bash
   kubectl get pods
   ```
   *Look for `CrashLoopBackOff`, `Error`, or `CreateContainerConfigError`.*

2. **Check Application Logs**: 
   ```bash
   kubectl logs <pod-name> 
   ```
   *This reveals errors thrown by your actual code (e.g., "Failed to connect to Mongo", "Missing JWT Secret"). Use this to diagnose Environment Variable/ConfigMap issues.*

3. **Check Kubernetes Events**: 
   ```bash
   kubectl get events --sort-by='.metadata.creationTimestamp'
   ```
   *If the logs are empty or the app starts fine but gets killed, the events will tell you if Kubernetes is intentionally assassinating the pod due to failed Liveness/Readiness probes, OOMKilled (Out of Memory), or scheduling limits.*

---

## 🛑 Issue 4: ECR Authentication & Docker Desktop Credential Store Bug (`ErrImagePull`)

### **Symptoms**
Pods were stuck in `ImagePullBackOff` and `ErrImagePull`. Kubernetes was unauthorized to pull the private images from AWS ECR, even after running `aws ecr get-login-password | docker login`.

### **Root Cause**
Docker Desktop on Mac intercepts container credential management through a tool called `desktop` (configured in `~/.docker/config.json` under `credsStore: desktop`). Because of this interception, the raw AWS access token generated by the CLI was locked inside the macOS Keychain instead of being placed directly inside the configuration file. When Kubernetes attempted to use the local docker credentials to create its secret, it read an empty file and was subsequently denied access by AWS.

### **The Fix**
Bypassed the Docker Desktop Keychain by manually generating the token and piping it directly into Kubernetes without touching local Docker files.

**Commands Used to Debug & Fix:**
```bash
# Verify the failing pods
kubectl get pods

# Drill into the pod to see the exact Kubelet pull error
kubectl describe pod <pod-name>

# The Master Fix: Inject the raw AWS token directly into the cluster
kubectl create secret docker-registry regcred \
  --docker-server=796786461592.dkr.ecr.ap-south-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-south-1)

# Force the pods to restart and apply the new secret
kubectl rollout restart deployment
```

---

## 🛑 Issue 5: Secret Architecture Mismatch (`CreateContainerConfigError`)

### **Symptoms**
Pods were failing to start with `CreateContainerConfigError`. `kubectl describe pod` revealed that the pod was trying to pull the `JWT_SECRET` and `AWS_ACCESS_KEY` from a `ConfigMap` that didn't have them.

### **Root Cause**
The credentials had been successfully migrated to a highly secure Kubernetes `Secret` named `streamingapp-secrets`. However, the Helm chart templates (`admin-deployment-service.yaml`, `auth-deployment-service.yaml`, etc.) had outdated configurations and were still searching for `configMapKeyRef` instead of `secretKeyRef`.

### **The Fix**
Refactored the Helm deployment YAMLs to extract sensitive credentials from the secure `secretKeyRef` instead.

**Commands Used to Debug & Fix:**
```bash
# Check why the container config failed
kubectl describe pod <pod-name>

# View the actual secrets loaded in the cluster
kubectl get secret streamingapp-secrets -o yaml

# Upgrade the Helm cluster after patching the YAMLs
helm upgrade streamingapp helm/streamingapp
```

---

## 🛑 Issue 6: React SPA Routing & 404 Trap (The Internal Tour Guide)

### **Symptoms**
The application loaded fine at the root URL `http://localhost`, but navigating directly to `http://localhost/admin` or refreshing the page resulted in an Nginx `404 Not Found` error.

### **Root Cause**
> 💡 **The Layman's Analogy:** Imagine your guest arrives at the Frontend "Lobby" and asks the internal Tour Guide (the Nginx web server) for the `/admin` room. By default, the Tour Guide looks at their physical building map, sees no physical door labeled "admin.html", completely panics, and screams: **"404 NOT FOUND!"** 

The frontend uses React, which is a Single Page Application (SPA). This means there is technically only **one** physical HTML file in the entire folder: `index.html`. All other pages (like `/browse`, `/admin`, `/login`) are fake illusions created by Javascript instantly drawing new shapes on the screen. The plain Nginx server hosting the frontend had no idea how to do this, so it threw a 404 when it couldn't find the literal `/admin` directory.

### **The Fix**
Created a new Kubernetes `ConfigMap` (`frontend-nginx-configmap.yaml`) explicitly defining an `nginx.conf` that contains the magic directive: `try_files $uri $uri/ /index.html;`. Then, attached this ConfigMap into the frontend pod via a volume mount.

*What does this do?* This tells the Tour Guide: *"If someone asks for a room that doesn't physically exist, don't scream 404. Just silently blindfold them, walk them back to the main `index.html` room, and let the magical Javascript inside that room draw the new dashboard for them."*

---

## 🛑 Issue 7: Ingress API Route Stripping (The Hotel Receptionist)

### **Symptoms**
The user could successfully initiate the signup/login flow in their browser, but nothing would happen. The backend `auth-service` logs remained completely empty (`kubectl logs`), indicating the network request was vanishing.

### **Root Cause**
> 💡 **The Layman's Analogy:** Imagine your Kubernetes cluster is a massive, high-tech hotel. The **Ingress Controller** (`ingress.yaml`) is the Master Receptionist. You don't just walk straight into the hotel kitchen; you go to the Receptionist, say *"I want to upload a video (/api/admin),"* and she physically points you down the hallway to the "Admin Suite" (the backend admin-service pod). 

The problem was that the `ingress.yaml` was applying a heavy-handed regex rule using `nginx.ingress.kubernetes.io/rewrite-target: /$2` across all traffic. 
When the React app requested `http://localhost/api/auth/login`, our Receptionist (the Ingress controller) was instructed to maliciously chop off the `/api/auth` prefix and forward only `/login` to the Auth Suite. However, the inner Node.js Express router inside the room was specifically waiting for someone to knock exactly on `/api/login`! The mismatched knock caused a silent 404 drop.

### **The Fix**
Split the singular monolithic `ingress.yaml` file into two carefully crafted Receptionists:
1. `streamingapp-ingress` (No rewrite config): Directly points guests to `/api/admin`, `/api/chat`, and `/api/streaming` exactly as they asked, without mutating their requests.
2. `streamingapp-auth-ingress` (Special rewrite config): Securely translates `/api/auth/login` to exactly `/api/login` using `rewrite-target: /api/$2` to satisfy the Node.js backend.

**Commands Used to Debug:**
```bash
# Spy on all literal network requests passing through the Nginx Ingress gateway
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50
```

---

## 🛑 Issue 8: React Localhost Domain Lock (The Pre-Flight Mechanics)

### **Symptoms**
After fixing the route stripping, the App worked elegantly on `localhost`. However, when deployed to production (e.g., via AWS CloudFront), it failed completely. The React UI loaded, but API requests refused to connect.

### **Root Cause**
> 💡 **The Layman's Analogy:** Imagine you bought a brand-new airplane (the Frontend Docker Image), but the manufacturer accidentally hardcoded the GPS coordinates for the *wrong airport* into the plane's unbreakable black box. The main pilot (the Nginx web server) is about to take off on a doomed flight.

When the Docker container for the frontend was built in the CI/CD pipeline, the build scripts hardcoded absolute URLs into the React Javascript (e.g., `http://localhost:3001/api/login`). Because Docker Images are "Read-Only" (locked vaults), when a user opened the site on the internet, the React app literally attempted to connect to `localhost` on the user's personal Mac/PC instead of routing back via the cloud network.

### **The Fix**
Because we wanted to solve this without waiting for a massive pipeline Docker rebuild, we utilized a Kubernetes **`initContainer`** inside `frontend-deployment-service.yaml`.

An `initContainer` is a highly specialized team of mechanics that run onto the runway **before** the main pilot is allowed to start the engine:
1. They copy the minified Javascript bundle to an `emptyDir` memory volume.
2. They use a highly powerful laser (`sed`) to surgically slice out the bad absolute coordinates (`http://localhost:3001/api`) and replace them with extremely clean relative paths (`/api/auth`).
3. The mechanics leave, the `initContainer` permanently dies, and the main pilot (Nginx) boots up perfectly. 
4. Because the paths are now relative, the user's web browser dynamically prepends its current host domain (CloudFront, Localhost, etc.) solving the routing block instantly across all environments!

**Commands Used to Debug & Fix:**
```bash
# Extract the minified Javascript from the active React pod to inspect the baked-in URLs
kubectl exec $(kubectl get pod -l component=frontend -o jsonpath='{.items[0].metadata.name}') -- grep -ro "http://localhost:[0-9]*" /usr/share/nginx/html/
```
---
