# 🛠️ Kubernetes Command Cheat Sheet

This document contains a comprehensive list of all the `kubectl` commands used to deploy, monitor, and troubleshoot the Streaming Application on your local Kubernetes cluster. It also includes an "Important Commands" section at the end for future reference.

---

## 🚀 1. Deployment and Creation Commands

These are the commands used to tell Kubernetes to consume your YAML files and create the actual resources (Pods, Services, PVCs, ConfigMaps) inside the cluster.

* **`kubectl apply -f [filename.yml]`**
  * *What we used it for:* Bootstrapping the entire application.
  * *Example:* `kubectl apply -f configmap.yml` creates or updates the ConfigMap.
  * *Pro Tip:* You can pass multiple files at once! We used `kubectl apply -f configmap.yml -f database-deployment-service.yml -f auth-deployment-service.yml ...` to spin up the entire backend in one go.

---

## 👀 2. Monitoring & Inspection Commands

Once you tell Kubernetes to create resources, you need a way to look at them to make sure they are actually running.

* **`kubectl get pods`**
  * *What it does:* Lists every single Pod in the default namespace, showing you if they are `READY` (e.g., `1/1`), or if they are crashing (`CrashLoopBackOff`).
  
* **`kubectl get pods -w`**
  * *What it does:* The `-w` stands for **watch**. Instead of running the command once and quitting, it keeps the terminal open and prints a new line the exact millisecond a Pod changes state (e.g., transitioning from `ContainerCreating` to `Running`). We used this heavily to wait for the new Pods to come up.

* **`kubectl get pods -o wide`**
  * *What it does:* Shows the standard `get pods` information, but adds extra "wide" columns, crucially showing you the internal **IP Address** of the Pod and the Node it is running on.

* **`kubectl get deployments`**
  * *What it does:* Lists your Deployments (the managers of the Pods) to see if they have successfully scaled up to the `REPLICAS: 2` limit we set in the YAML files.

---

## 🕵️ 3. Troubleshooting & Debugging Commands

When `kubectl get pods` shows `CrashLoopBackOff`, `0/1 READY`, or `CreateContainerConfigError`, these are the commands we used to find the root cause.

* **`kubectl logs [pod-name]`**
  * *What we used it for:* We used this to look at the exact `console.log()` output coming from your Express.js and React containers. This is how we proved the apps were successfully starting on port 3004, but failing later.
  * *Example:* `kubectl logs admin-service-deployment-58b564bf98-kqnzn`

* **`kubectl logs deploy/[deployment-name]`**
  * *What it does:* If you have two replicas of the Admin service, fetching the logs for a specific pod name can be annoying because the names change randomly when they restart. This command fetches logs for *any* pod managed by that Deployment.
  * *Example:* `kubectl logs deploy/admin-service-deployment`

* **`kubectl get events --sort-by='.metadata.creationTimestamp'`**
  * *What we used it for:* This was the **silver bullet** that solved the problem! If an application crashes *before* it can write a log, or if Kubernetes intentionally kills the Pod (like a failed health check probe), the logs will be empty. The `events` log shows what Kubernetes itself is thinking. This is where we saw the `404 Not Found` errors regarding your health probes.

---

## 🗑️ 4. Deletion and Cleanup Commands

When an environment gets messy or a ConfigMap gets poisoned, sometimes the fastest way to fix it is to nuke the Pods and let the Deployments regenerate fresh ones.

* **`kubectl delete pods --all`**
  * *What we used it for:* After fixing the ConfigMap and health probe paths, we used this to forcefully kill the old corrupted Pods. Because Deployments are designed to keep the app alive, they immediately noticed the Pods were gone and spun up fresh, fixed replacements within 10 seconds.

* **`kubectl delete deployments --all`**
  * *What it does:* Completely destroys the application instances. They will not rebuild themselves unless you run `kubectl apply` again.

* **`kubectl delete pvc --all`**
  * *What it does:* Deletes the Persistent Volume Claims (your storage request vouchers).

* **`kubectl delete pv --all`**
  * *What it does:* Deletes the actual underlying Persistent Volume physical disks. Only do this if you want to wipe the MongoDB database clean permanently!

---

## 🔗 5. Networking Commands

* **`kubectl port-forward svc/[service-name] [local-port]:[cluster-port]`**
  * *What we used it for:* Kubernetes runs in an isolated virtual network bubble. When you tested the UI, you used this command to poke a temporary hole from your laptop's browser connection directly into the `frontend` Service running inside Kubernetes.
  * *Example:* `kubectl port-forward svc/frontend 3000:3000`

---

## ⭐ Important Future Commands (Cheat Sheet)
These are commands we didn't explicitly use today, but you will **absolutely** need as you become a master:

* **`kubectl describe pod [pod-name]`**
  * Extremely detailed output about a single pod. It shows you the injected environment variables, the volume mount statuses, the exact Docker image it pulled, and events specific ONLY to that pod.

* **`kubectl exec -it [pod-name] -- sh`** (or `bash`)
  * Just like `docker exec`! This opens an interactive terminal *inside* the running Kubernetes Pod container. Amazing for checking if files exist inside the container or using `curl` to test internal networking. 

* **`kubectl config current-context`**
  * Essential if you start working with multiple clusters (e.g., Local Desktop, AWS Staging, AWS Production). This command prevents you from accidentally deleting Production when you thought you were working Locally!

* **`kompose convert -f docker-compose.yml`**
  * The command line tool that reads a `docker-compose.yml` file and automatically translates it and generates Kubernetes YAML files for you! Great for getting a starting point for a new project.
