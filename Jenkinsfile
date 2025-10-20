// This is the file that defines your build, test, and deployment process in Jenkins.
pipeline {
    agent any

    stages {
        stage('Setup Tools') {
            steps {
                sh '''
                echo "--- Installing kubectl locally in the workspace ---"
                
                # 1. Get the latest stable Kubernetes version number
                KUBE_VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
                
                # 2. Download the kubectl binary to the CURRENT WORKING DIRECTORY (./kubectl)
                # This directory is the build workspace, which the Jenkins user can always write to.
                curl -LO "https://storage.googleapis.com/kubernetes-release/release/$KUBE_VERSION/bin/linux/amd64/kubectl"
                
                # 3. Make the binary executable
                chmod +x ./kubectl
                
                # NOTE: The 'mv' command to /usr/local/bin has been removed to avoid Permission Denied errors.
                
                echo "kubectl downloaded and ready in: $PWD/kubectl"
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
                    # Add the current directory (where ./kubectl is) to the execution PATH.
                    # This allows the 'kubectl' command to be found without specifying './kubectl'.
                    export PATH=$PATH:$PWD
                    
                    echo "Kubeconfig secret file successfully accessed at: ****"

                    # Set the KUBECONFIG environment variable to the path provided by Jenkins
                    export KUBECONFIG=$KUBECONFIG_FILE

                    # --- SECURITY CHECK ---
                    echo "--- Testing Kubernetes Connection ---"
                    
                    # This command will now succeed because the PATH is set correctly.
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
