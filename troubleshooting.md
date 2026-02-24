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
