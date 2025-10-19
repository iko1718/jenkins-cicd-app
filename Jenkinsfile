// Jenkinsfile

pipeline {
    // Defines the agent where the pipeline runs (the host VM where Jenkins is running)
    agent {
        docker {
            image 'jenkins/jenkins:lts'
            args '-u root -v /var/run/docker.sock:/var/run/docker.sock' 
        }
    }

    // Environment variables used throughout the pipeline
    environment {
        // The ID of the Docker Hub credentials stored in Jenkins
        DOCKER_HUB_CREDS = 'dockerhub-creds'
        // Your Docker Hub username and image name (e.g., 'yourusername/cicd-app')
        DOCKER_IMAGE_NAME = "your-dockerhub-username/cicd-app"
        // The ID of the Kubeconfig credential stored in Jenkins
        KUBECONFIG_CREDENTIAL_ID = 'minikube-config'
    }

    stages {
        // ===================================
        // Stage 1: Checkout (Pull Code)
        // ===================================
        stage('Checkout Code') {
            steps {
                echo 'Checking out source code from Git...'
                // The 'checkout scm' step uses the Git configuration from the job
                // If the repository is public, this might be enough.
                // If using SSH/Private, Jenkins credentials handle the authentication.
                checkout scm
            }
        }

        // ===================================
        // Stage 2: Build (Create Docker Image)
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
                // Use 'withKubeConfig' to securely access the stored Kubeconfig file
                withKubeConfig([credentialsId: env.KUBECONFIG_CREDENTIAL_ID]) {
                    script {
                        def imageTag = "${env.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                        echo "Deploying image ${imageTag} to Minikube cluster..."

                        // 1. Find the deployment manifest file
                        def deployFile = 'kubernetes-deployment.yaml'

                        // 2. Temporarily replace the placeholder with the actual image tag
                        // This uses sed to find and replace the PLACEHOLDER_IMAGE_URL
                        sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${imageTag}|g' ${deployFile}"
                        
                        // 3. Apply the deployment to Minikube using kubectl
                        // 'kubectl apply' creates/updates the deployment and service
                        sh "kubectl apply -f ${deployFile}"

                        // 4. (Optional) Revert the file change for the next commit/build
                        sh "git checkout ${deployFile}"

                        echo "Deployment complete! Application is now running on Kubernetes."
                    }
                }
            }
        }
    }
}
