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
                        
                        // Added retry block to handle 504 Gateway Time-out errors
                        retry(3) {
                            echo "Attempting docker push (Attempt ${currentBuild.number})..."
                            // Push the version-specific tag first
                            customImage.push()
                        }

                        // Tag and push 'latest' separately (less critical if it fails)
                        customImage.tag('latest')
                        retry(3) {
                            customImage.push('latest')
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
