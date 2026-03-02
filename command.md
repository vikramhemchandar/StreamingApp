Docker push multiple images at once
docker push --all-tags vikramhemchandar/streamingapp

List of commands using during this project
kubectl apply -f configmap.yml
kubectl apply -f database-persistentvolume.yml
kubectl apply -f database-persistentvolumeclaim.yml
kubectl apply -f database-deployment-service.yml
kubectl apply -f auth-deployment-service.yml
kubectl apply -f streaming-deployment-service.yml
kubectl apply -f admin-deployment-service.yml
kubectl apply -f chat-deployment-service.yml
kubectl apply -f frontend-deployment-service.yml

To delete everything
#### 1. Delete Deployments and Pods
kubectl delete deployments --all -n default
kubectl delete pods --all -n default

#### 2. Delete PVCs (This frees up PVs if ReclaimPolicy is 'Delete')
kubectl delete pvc --all -n default

#### 3. Delete PVs (If they are in 'Released' state)
kubectl delete pv --all

Port Forward
kubectl port-forward svc/frontend 3000:3000
kubectl port-forward svc/frontend 3001:3001
kubectl port-forward svc/streaming-service 3002:3002
kubectl port-forward svc/admin-service-service 3003:3003
kubectl port-forward svc/chat-service 3004:3004
