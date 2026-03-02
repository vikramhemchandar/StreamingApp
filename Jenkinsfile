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
        ECR_REPO = 'vikramhemchandar-streamingapp'        
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
                        "TAG=${IMAGE_TAG}"
                    ]) {
                        sh 'docker-compose -f docker-compose.yml build'
                        sh 'echo "Build completed successfully for tag: ${IMAGE_TAG}"'
                        sh 'docker images'
                        sh 'echo "DOCKER_USER: ${DOCKER_USER}"'
                        sh 'echo "TAG: ${TAG}"'
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
                            sh "echo 'Pushing ${ECR_REGISTRY}/${service}:${IMAGE_TAG}'"
                            sh "docker push ${ECR_REGISTRY}/${service}:${IMAGE_TAG}"
                            sh "echo 'Pushed ${ECR_REGISTRY}/${service}:${IMAGE_TAG}'"
                        }
                    }
                }
            }
        }
        
        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "StreamingApp"
                GIT_USER_NAME = "vikramhemchandar"
            }
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    def services = ['auth', 'streaming', 'admin', 'chat', 'frontend']                        
                    for (String service : services) {
                        sh "sed -i \"s|image: .*|image: ${ECR_REGISTRY}/${service}:${IMAGE_TAG}|g\" k8s/${service}-deployment-service.yml"
                    }
                    sh '''
                        # Bypass the Git security block first
                        git config --global --add safe.directory '*'
                        
                        git config user.email "vikramhemchandar@gmail.com"
                        git config user.name "Vikram Hem Chandar"

                        git add .
                        # Added '|| true' so the pipeline doesn't crash if the file is already up to date!
                        git commit -m "Update deployment image to version ${IMAGE_TAG}" || true
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                    '''
                }   
            }
        }  
    }
}
