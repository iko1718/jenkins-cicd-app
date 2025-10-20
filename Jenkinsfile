pipeline {
    agent any

    // Define environment variables used throughout the pipeline
    environment {
        DOCKER_IMAGE = 'sonaliponnappaa/cicd-app'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                // Checkout the repository code
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                script {
                    def imageTag = env.BUILD_NUMBER
                    
                    // Use credentials to perform manual shell login/build, bypassing flaky Groovy wrappers
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        // 1. Robustly log in for base image pulls (with retry for 503/504 errors)
                        retry(3) {
                            echo "Attempting robust docker login for base image pull..."
                            sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        }
                        
                        echo "Building Docker image: ${DOCKER_IMAGE}:${imageTag} using shell command..."
                        // 2. Perform manual shell build
                        sh "docker build -t ${DOCKER_IMAGE}:${imageTag} ."
                    }
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    def imageTag = env.BUILD_NUMBER
                    
                    echo "Starting authenticated shell push for ${DOCKER_IMAGE}:${imageTag}"
                    
                    // Add an initial stabilization delay to combat persistent 504s
                    echo "Network stabilization: Waiting 15 seconds before first push attempt..."
                    sleep 15
                    
                    // Use withCredentials to securely inject username/password variables for raw shell push.
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        // 1. Robustly log in again just before push (with retry)
                        retry(3) {
                            echo "Attempting robust docker login before push..."
                            sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        }
                        
                        // Define sandbox-safe delays for retries (in seconds)
                        def delays = [0, 20, 40, 80, 160] 
                        def maxRetries = 5
                        def retryCount = 0
                        def pushSuccess = false
                        
                        // --- Push Versioned Tag with Exponential Backoff ---
                        while (!pushSuccess && retryCount < maxRetries) {
                            try {
                                retryCount++
                                echo "Attempt ${retryCount}/${maxRetries}: Pushing ${DOCKER_IMAGE}:${imageTag}"
                                
                                // Apply sandbox-safe exponential delay from the predefined list
                                if (retryCount > 1) {
                                    def delaySeconds = delays[retryCount - 1]
                                    echo "Waiting ${delaySeconds} seconds before retry..."
                                    sleep time: delaySeconds, unit: 'SECONDS'
                                }
                                
                                sh "docker push ${DOCKER_IMAGE}:${imageTag}"
                                pushSuccess = true
                                echo "Push successful on attempt ${retryCount}"
                                
                            } catch (Exception e) {
                                echo "Push attempt ${retryCount} failed: ${e.getMessage()}"
                                if (retryCount >= maxRetries) {
                                    error("Failed to push image after ${maxRetries} attempts")
                                }
                            }
                        }
                        
                        // --- Tag and push latest only if versioned push succeeded ---
                        if (pushSuccess) {
                            sh "docker tag ${DOCKER_IMAGE}:${imageTag} ${DOCKER_IMAGE}:latest"
                            
                            // Push latest with same retry logic (resetting counters)
                            def latestSuccess = false
                            retryCount = 0
                            
                            while (!latestSuccess && retryCount < maxRetries) {
                                try {
                                    retryCount++
                                    echo "Attempt ${retryCount}/${maxRetries}: Pushing ${DOCKER_IMAGE}:latest"
                                    
                                    if (retryCount > 1) {
                                        def delaySeconds = delays[retryCount - 1]
                                        echo "Waiting ${delaySeconds} seconds before retry..."
                                        sleep time: delaySeconds, unit: 'SECONDS'
                                    }
                                    
                                    sh "docker push ${DOCKER_IMAGE}:latest"
                                    latestSuccess = true
                                    echo "Latest push successful on attempt ${retryCount}"
                                    
                                } catch (Exception e) {
                                    echo "Latest push attempt ${retryCount} failed: ${e.getMessage()}"
                                    if (retryCount >= maxRetries) {
                                        echo "Warning: Failed to push 'latest' tag, but versioned tag was successful. Continuing pipeline..."
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to K8s') {
            agent {
                docker {
                    // Use a kubectl image to run deployment commands
                    image 'bitnami/kubectl:latest'
                    // IMPORTANT: Use --entrypoint="" to bypass Bitnami's custom entrypoint
                    args '--entrypoint="" -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                // Get the Kubeconfig content from Jenkins credentials
                withCredentials([string(credentialsId: 'minikube-config', variable: 'KUBECFG_CONTENT')]) {
                    withEnv(["IMAGE_TAG=${BUILD_NUMBER}"]) {
                        sh "mkdir -p .kube"

                        // Write the Kubeconfig secret content to a temporary file
                        writeFile file: '.kube/config.temp', text: "$KUBECFG_CONTENT"

                        echo "Performing ultimate cleanup on Kubeconfig file..."

                        // 1. Aggressively strip ALL non-printable characters (except basic space, tab, and newline)
                        sh "tr -cd '\\11\\12\\15\\40-\\176' < .kube/config.temp > .kube/config"

                        // 2. Remove all blank lines and all leading/trailing whitespace in-place
                        sh 'sed -i -e "/^$/d" -e "s/^[[:space:]]*//" -e "s/[[:space:]]*$//" .kube/config'
                        
                        // Clean up temporary file
                        sh "rm .kube/config.temp"

                        sh 'chmod 600 .kube/config'

                        echo "Deploying image ${DOCKER_IMAGE}:${IMAGE_TAG} to Kubernetes..."

                        // Replace the placeholder image tag in kubernetes-deployment.yaml
                        sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${DOCKER_IMAGE}:${IMAGE_TAG}|g' kubernetes-deployment.yaml"

                        // Apply the deployment using the cleaned config file
                        withEnv(["KUBECONFIG=.kube/config"]) {
                            sh "kubectl apply -f kubernetes-deployment.yaml"
                        }

                        // Clean up the working directory after deployment
                        sh 'git checkout kubernetes-deployment.yaml'
                        sh 'rm -rf .kube'
                    }
                }
            }
        }
    }
}
