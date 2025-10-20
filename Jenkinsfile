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
                // We use the docker tool agent for this step
                script {
                    def imageTag = env.BUILD_NUMBER
                    // Use Groovy login just for image building and simple tagging
                    docker.withRegistry('', 'dockerhub-creds') {
                        echo "Building Docker image: ${DOCKER_IMAGE}:${imageTag}"
                        // Build the image and store the image object reference
                        docker.build("${DOCKER_IMAGE}:${imageTag}", '.')
                    }
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    def imageTag = env.BUILD_NUMBER
                    
                    echo "Starting authenticated shell push for ${DOCKER_IMAGE}:${imageTag}"

                    // Use withCredentials to securely inject username/password variables for raw shell push.
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        // 1. Perform authenticated login using standard input
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        
                        // Enhanced retry logic with exponential backoff
                        def maxRetries = 5
                        def retryCount = 0
                        def pushSuccess = false
                        
                        // --- Push Versioned Tag ---
                        while (!pushSuccess && retryCount < maxRetries) {
                            try {
                                retryCount++
                                echo "Attempt ${retryCount}/${maxRetries}: Pushing ${DOCKER_IMAGE}:${imageTag}"
                                
                                // Add delay between retries (exponential backoff: 10, 20, 40, 80 seconds)
                                if (retryCount > 1) {
                                    def delaySeconds = Math.pow(2, retryCount - 2) * 10
                                    echo "Waiting ${delaySeconds} seconds before retry..."
                                    sleep time: delaySeconds.toInteger(), unit: 'SECONDS'
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
                            
                            // Push latest with same retry logic
                            def latestSuccess = false
                            retryCount = 0
                            
                            while (!latestSuccess && retryCount < maxRetries) {
                                try {
                                    retryCount++
                                    echo "Attempt ${retryCount}/${maxRetries}: Pushing ${DOCKER_IMAGE}:latest"
                                    
                                    if (retryCount > 1) {
                                        def delaySeconds = Math.pow(2, retryCount - 2) * 10
                                        echo "Waiting ${delaySeconds} seconds before retry..."
                                        sleep time: delaySeconds.toInteger(), unit: 'SECONDS'
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
                        // Uses single quotes to prevent Groovy compilation errors with '$'
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
