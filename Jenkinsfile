// Jenkinsfile to deploy an application to a Kubernetes cluster using a Secret File credential.

pipeline {
    // Using a standard 'ubuntu' image and installing kubectl manually for robustness.
    agent {
        docker {
            image 'ubuntu:latest'
            args '-u root' // Helps ensure permissions inside the container for installation
        }
    }

    stages {
        stage('Setup Tools') {
            steps {
                sh '''
                echo "Installing dependencies..."
                
                # 1. Install necessary utility packages (curl, gnupg, etc.)
                apt-get update -y
                apt-get install -y curl gnupg apt-transport-https
                
                # --- START KUBECTL INSTALL (Modern Method) ---
                
                # 2. Add Kubernetes GPG Key (using the modern, secured keyring method)
                install -m 0755 -d /etc/apt/keyrings
                
                # Download the key and register it with gpg --dearmor
                curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
                
                # 3. Add the Kubernetes repository (targeting v1.29 path)
                echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
                
                # 4. Update package index to include the new repository
                apt-get update -y
                
                # 5. Install kubectl at the specific version v1.29.0
                apt-get install -y kubectl=1.29.0-00 
                
                # --- END KUBECTL INSTALL ---
                
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
                // The 'kubeconfig' Secret File is mounted and the path is stored in KUBECONFIG_FILE_PATH
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE_PATH')]) {
                    
                    // 1. Set the KUBECONFIG environment variable for kubectl
                    sh 'export KUBECONFIG=${KUBECONFIG_FILE_PATH}'
                    
                    echo "KUBECONFIG set to: ${KUBECONFIG_FILE_PATH}"

                    // 2. Run kubectl commands using the embedded configuration
                    sh '''
                    echo "--- Applying Kubernetes deployment (nginx-deployment.yaml) ---"
                    
                    # Check cluster connection (this will test the KUBECONFIG file)
                    kubectl cluster-info

                    # Apply the deployment YAML
                    kubectl apply -f nginx-deployment.yaml
                    
                    echo "Deployment applied successfully."
                    
                    # Wait for the deployment to become ready
                    kubectl wait --for=condition=Available deployment/nginx-web --timeout=120s

                    # Apply a simple NodePort service to expose the app
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
