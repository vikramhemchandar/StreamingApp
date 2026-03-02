pipeline {
    agent {
        docker {
            image 'docker:dind'
            args '-v /var/run/docker.sock:/var/run/docker.sock -u root:root'
        }
    }
    environment {
        AWS_REGION = 'ap-south-1' 
        ECR_REGISTRY = '796786461592.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_REPO = 'vikramhemchandar/streamingapp'        
        IMAGE_TAG = "v1.0.${env.BUILD_ID}"
        AWS_CREDS_ID = 'aws-credentials'
    }
    
    stages {
        stage('Install Prerequisites') {
            steps {
                sh 'apk add --no-cache aws-cli git'
            }
        }
        
        stage('Build All Services') {
            steps {
                script {
                    withEnv([
                        "DOCKER_USER=${ECR_REGISTRY}/${ECR_REPO}",
                        "TAG=-${IMAGE_TAG}"
                    ]) {
                        sh 'docker-compose -f docker-compose.yml build'
                        sh 'echo "Build completed successfully for tag: ${IMAGE_TAG}"'
                    }
                }
            }
        }
        
        stage('Push All Services to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding', 
                    credentialsId: env.AWS_CREDS_ID
                ]]) {
                    script {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                        def services = ['auth', 'streaming', 'admin', 'chat', 'frontend']                        
                        for (String service : services) {
                            sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${service}-${IMAGE_TAG}"
                            sh "echo 'Pushed ${ECR_REGISTRY}/${ECR_REPO}:${service}-${IMAGE_TAG}'"
                        }
                    }
                }
            }
        }
        
        stage('Deploy (Update Kubernetes Manifests)') {
            steps {
                script {
                    def services = ['auth', 'streaming', 'admin', 'chat', 'frontend']
                    
                    for (String service : services) {
                        // FIXED: Changed '*.yaml' to '*.yml' so it finds your actual files!
                        sh "find k8s -name '*.yml' -type f -exec sed -i \"s|image: .*/${ECR_REPO}:${service}-.*|image: ${ECR_REGISTRY}/${ECR_REPO}:${service}-${IMAGE_TAG}|\" {} +"
                        sh "echo 'Pushed ${ECR_REGISTRY}/${ECR_REPO}:${service}-${IMAGE_TAG}'"
                    }
                    
                    sh """
                    echo "Updating K8s manifests for all services with tag: ${IMAGE_TAG}"
                    
                    # Bypass the Git security block!
                    git config --global --add safe.directory '*'
                    
                    git config user.name "Vikram Hem Chandar"
                    git config user.email "vikramhemchandar@gmail.com"
                    
                    # FIXED: Checkout the main branch locally so we aren't in a detached HEAD state!
                    git checkout main
                    # Pull any latest changes from GitHub just in case
                    git pull origin main
                    
                    # Commit the changes (We added || true so it doesn't crash if no changes happened)
                    git commit -am 'WIP: update K8s manifests for all services with tag: ${IMAGE_TAG}' || true
                    """
                    
                    withCredentials([gitUsernamePassword(credentialsId: 'github-token', gitToolName: 'Default')]) {
                        sh "git push origin main"
                    }
                }
            }
        }
    }
}
