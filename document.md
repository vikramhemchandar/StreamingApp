# The Ultimate Beginner's Guide to Kubernetes Manifests

Welcome to Kubernetes! When you built your project with Docker Compose, you used a single `docker-compose.yml` file to run everything. Kubernetes does the same job but at a massive, robust, enterprise scale. Because of this scale, Kubernetes splits its configuration into **multiple separate files** (called Manifests) to give you fine-grained control over networking, storage, and passwords.

This guide will explain every single piece of the puzzle used to launch your Streaming Application, line by line.

---

## 🏗️ 1. The ConfigMap: Injecting Environment Variables
**File:** `configmap.yml`  
**Purpose:** A ConfigMap is basically a giant `.env` file that lives in Kubernetes. Instead of hardcoding API keys and database URLs inside your Docker images, you store them here. When your backend pods start, they "read" this ConfigMap to know what ports to use and what database to connect to.

```yaml
apiVersion: v1               # 1. The Kubernetes API version used for this resource
kind: ConfigMap              # 2. Telling Kubernetes "I am creating a ConfigMap"
metadata:                    # 3. Data about the ConfigMap itself
  name: configmap            # 4. The name we will use to reference this later
  labels:                    # 5. Tags we can attach to search for it later
    app: configMap
data:                        # 6. The dictionary of actual environment variables!
  AUTH_PORT: "3001"          # 7. We create unique keys for every port so they don't overwrite each other
  STREAMING_PORT: "3002"
  ADMIN_PORT: "3003"
  CHAT_PORT: "3004"
  # Variables shared across multiple backends
  MONGO_URI: "mongodb://database-service:27017/streamingapp" # 8. Notice we use the Kubernetes Service name instead of localhost!
  JWT_SECRET: "changeme"
```

---

## 💽 2. Persistent Storage (PV and PVC)
Docker Compose easily handled your database via `volumes: - mongo-data:/data/db`. Kubernetes requires two files to recreate this because a Kubernetes cluster might have 100 different physical hard drives!

### A. The Persistent Volume (PV)
**File:** `database-persistentvolume.yml`  
**Purpose:** The PV represents a **physical piece of hard drive space** on the actual master/worker node server.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: database-persistentvolume # The name of this disk
spec:                             # The specification of the disk
  capacity:
    storage: 2Gi                  # How big is the disk? 2 Gigabytes.
  accessModes:
    - ReadWriteOnce               # Only one Node can mount this disk at a time
  persistentVolumeReclaimPolicy: Retain # If the Pod dies, DO NOT format the disk! Keep the data!
  storageClassName: manual
  hostPath:                       # Since this is a local cluster, we just use a folder on the host machine
    path: /mnt/data               # The physical path on the server where MP4s/data live
```

### B. The Persistent Volume Claim (PVC)
**File:** `database-persistentvolumeclaim.yml`  
**Purpose:** The PVC is a "request voucher". Your database Pod doesn't talk directly to the PV disk. Instead, the Pod creates a Claim ("I need 2GB of storage!"). Kubernetes checks its inventory of PVs, finds our 2GB disk, and binds them together.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim       # The "request voucher"
metadata:
  name: database-persistant-volume-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:                      # What are we requesting?
    requests:
      storage: 2Gi                # Give me 2GB of space!
```

---

## 🗄️ 3. The Database Deployment & Service
**File:** `database-deployment-service.yml`  
**Purpose:** This file does two things:
1. **Deployment:** Downloads the `mongo` Docker image and runs the container.
2. **Service:** Creates a permanent internal IP address and DNS name (`database-service`) so the Node.js backends can talk to Mongo.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-deployment
spec:                             
  replicas: 1                     # 1. Run exactly 1 copy of the database
  selector:
    matchLabels:
      component: mongo            # 2. How the Deployment finds the Pods it controls
  template:                       # 3. The blueprint for creating the Pod
    metadata:
      labels:
        component: mongo
    spec:
      volumes:                    # 4. We grab the PVC "request voucher" we made earlier
        - name: mongo-storage
          persistentVolumeClaim:
            claimName: database-persistant-volume-claim
      containers:
        - name: mongo             # 5. Name of the container
          image: mongo:latest     # 6. The Docker Hub image to download
          ports:
            - containerPort: 27017 # 7. The internal port Mongo listens on
          volumeMounts:           # 8. We mount our "mongo-storage" to Mongo's internal data folder
            - name: mongo-storage
              mountPath: /data/db
---
# The Service (Think of this as an internal Load Balancer/Router)
apiVersion: v1
kind: Service
metadata:
  name: database-service          # The internal DNS name (e.g. mongodb://database-service:27017)
spec:
  type: ClusterIP                 # Type ClusterIP means this is heavily isolated; it is ONLY accessible from inside the Kubernetes cluster.
  selector:
    component: mongo              # Send traffic only to Pods labeled with 'component: mongo'
  ports:
    - port: 27017                 # The port the Service listens on
      targetPort: 27017           # The port it forwards traffic to inside the container
```

---

## ⚙️ 4. The Backend Deployments & Services
*(Note: You have multiple similar backends: Admin, Auth, Chat, and Streaming. We will use `admin-deployment-service.yml` as the sample because their structure is fundamentally identical!)*

**File:** `admin-deployment-service.yml`  
**Purpose:** Deploys your Node.js/Express Docker image, injects the `configmap.yml` variables into it, checks its health, and creates an internal Service to route traffic to it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-service-deployment
spec:
  replicas: 2                     # 1. Run 2 copies of the Admin backend for High Availability (Scale!)
  selector:
    matchLabels:
      component: admin-service
  template:
    metadata:
      labels:
        component: admin-service
    spec:
      containers:
        - name: admin-service
          image: vikramhemchandar/streamingapp:adminv1.0 # 2. Pulling your custom Docker Image!
          ports:
            - containerPort: 3003
          envFrom:                # 3. Pull ALL environment variables from our generic ConfigMap
            - configMapRef:
                name: configmap
          env:                    # 4. We explicitly map the unique keys we made in the ConfigMap
            - name: PORT          # Standardizes the variable name for Node.js (process.env.PORT)
              valueFrom:
                configMapKeyRef:
                  name: configmap
                  key: ADMIN_PORT # Pulls '3003' instead of pulling another service's port
            - name: MONGO_URI
              valueFrom:
                configMapKeyRef:
                  name: configmap
                  key: MONGO_URI  # Pulls the database connection string
          
          # Health Checks (Very Important!)
          livenessProbe:          # 5. "Is the app frozen/crashed? If so, KILL IT and RESTART!"
            httpGet:
              path: /api/health   # Hits your Express framework router
              port: 3003
            initialDelaySeconds: 5 # Wait 5 seconds on startup before checking
            periodSeconds: 10      # Check every 10 seconds forever
          readinessProbe:         # 6. "Is the app still loading? If so, STOP SENDING USER TRAFFIC to it!"
            httpGet:
              path: /api/health
              port: 3003
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:              # 7. CPU and Memory constraints 
            requests:
              cpu: "100m"         # 100 millicores (0.1 of a CPU)
              memory: "128Mi"     # 128 Megabytes RAM minimum request
            limits:
              cpu: "250m"         # Do not let Node.js use more than 0.25 of a CPU
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: admin-service-service
spec:
  type: ClusterIP                 # Internal access only
  selector:
    component: admin-service
  ports:
    - port: 3000                  # The port other pods use to ping this service
      targetPort: 3000            # Forwards down to the container's running port
```

---

## 🌐 5. The Frontend React Deployment
*(This file follows the exact same logic as the backend deployments above, but handles ports completely differently because of Nginx!)*

**File:** `frontend-deployment-service.yml`  
**What makes it distinct:**
1. **The Web Server Engine:** A production React app doesn't use `npm start` (Node.js). It builds into static HTML/CSS/JS files and is served by an **Nginx** web server container.
2. **The Magic Port 80:** By default, Docker Nginx containers always listen to HTTP web traffic on **Port 80**. 
3. **External Access:** The Service type is typically different (e.g., NodePort or LoadBalancer) to pierce the Kubernetes bubble and allow regular users on the internet to see the website.

```yaml
# Inside frontend-deployment-service.yml

          containers:
            - name: frontend
              image: vikramhemchandar/streamingapp:frontendv1.0
              ports:
                - containerPort: 80 # 1. WE MUST USE 80. Nginx inside the Docker image demands it!

          # ... (Environment variables are injected the same way as the backend) ...

          livenessProbe:
            httpGet:
              path: /               # 2. Nginx doesn't have an /api/health route. We just ask it to load the index page.
              port: 80              # 3. We ping port 80 to see if Nginx is alive
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort                    # 4. Unlike ClusterIP (Internal), NodePort opens a hole to the outside world!
  selector:
    component: frontend
  ports:
    - port: 3000                    # This is the port we hit from the outside user's perspective (localhost:3000)
      targetPort: 80                # Traffic from 3000 is routed down to Nginx's strictly enforced Port 80
```

### Summary of How Traffic Flows in the Frontend
User's Browser `(localhost:3000)` ➡️ `Service named 'frontend' (port: 3000)` ➡️ `Frontend Pod container (targetPort: 80)` ➡️ React App is Served!
