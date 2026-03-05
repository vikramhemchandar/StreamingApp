# 🚢 End-to-End Modern CI/CD Architecture Flow

This document details the complete Continuous Integration and Continuous Deployment (CI/CD) lifecycle of deploying microservices (e.g., `auth-service`) from a developer's local machine all the way to a production Amazon Kubernetes cluster (EKS) with **zero-downtime**.

The core technologies driving this flow are **GitHub** (Source Control), **Jenkins** (Automation Server), **Docker** (Containerization), **Amazon ECR** (Image Registry), and **Kubernetes** (Orchestration).

---

## 🗺️ High-Level Architecture Flow

This sequence diagram illustrates exactly what happens the moment a developer pushes new code.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant Git as GitHub
    participant Jnk as Jenkins EC2
    participant ECR as Amazon ECR
    participant EKS as Kubernetes (EKS)

    Dev->>Git: git push origin main
    Git->>Jnk: Trigger Webhook (HTTP POST)
    Jnk->>Git: Clone repository code
    Jnk->>Jnk: docker build & tag image
    
    note over Jnk,ECR: Authentication: AWS IAM Instance Profile
    Jnk->>ECR: docker push (Upload new image)
    
    note over Jnk,EKS: Authentication: Kubeconfig / Service Account
    Jnk->>EKS: helm upgrade streamingapp (Inject new tag)
    
    EKS->>ECR: Pulls newly built image
    EKS->>EKS: Performs Rolling Update (Swap Old Pods for New Pods)
```

---

## 🛠️ Phase-by-Phase Breakdown

### Phase 1: Source Control & Trigger (GitHub)
**The Goal:** Automatically listen for code updates.

* **What Happens:** When a developer pushes a new feature to the `main` branch, GitHub immediately fires an HTTP POST payload (a Webhook) to the public IP/Domain of your Jenkins server.
* **Commands / Code (Developer):**
  ```bash
  git add .
  git commit -m "feat: added new auth endpoint"
  git push origin main
  ```
* **Required Access / Setup:**
  * **GitHub Webhook:** Configured in repository settings (e.g., `http://your-jenkins-ip:8080/github-webhook/`).
  * **Jenkins:** The "GitHub Integration Plugin" must be installed and configured on the Jenkins Server.

---

### Phase 2: Continuous Integration (Jenkins Build Workspace)
**The Goal:** Receive the code, test it, and build it into a portable, runnable Docker container.

* **What Happens:** Jenkins wakes up upon receiving the Webhook. It looks at the project's `Jenkinsfile`, checks out the new code, and runs the `docker build` command. It dynamically tags the image with the specific Jenkins Build Number (e.g., `BUILD_ID=42`) rather than `:latest` to ensure every release is uniquely traceable.
* **Jenkinsfile Snippet (Pipeline Code):**
  ```groovy
  stage('Build Docker Image') {
      steps {
          script {
              // Creating a unique tag using Jenkins built-in env.BUILD_ID
              def IMAGE_TAG = "streamingapp/auth-service:build-${env.BUILD_ID}"
              sh "docker build -t ${IMAGE_TAG} ."
          }
      }
  }
  ```
* **Required Access / Setup:**
  * **Jenkins Server:** Must have Docker installed and the `jenkins` user added to the `docker` group.
  * **GitHub Credentials:** Jenkins needs SSH keys or a Personal Access Token to safely clone private repositories.

---

### Phase 3: The Artifact Repository (Amazon ECR)
**The Goal:** Upload the newly built, unique Docker image to a private, secure cloud vault.

* **What Happens:** Jenkins authenticates directly with Amazon Web Services. It re-tags the image to include the AWS routing URL, and pushes the image layers upward to Elastic Container Registry.
* **Jenkinsfile Snippet (Pipeline Code):**
  ```groovy
  stage('Push to ECR') {
      steps {
          script {
              def AWS_ACCOUNT = "975050024946"
              def REGION = "ap-south-1"
              def ECR_URL = "${AWS_ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com"
              def LOCAL_TAG = "streamingapp/auth-service:build-${env.BUILD_ID}"
              def AWS_TAG = "${ECR_URL}/streamingapp/auth-service:build-${env.BUILD_ID}"

              // 1. Authenticate Docker with AWS
              sh "aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_URL}"
              
              // 2. Add the AWS tracking tag
              sh "docker tag ${LOCAL_TAG} ${AWS_TAG}"
              
              // 3. Upload to the cloud
              sh "docker push ${AWS_TAG}"
          }
      }
  }
  ```
* **Required Access / Setup:**
  * **AWS IAM Role:** This is critical for security. You do NOT want to store hardcoded AWS keys (`AWS_ACCESS_KEY_ID`) inside Jenkins. Instead, attach an **IAM Instance Profile** (AmazonEC2ContainerRegistryPowerUser) directly to the Jenkins EC2 instance. Jenkins will automatically inherit these permissions seamlessly.

---

### Phase 4: Continuous Deployment (GitOps via Helm Values)
**The Goal:** Tell the production Kubernetes cluster that a new update is ready, and cleanly swap the users over to the new version using an automated GitOps package manager.

* **What Happens:** Jenkins must tell Kubernetes to deploy the new `build` image. Instead of deploying directly or parsing `.yml` files, it leverages **GitOps**. It securely clones the repository, uses `sed` to update the central Master Configuration file (`helm/streamingapp/values.yaml`), and explicitly commits the updated tag back to GitHub (`main` branch). A Kubernetes CD listener (like ArgoCD or Flux) detects this repository update and orchestrates a "Rolling Update".
* **Jenkinsfile Snippet (Pipeline Code):**
  ```groovy
        stage('Update Deploymen Files - Helm Values (GitOps)') {
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
  ```
* **Required Access / Setup:**
  * **GitHub Token:** Jenkins needs a highly-privileged Personal Access Token to commit source code back to the `main` branch.
  * **CD Operator:** A GitOps mechanism (like ArgoCD) or another isolated Jenkins job must be watching the repository to run `helm upgrade`.

---

### 🎉 The Final Result
Within approximately 60 seconds of a developer pushing code to GitHub, the new feature is cleanly deployed, thoroughly tested, securely stored in AWS, and running live in the Kubernetes cluster with **zero human intervention, and zero application downtime.**
