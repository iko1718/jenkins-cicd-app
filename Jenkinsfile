// This pipeline builds a Docker image, pushes it to Docker Hub, 
// and deploys the updated image to a Kubernetes cluster using kubectl.

pipeline {
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
        stage('Checkout SCM') {
            steps {
                // Gets the source code from the configured Git repository
                checkout scm
            }
        }

        // Stage 2: Build the Docker Image
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

        // Stage 3: Push the Docker Image (Includes network resilience and retry logic)
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

        // Stage 4: Deploy to Kubernetes (Fixed Kubeconfig parsing issue)
        stage('Deploy to K8s') {
            // Run on a clean node/workspace (assuming your Jenkins VM has kubectl installed)
            agent {
                node { label 'master' } 
            }
            steps {
                sh "mkdir -p .kube"
                
                // Use withCredentials to inject KUBECFG_CONTENT securely
                withCredentials([string(credentialsId: env.KUBE_CONFIG_CREDS, variable: 'KUBECFG_CONTENT')]) {
                    
                    // 1. Write the content to a temp config file
                    writeFile(file: ".kube/config.temp", text: "${KUBECFG_CONTENT}", encoding: 'UTF-8')

                    // 2. SIMPLIFIED CLEANUP: Remove empty lines and surrounding whitespace. 
                    // This avoids the 'tr -cd' command which damaged the YAML structure.
                    sh "sed -i -e '/^$/d' -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' .kube/config.temp"

                    // 3. Rename the cleaned file and set restrictive permissions
                    sh "mv .kube/config.temp .kube/config"
                    sh "chmod 600 .kube/config"

                    // 4. Set the KUBECONFIG environment variable for kubectl
                    withEnv(["KUBECONFIG=.kube/config", "IMAGE_URL=${DOCKER_IMAGE_NAME}:${BUILD_ID}"]) {
                        sh "echo Deploying image ${IMAGE_URL} to Kubernetes..."
                        
                        // 5. Update the K8s manifest with the new image tag
                        sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${IMAGE_URL}|g' kubernetes-deployment.yaml"

                        // 6. Apply the deployment
                        sh "kubectl apply -f kubernetes-deployment.yaml"
                        
                        echo "Deployment command executed successfully."
                    }
                }
            }
        }
    }
}
