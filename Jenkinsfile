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
                        "DOCKER_REGISTRY=${ECR_REGISTRY}",
                        "TAG=${IMAGE_TAG}"
                    ]) {
                        sh 'docker-compose -f docker-compose.yml build'
                        sh 'echo "Build completed successfully for tag: ${IMAGE_TAG}"'
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
        
        stage('Update Deployment Files - Helm Values (GitOps)') {
            environment {
                GIT_REPO_NAME = "StreamingApp"
                GIT_USER_NAME = "vikramhemchandar"
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                        sh """
                            # Bypass the Git security block first
                            git config --global --add safe.directory '*'
                            
                            git config user.email "vikramhemchandar@gmail.com"
                            git config user.name "Vikram Hem Chandar"
                            
                            git checkout main
                            git pull origin main
                        """

                        def services = ['auth', 'streaming', 'admin', 'chat', 'frontend']                        
                        for (String service : services) {
                            // Uses Alpine sed to find the specific microservice block and safely replace its trailing tag
                            sh "sed -i \"/^ *${service}:/,/^ *tag:/ s/tag:.*/tag: ${IMAGE_TAG}/\" helm/streamingapp/values.yaml"
                        }
                        
                        sh """
                            git add .
                            # Added '|| true' so the pipeline doesn't crash if the file is already up to date!
                            git commit -m "Update deployment image to version ${IMAGE_TAG}" || true
                            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                        """
                    }   
                }   
            }
        }
        
        // Note: This has not been implemented but kept the script for future use.
        // Do not use this stage as it is not working.
        // Comment the below script before running the Jekinsfile.
        // stage('Deploy to EKS via Helm (Direct CD)') {
        //     environment {
        //         // Ensure Jenkins has the path to Helm and AWSCLI
        //         EKS_CLUSTER_NAME = "your-eks-cluster-name" // Replace with your actual EKS cluster name
        //     }
        //     steps {
        //         script {
        //             // Requires AWS credentials installed in Jenkins
        //             withCredentials([[
        //                 $class: 'AmazonWebServicesCredentialsBinding', 
        //                 credentialsId: env.AWS_CREDS_ID
        //             ]]) {
        //                 sh """
        //                     # 1. Update Kubeconfig so Jenkins can talk to the EKS Cluster
        //                     aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
                            
        //                     # 2. Deploy or Upgrade the Helm Chart directly into the cluster
        //                     helm upgrade --install streamingapp helm/streamingapp \\
        //                       --set images.auth.tag=${IMAGE_TAG} \\
        //                       --set images.streaming.tag=${IMAGE_TAG} \\
        //                       --set images.admin.tag=${IMAGE_TAG} \\
        //                       --set images.chat.tag=${IMAGE_TAG} \\
        //                       --set images.frontend.tag=${IMAGE_TAG}
        //                 """
        //             }
        //         }
        //     }
        // }
    }
}
