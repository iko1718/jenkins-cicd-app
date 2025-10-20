pipeline {
    // The agent for the outer pipeline, where the initial SCM checkout happens.
    agent any

    // Define the required environment variables or tool images
    environment {
        // Use a lightweight base image where we can install kubectl
        KUBE_TOOLS_IMAGE = 'ubuntu:latest'
    }

    stages {
        // Stage 1: Initial SCM Checkout (as seen in your log)
        stage('Declarative: Checkout SCM') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        // Stage 2: Execute build and deployment steps inside a Docker container
        stage('CI/CD Process') {
            agent {
                docker {
                    image "${KUBE_TOOLS_IMAGE}"
                    // Important: Use 'root' user so we can run apt-get install
                    args '-u root'
                }
            }
            stages {
                
                // Sub-Stage A: Ensure the code is available inside the container
                stage('Checkout Code') {
                    steps {
                        // Re-checkout the code to ensure the container volume sees it correctly
                        checkout scm
                    }
                }

                // Sub-Stage B: Setup kubectl and other tools
                stage('Setup Tools') {
                    steps {
                        echo "Installing kubectl inside ${KUBE_TOOLS_IMAGE}..."
                        sh """
                            # 1. Update package list and install prerequisites
                            apt-get update -y
                            apt-get install -y curl gnupg apt-transport-https

                            # 2. Add Kubernetes official signing key
                            install -m 0755 -d /etc/apt/keyrings
                            curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
                            
                            # 3. Add the Kubernetes repository
                            echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
                            
                            # 4. Install kubectl
                            apt-get update -y
                            apt-get install -y kubectl
                            
                            # 5. Verify installation
                            kubectl version --client
                        """
                    }
                }

                // Sub-Stage C: FIXED DEPLOYMENT COMMAND AND ADDED AUTHENTICATION
                stage('Deploy to Kubernetes') {
                    steps {
                        echo '--- Starting Deployment to Kubernetes Cluster ---'
                        // Use Jenkins credentials to securely inject the Kubeconfig file content.
                        withCredentials([file(credentialsId: 'k8s_prod_config', variable: 'KUBECONFIG_PATH')]) {
                            echo "Kubeconfig injected from credentials. Deploying..."
                            // Set the KUBECONFIG environment variable to point to the temporary file
                            // created by Jenkins, allowing kubectl to connect to the cluster.
                            sh "export KUBECONFIG=\$KUBECONFIG_PATH && kubectl apply -f kubernetes-deployment.yaml"
                        }
                        echo 'Deployment command executed. The next step will confirm if the cluster accepted the changes.'
                    }
                }
            }
        }
    }
}
