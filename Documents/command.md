## List of commands used during this project

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
kubectl port-forward svc/auth-service-service 3001:3001
kubectl port-forward svc/streaming-service 3002:3002
kubectl port-forward svc/admin-service-service 3003:3003
kubectl port-forward svc/chat-service 3004:3004

#### 4. Fetch ECR images from AWS CLI
Authenticate with AWS CLI for AWS ECr
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 796786461592.dkr.ecr.ap-south-1.amazonaws.com

List of Repositories in ECR
aws ecr describe-repositories --region ap-south-1 --query "repositories[].repositoryName" --output text

List of Images in ECR
aws ecr list-images --repository-name streaming --region ap-south-1 --query "imageIds[].imageTag" --output text

Pull an image from ECR
aws ecr get-download-url-for-layer --repository-name streaming --region ap-south-1 --layer-digest sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef