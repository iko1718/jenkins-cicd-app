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
        // CRITICAL CHANGE: This must now be the ID of a "Secret file" credential
        KUBE_CONFIG_CREDS = 'kubeconfig' 
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

        // Stage 4: Deploy to Kubernetes
        stage('Deploy to K8s') {
            // Run this stage inside the kubectl Docker image
            agent {
                docker {
                    image 'bitnami/kubectl:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            
            steps {
                script {
                    // FIX: Use withCredentials([file(...)]). This binds the Secret File 
                    // to the KUBECONFIG environment variable, ensuring a clean, valid file.
                    withCredentials([file(credentialsId: env.KUBE_CONFIG_CREDS, variable: 'KUBECONFIG')]) {
                        
                        // The KUBECONFIG environment variable now points to the temporary, correct Kubeconfig file.
                        sh '''
                            echo "Deploying image ${DOCKER_IMAGE_NAME}:${BUILD_ID} to Kubernetes..."
                            
                            # 1. Update the image URL in the manifest
                            sed -i "s|PLACEHOLDER_IMAGE_URL|${DOCKER_IMAGE_NAME}:${BUILD_ID}|g" kubernetes-deployment.yaml

                            # 2. Apply the deployment. kubectl automatically uses the KUBECONFIG env var.
                            kubectl apply -f kubernetes-deployment.yaml
                            
                            echo "Deployment command executed successfully."
                        '''
                    }
                }
            }
        }
    }
}
