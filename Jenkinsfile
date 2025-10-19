pipeline {
    // Defines the agent where the pipeline runs (the host VM where Jenkins is running)
    agent any

    // Environment variables used throughout the pipeline
    environment {
        // The ID of the Docker Hub credentials stored in Jenkins
        DOCKER_HUB_CREDS = 'dockerhub-creds'
        // Your Docker Hub username and image name (e.g., 'yourusername/cicd-app')
        DOCKER_IMAGE_NAME = "sonaliponnappaa/cicd-app"
        // The ID of the Kubeconfig credential stored in Jenkins
        // NOTE: This MUST now be a Secret Text credential in Jenkins!
        KUBECONFIG_CREDENTIAL_ID = 'minikube-config'
        // Define a name for the temporary file that will hold the Kubeconfig content
        KUBECONFIG_FILE = 'kubeconfig-temp.yaml'
    }

    stages {
      
        // ===================================
        // Stage 1: Build (Create Docker Image)
        // ===================================
        stage('Build Image') {
            steps {
                script {
                    // Tag the image using the current build number for versioning
                    def imageTag = "${env.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    echo "Building Docker image: ${imageTag}"

                    // We use the sh step to execute Docker commands on the host VM
                    // The DOCKER_IMAGE_NAME is defined in the environment block
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
                        
                        // Optional: Push a 'latest' tag for convenience
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
                    def deployFile = 'kubernetes-deployment.yaml'

                    // Use 'withCredentials' (Secret Text) to get the Kubeconfig content
                    withCredentials([string(
                        credentialsId: env.KUBECONFIG_CREDENTIAL_ID,
                        variable: 'KUBECFG_CONTENT' // This variable will hold the YAML text
                    )]) {
                        echo "Deploying image ${imageTag} to Minikube cluster..."

                        // 1. Write the Secret Text content to a temporary file
                        // The 'echo' must be wrapped in 'sh' and the variable quoted to preserve multi-line content
                        sh "echo \"\${KUBECFG_CONTENT}\" > ${env.KUBECONFIG_FILE}"
                        sh "echo 'Kubeconfig file created at: ${env.KUBECONFIG_FILE}'"

                        // 2. Temporarily replace the placeholder with the actual image tag
                        // This uses sed to find and replace the PLACEHOLDER_IMAGE_URL
                        sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${imageTag}|g' ${deployFile}"
                        
                        // 3. Apply the deployment to Minikube using kubectl
                        // CRITICAL: We use '--kubeconfig=' to point to the temporary file we just created.
                        sh "kubectl --kubeconfig=${env.KUBECONFIG_FILE} apply -f ${deployFile}"

                        // 4. Revert the file change for the next commit/build
                        sh "git checkout ${deployFile}"
                        
                        // 5. Cleanup: Delete the temporary Kubeconfig file
                        sh "rm ${env.KUBECONFIG_FILE}"

                        echo "Deployment complete! Application is now running on Kubernetes."
                    }
                }
            }
        }
    }
}
