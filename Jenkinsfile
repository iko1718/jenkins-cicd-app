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
                curl -LO "https://storage.googleapis.com/kubernetes-release/release/$KUBE_VERSION/bin/linux/amd64/kubectl"
                
                # 3. Make the binary executable
                chmod +x ./kubectl
                
                echo "kubectl downloaded and ready in: $PWD/kubectl"
                '''
            }
        }
        
        stage('Initialize and Deploy') {
            steps {
                // The crucial step: 'withCredentials' handles the secure secret injection.
                withCredentials([
                    // ID must match the Secret File ID you set in Jenkins
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh '''
                    # 1. Add the current directory (where ./kubectl is) to the execution PATH.
                    export PATH=$PATH:$PWD
                    
                    # 2. Define the path for the new, CLEAN kubeconfig file
                    export KUBECONFIG_CLEAN=kubeconfig_clean.yaml
                    
                    echo "--- Decoding and cleaning up Kubeconfig ---"
                    
                    # 1. Decode CA Certificate (Explicit commands for robustness)
                    grep 'certificate-authority-data:' $KUBECONFIG_SOURCE | awk '{print $2}' | sed -e 's/[[:space:]]//g' | base64 -d > ca.crt
                    
                    # 2. Decode Client Certificate
                    grep 'client-certificate-data:' $KUBECONFIG_SOURCE | awk '{print $2}' | sed -e 's/[[:space:]]//g' | base64 -d > client.crt
                    
                    # 3. Decode Client Key
                    grep 'client-key-data:' $KUBECONFIG_SOURCE | awk '{print $2}' | sed -e 's/[[:space:]]//g' | base64 -d > client.key
                    
                    echo "Certificates successfully extracted to ca.crt, client.crt, client.key"
                    
                    # 4. Create a NEW KUBECONFIG using file paths and the VM's PRIVATE IP.
                    # **CRITICAL CHANGE**: Using the reliable private IP and port 8443
                    cat << EOF > $KUBECONFIG_CLEAN
apiVersion: v1
clusters:
- cluster:
    certificate-authority: $PWD/ca.crt
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: $PWD/client.crt
    client-key: $PWD/client.key
EOF

                    # 5. Set KUBECONFIG to point to the new, clean file
                    export KUBECONFIG=$KUBECONFIG_CLEAN

                    # --- TESTING KUBERNETES CONNECTION ---
                    echo "--- Testing Kubernetes Connection ---"
                    
                    # This test will now attempt to connect to the private IP on port 8443
                    kubectl cluster-info
                    
                    # --- DEPLOYMENT STEP ---
                    echo "--- Starting Deployment Logic ---"
                    # Add your actual deployment commands here, e.g., kubectl apply -f deployment.yaml

                    echo "Deployment logic completed."
                    '''
                }
            }
        }
    }
}
