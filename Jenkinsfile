pipeline {
    // 💡 NEW FIX: Use a Docker image that already contains the 'docker' client.
    // This runs the entire pipeline inside a separate Docker container using the official client image.
    agent {
        docker {
            image 'docker:latest'
            // Ensure the Docker daemon socket is mounted for the inner container to work
            args '-v /var/run/docker.sock:/var/run/docker.sock' 
        }
    }

    // Environment variables used throughout the pipeline
    environment {
        // The ID of the Docker Hub credentials stored in Jenkins (Username with password)
        DOCKER_HUB_CREDS = 'dockerhub-creds'
        // Your Docker Hub username and image name (e.g., 'yourusername/cicd-app')
        DOCKER_IMAGE_NAME = "sonaliponnappaa/cicd-app"
        // The ID of the Kubeconfig credential stored in Jenkins (Secret Text)
        KUBECONFIG_CREDENTIAL_ID = 'minikube-config'
        // Define a name for the temporary file that will hold the Kubeconfig content
        KUBECONFIG_FILE = 'kubeconfig-temp.yaml'
        // The file containing your Kubernetes Deployment YAML
        K8S_DEPLOYMENT_FILE = 'kubernetes-deployment.yaml' 
    }

    stages {
        
        // ===================================
        // Stage 1: Checkout
        // ===================================
        stage('Checkout SCM') {
            steps {
                // Checkout must be outside the 'script' block for declarative
                checkout scm 
            }
        }
        
        // ===================================
        // Stage 2: Build (Create Docker Image)
        // ===================================
        stage('Build Image') {
            steps {
                script {
                    def imageTag = "${env.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    echo "Building Docker image: ${imageTag}"
                    
                    // ❌ NO LONGER NEEDED: The 'docker:latest' image already has the client.
                    // sh 'apt-get update && apt-get install -y docker.io' 
                    
                    // We use the sh step to execute Docker commands on the host VM
                    sh "docker build -t ${imageTag} ."
                }
            }
        }

        // ===================================
        // Stage 3: Push (Upload Image to Registry)
        // ===================================
        stage('Push Image') {
            steps {
                script {
                    def imageTag = "${env.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    
                    withCredentials([usernamePassword(
                        credentialsId: env.DOCKER_HUB_CREDS,
                        passwordVariable: 'DOCKER_PASSWORD',
                        usernameVariable: 'DOCKER_USER'
                    )]) {
                        echo "Logging into Docker Hub and pushing image: ${imageTag}"

                        // Secure login
                        sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USER --password-stdin"
                        
                        // Push tags
                        sh "docker push ${imageTag}"
                        sh "docker tag ${imageTag} ${env.DOCKER_IMAGE_NAME}:latest"
                        sh "docker push ${env.DOCKER_IMAGE_NAME}:latest"

                        // Logout for security
                        sh "docker logout"
                    }
                }
            }
        }

        // ===================================
        // Stage 4: Deploy (to Kubernetes)
        // ===================================
        stage('Deploy to K8s') {
            // ⚠️ TEMPORARY OVERRIDE: Deploy stage needs a different agent (one with kubectl/sed)
            // For the fastest fix, we will skip this stage temporarily unless you are sure
            // the 'docker:latest' image has kubectl and sed installed (it may not).
            // For now, let's focus on Build and Push.
            agent any // Switch back to the main Jenkins agent for this, if needed.

            steps {
                script {
                    def imageTag = "${env.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    
                    // ⚠️ Install Kubectl and sed since the 'agent any' doesn't guarantee them.
                    sh 'apt-get update && apt-get install -y kubectl sed'

                    withCredentials([string(
                        credentialsId: env.KUBECONFIG_CREDENTIAL_ID,
                        variable: 'KUBECFG_CONTENT'
                    )]) {
                        echo "Deploying image ${imageTag} to Minikube cluster..."

                        // 1. Write the Secret Text content to a temporary file
                        sh "echo \"\${KUBECFG_CONTENT}\" > ${env.KUBECONFIG_FILE}"

                        // 2. Temporarily replace the placeholder
                        sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${imageTag}|g' ${env.K8S_DEPLOYMENT_FILE}"
                        
                        // 3. Apply the deployment
                        sh "kubectl --kubeconfig=${env.KUBECONFIG_FILE} apply -f ${env.K8S_DEPLOYMENT_FILE}"

                        // 4. Revert and cleanup
                        sh "git checkout ${env.K8S_DEPLOYMENT_FILE}"
                        sh "rm ${env.KUBECONFIG_FILE}"

                        echo "Deployment complete! Application is now running on Kubernetes."
                    }
                }
            }
        }
    }
}
