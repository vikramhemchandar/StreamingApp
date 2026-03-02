pipeline {
    agent {
        docker {
            // Using a Docker-in-Docker image so we have docker and docker-compose
            image 'docker:dind'
            // Mount the host's docker socket so this container can run builds
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        AWS_REGION = 'us-east-1'
        // Replace with your actual ECR Registry URI
        ECR_REGISTRY = '123456789012.dkr.ecr.us-east-1.amazonaws.com'
        
        // This is the incremental tag. env.BUILD_ID increments by 1 automatically every Jenkins run.
        IMAGE_TAG = "v1.0.${env.BUILD_ID}"
        
        // This is the ID of the AWS credentials saved in Jenkins Credentials Manager
        AWS_CREDS_ID = 'aws-credentials'
    }
    stages {
        stage('Install Prerequisites') {
            steps {
                // The docker:dind image is alpine-based, so we use 'apk' to install aws-cli and git
                sh 'apk add --no-cache aws-cli git'
            }
        }
        
        stage('Build Image') {
            steps {
                // This builds the images defined in your docker-compose.yml
                sh 'docker-compose -f docker-compose.yml build'
            }
        }
        
        stage('Push Image to ECR') {
            steps {
                // Securely load AWS credentials from Jenkins
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding', 
                    credentialsId: env.AWS_CREDS_ID
                ]]) {
                    script {
                        // 1. Authenticate Docker with AWS ECR
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                        
                        // 2. Tag the locally built image with the new incremental tag
                        // NOTE: Replace 'my_app_service' with the actual service name from your docker-compose.yml
                        sh "docker tag my_app_service:latest ${ECR_REGISTRY}/my_app_service:${IMAGE_TAG}"
                        
                        // 3. Push the newly tagged image to ECR
                        sh "docker push ${ECR_REGISTRY}/my_app_service:${IMAGE_TAG}"
                    }
                }
            }
        }
        
        stage('Deploy (Update Manifests for Argo CD)') {
            steps {
                script {
                    // 1. Update the image tag in ALL deployment manifest files inside the k8s folder
                    // This finds any line starting with 'image:' and replaces it with the new ECR image URL
                    sh "find k8s -name '*.yaml' -type f -exec sed -i 's|image:.*$|image: ${ECR_REGISTRY}/my_app_service:${IMAGE_TAG}|' {} +"
                    
                    // 2. Configure Git so Jenkins can make a commit
                    sh "git config user.name 'Jenkins CI'"
                    sh "git config user.email 'jenkins@yourcompany.com'"
                    
                    // 3. Stage the changed YAML files and commit them
                    sh "git commit -am 'Update K8s manifests with new ECR image tag: ${IMAGE_TAG}'"
                    
                    // 4. Securely load GitHub credentials and push the commit to trigger Argo CD
                    withCredentials([gitUsernamePassword(credentialsId: 'github-token', gitToolName: 'Default')]) {
                        sh "git push origin main"
                    }
                }
            }
        }
    }
}
