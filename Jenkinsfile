// This pipeline builds a Docker image, pushes it to Docker Hub, 
// and deploys the updated image to a Kubernetes cluster using kubectl.

pipeline {
    // The main pipeline runs on the Jenkins built-in agent
    agent any

    // Define environment variables and credential IDs
    environment {
        // Your Docker Hub repo name
        DOCKER_IMAGE_NAME = 'sonaliponnappaa/cicd-app' 
        
        // Credential IDs configured in Jenkins
        DOCKER_USER_CREDS = 'dockerhub-creds' // ID for Docker Hub username/password
        KUBE_CONFIG_CREDS = 'minikube-config' // ID for Kubernetes secret text
    }

    stages {
        // Stage 1: Checkout the source code
        stage('Declarative: Checkout SCM') {
            steps {
                // Gets the source code from the configured Git repository
                checkout scm
            }
        }

        // Stage 2: Checkout SCM (for reference, kept for pipeline compatibility)
        stage('Checkout SCM') {
            steps {
                // Gets the source code from the configured Git repository
                checkout scm
            }
        }

        // Stage 3: Build the Docker Image
        stage('Build Image') {
            steps {
                script {
                    // Use withCredentials to securely inject Docker username and password (DOCKER_PASS)
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_USER_CREDS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        
                        // Robust login for base image pull during build
                        retry(3) {
                            echo "Attempting robust docker login for base image pull..."
                            sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"
                        }

                        // Build the image using the current build number as the tag
                        echo "Building Docker image: ${DOCKER_IMAGE_NAME}:${BUILD_ID}..."
                        sh "docker build -t ${DOCKER_IMAGE_NAME}:${BUILD_ID} ."
                    }
                }
            }
        }

        // Stage 4: Push the Docker Image (Includes network resilience and retry logic)
        stage('Push Image') {
            steps {
                script {
                    echo "Starting authenticated push for ${DOCKER_IMAGE_NAME}:${BUILD_ID}"
                    
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_USER_CREDS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        
                        // Network stabilization pause (15 seconds proved effective)
                        echo "Network stabilization: Waiting 15 seconds before first push attempt..."
                        sleep 15
                        
                        // Robust login right before push
                        retry(3) {
                            echo "Attempting robust docker login before push..."
                            sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"
                        }
                        
                        // Retry loop for the push operation
                        for (int i = 1; i <= 5; i++) {
                            try {
                                echo "Attempt ${i}/5: Pushing ${DOCKER_IMAGE_NAME}:${BUILD_ID}"
                                sh "docker push ${DOCKER_IMAGE_NAME}:${BUILD_ID}"
                                echo "Push successful on attempt ${i}"
                                
                                // Tag and push 'latest'
                                sh "docker tag ${DOCKER_IMAGE_NAME}:${BUILD_ID} ${DOCKER_IMAGE_NAME}:latest"
                                echo "Attempt ${i}/5: Pushing ${DOCKER_IMAGE_NAME}:latest"
                                sh "docker push ${DOCKER_IMAGE_NAME}:latest"
                                echo "Latest push successful on attempt ${i}"
                                
                                break // Exit loop on success
                            } catch (Exception e) {
                                if (i == 5) {
                                    error("Push failed after ${i} attempts: ${e.getMessage()}")
                                }
                                echo "Push failed on attempt ${i}. Retrying in ${i * 5} seconds..."
                                sleep i * 5
                            }
                        }
                    }
                }
            }
        }

        // Stage 5: Deploy to Kubernetes
        stage('Deploy to K8s') {
            // Run this stage inside the kubectl Docker image
            agent {
                docker {
                    image 'bitnami/kubectl:latest'
                    args '--entrypoint="" -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            
            steps {
                // Setup Kubeconfig file
                sh "mkdir -p .kube"
                withCredentials([string(credentialsId: env.KUBE_CONFIG_CREDS, variable: 'KUBECFG_CONTENT')]) {
                    
                    // 1. Write the secret content directly to .kube/config
                    writeFile(file: ".kube/config", text: "${KUBECFG_CONTENT}", encoding: 'UTF-8')

                    // 2. CRITICAL FIX: Double-escape all backslashes for Groovy/sed compatibility.
                    // s/\\r//g: Removes Windows carriage returns.
                    // /^\\s*$/d: Removes empty lines (where \s* means zero or more whitespace characters).
                    sh '''
                        sed -i 's/\\r//g; s/"//g; /^\\s*$/d; s/^[[:space:]]*//; s/[[:space:]]*$//' .kube/config
                    '''
                    
                    sh "chmod 600 .kube/config"

                    // Set the KUBECONFIG environment variable (runs inside the kubectl container)
                    withEnv(["KUBECONFIG=.kube/config", "IMAGE_URL=${DOCKER_IMAGE_NAME}:${BUILD_ID}"]) {
                        sh "echo Deploying image ${IMAGE_URL} to Kubernetes..."
                        
                        // Update the K8s manifest with the new image tag
                        sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${IMAGE_URL}|g' kubernetes-deployment.yaml"

                        // Apply the deployment (kubectl is now available)
                        sh "kubectl apply -f kubernetes-deployment.yaml"
                        
                        echo "Deployment command executed successfully."
                    }
                }
            }
        }
    }
}
