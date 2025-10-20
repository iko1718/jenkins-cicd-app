pipeline {
    // Defines the agent where the pipeline runs (the host VM where Jenkins is running)
    agent any

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
                    
                    // 🚨 CRITICAL FIX: Install Docker client before using 'docker build'
                    // This prevents the "docker: not found" error
                    sh 'apt-get update && apt-get install -y docker.io' 
                    
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
                    
                    // Use 'withCredentials' to securely access the stored Docker Hub credentials
                    withCredentials([usernamePassword(
                        credentialsId: env.DOCKER_HUB_CREDS,
                        passwordVariable: 'DOCKER_PASSWORD',
                        usernameVariable: 'DOCKER_USER'
                    )]) {
                        echo "Logging into Docker Hub and pushing image: ${imageTag}"

                        // Secure login using the credentials injected above
                        sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USER --password-stdin"
                        
                        // Push the newly tagged image
                        sh "docker push ${imageTag}"
                        
                        // Push a 'latest' tag for convenience
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
            steps {
                script {
                    def imageTag = "${env.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    
                    // ⚠️ PRELIMINARY CHECK: Install kubectl and sed if they are missing
                    // This prevents errors if the agent doesn't have these essential tools.
                    sh 'apt-get update && apt-get install -y kubectl sed'

                    // Use 'withCredentials' (Secret Text) to get the Kubeconfig content
                    withCredentials([string(
                        credentialsId: env.KUBECONFIG_CREDENTIAL_ID,
                        variable: 'KUBECFG_CONTENT' // This variable will hold the YAML text
                    )]) {
                        echo "Deploying image ${imageTag} to Minikube cluster..."

                        // 1. Write the Secret Text content to a temporary file
                        // The 'echo' is wrapped in 'sh' and the variable quoted to preserve multi-line content
                        sh "echo \"\${KUBECFG_CONTENT}\" > ${env.KUBECONFIG_FILE}"

                        // 2. Temporarily replace the placeholder with the actual image tag
                        sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${imageTag}|g' ${env.K8S_DEPLOYMENT_FILE}"
                        
                        // 3. Apply the deployment to Minikube using kubectl
                        // CRITICAL: We use '--kubeconfig=' to point to the temporary file we just created.
                        sh "kubectl --kubeconfig=${env.KUBECONFIG_FILE} apply -f ${env.K8S_DEPLOYMENT_FILE}"

                        // 4. Revert the file change for the next commit/build (clean workspace)
                        sh "git checkout ${env.K8S_DEPLOYMENT_FILE}"
                        
                        // 5. Cleanup: Delete the temporary Kubeconfig file
                        sh "rm ${env.KUBECONFIG_FILE}"

                        echo "Deployment complete! Application is now running on Kubernetes."
                    }
                }
            }
        }
    }
}
