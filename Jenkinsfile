// This is the file that defines your build, test, and deployment process in Jenkins.
pipeline {
    agent any

    stages {
        stage('Setup Tools') {
            steps {
                sh '''
                echo "--- Installing kubectl ---"
                
                # 1. Get the latest stable Kubernetes version number
                KUBE_VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
                
                # 2. Download the kubectl binary (assuming Linux/amd64 agent)
                curl -LO "https://storage.googleapis.com/kubernetes-release/release/$KUBE_VERSION/bin/linux/amd64/kubectl"
                
                # 3. Make the binary executable
                chmod +x ./kubectl
                
                # 4. Move it to a directory that is in the PATH (e.g., /usr/local/bin)
                # NOTE: This step might require elevated permissions (sudo) depending on your Jenkins agent configuration.
                # If this step fails due to permission errors, you may need to install it to a local bin directory 
                # or contact your Jenkins administrator.
                mv ./kubectl /usr/local/bin/kubectl
                
                echo "kubectl version: $(kubectl version --client --short)"
                '''
            }
        }
        
        stage('Initialize and Deploy') {
            steps {
                // The crucial step: 'withCredentials' handles the secure secret injection.
                withCredentials([
                    // ID must match the Secret File ID you set in Jenkins
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                    echo "Kubeconfig secret file successfully accessed at: ****"

                    # Set the KUBECONFIG environment variable to the path provided by Jenkins
                    export KUBECONFIG=$KUBECONFIG_FILE

                    # --- SECURITY CHECK ---
                    echo "--- Testing Kubernetes Connection ---"
                    kubectl cluster-info

                    # --- DEPLOYMENT STEP (Replace with your actual commands) ---
                    echo "--- Starting Deployment with Kubeconfig ---"
                    # Example: Apply a manifest file using the newly configured kubeconfig
                    # kubectl apply -f my-deployment.yaml

                    echo "Deployment logic completed."
                    '''
                }
            }
        }
    }
}
