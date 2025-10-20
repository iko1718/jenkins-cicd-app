// Jenkinsfile to deploy an application to a Kubernetes cluster using a Secret File credential.

pipeline {
    // Using a standard 'ubuntu' image and installing kubectl manually for robustness.
    agent {
        docker {
            image 'ubuntu:latest'
            args '-u root' // Helps ensure permissions inside the container
        }
    }

    stages {
        stage('Setup Tools') {
            steps {
                sh '''
                echo "Installing dependencies..."
                
                # 1. Update and install necessary packages (curl, gnupg, apt-transport-https)
                # FIX: Added 'gnupg' to resolve the "E: gnupg... not installed" error.
                apt-get update -y
                apt-get install -y curl apt-transport-https gnupg

                # 2. Install kubectl (using the standard Kubernetes repository setup)
                curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
                echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
                
                apt-get update -y
                # Install kubectl at the version corresponding to our target cluster, v1.29.0
                apt-get install -y kubectl=1.29.0-00 
                
                echo "kubectl installed successfully."
                kubectl version --client --short
                '''
            }
        }

        stage('Checkout Code') {
            steps {
                echo 'Source code checked out.'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // 1. Use the Secret File credential named 'kubeconfig'
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE_PATH')]) {
                    
                    // 2. Set the KUBECONFIG environment variable. This is the key step.
                    sh 'export KUBECONFIG=${KUBECONFIG_FILE_PATH}'
                    
                    echo "KUBECONFIG set to: ${KUBECONFIG_FILE_PATH}"

                    // 3. Run kubectl commands using the embedded configuration
                    sh '''
                    echo "--- Applying Kubernetes deployment (nginx-deployment.yaml) ---"
                    
                    # Check cluster connection
                    kubectl cluster-info

                    # Apply the deployment YAML
                    kubectl apply -f nginx-deployment.yaml
                    
                    echo "Deployment applied successfully."
                    
                    # Wait for the deployment to become ready
                    kubectl wait --for=condition=Available deployment/nginx-web --timeout=120s

                    # Apply a simple NodePort service to expose the app (Minikube access)
                    kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-web-service
spec:
  type: NodePort
  selector:
    app: nginx-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF

                    echo "Service exposed via NodePort."
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline finished."
        }
    }
}
