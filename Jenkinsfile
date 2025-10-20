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
                    // Credentials ID: dockerhub-creds
                    docker.withRegistry('', 'dockerhub-creds') {
                        echo "Building Docker image: ${DOCKER_IMAGE}:${imageTag}"
                        def customImage = docker.build("${DOCKER_IMAGE}:${imageTag}", '.')
                    }
                }
            }
        }

        stage('Push Image') {
            steps {
                // We use the docker tool agent for this step
                script {
                    def imageTag = env.BUILD_NUMBER
                    // Credentials ID: dockerhub-creds
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-creds') {
                        echo "Logging into Docker Hub and pushing image: ${DOCKER_IMAGE}:${imageTag}"
                        def customImage = docker.image("${DOCKER_IMAGE}:${imageTag}")
                        customImage.push()
                        // Tag and push 'latest'
                        customImage.tag('latest')
                        customImage.push('latest')
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

                        // --- FINAL, ROBUST KUBECONFIG CLEANUP ---
                        sh """
                            echo "Performing aggressive cleanup on Kubeconfig file..."
                            
                            # 1. Use 'tr' to strip ALL non-printable control characters and carriage returns (CR)
                            tr -cd '[:print:]\n' < .kube/config.temp > .kube/config
                            
                            # 2. Use 'sed' on the final file to remove ALL blank lines and all leading/trailing whitespace
                            # This ensures the YAML starts exactly with 'apiVersion'.
                            sed -i -e '/^$/d' -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//' .kube/config
                            
                            # Clean up temporary file
                            rm .kube/config.temp
                        """

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
