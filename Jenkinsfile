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
                    
                    # --- ROBUST DECODING START ---
                    # Function to reliably extract and decode Base64 data
                    extract_and_decode() {
                        # Isolates the Base64 string, removes ALL whitespace (including spaces and newlines) for clean decoding, and saves to file.
                        grep "$1" $KUBECONFIG_SOURCE | awk '{print $2}' | sed -e 's/[[:space:]]//g' | base64 -d > "$2"
                    }

                    # Decode CA Certificate
                    extract_and_decode 'certificate-authority-data:' ca.crt
                    
                    # Decode Client Certificate
                    extract_and_decode 'client-certificate-data:' client.crt
                    
                    # Decode Client Key
                    extract_and_decode 'client-key-data:' client.key
                    
                    echo "Certificates successfully extracted to ca.crt, client.crt, client.key"
                    # --- ROBUST DECODING END ---
                    
                    # 3. Create a NEW KUBECONFIG using file paths and the NEW EXTERNAL IP.
                    # !!! IMPORTANT: REPLACE 'YOUR_EXTERNAL_K8S_IP' WITH THE ROUTABLE IP ADDRESS !!!
                    cat << EOF > $KUBECONFIG_CLEAN
apiVersion: v1
clusters:
- cluster:
    certificate-authority: $PWD/ca.crt
    server: https://YOUR_EXTERNAL_K8S_IP:8443
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

                    # 4. Set KUBECONFIG to point to the new, clean file
                    export KUBECONFIG=$KUBECONFIG_CLEAN

                    # --- TESTING KUBERNETES CONNECTION ---
                    echo "--- Testing Kubernetes Connection ---"
                    
                    # This test will only succeed if 'YOUR_EXTERNAL_K8S_IP' is correct and routable.
                    kubectl cluster-info
                    
                    # --- DEPLOYMENT STEP ---
                    echo "--- Starting Deployment Logic ---"
                    # kubectl apply -f my-deployment.yaml

                    echo "Deployment logic completed."
                    '''
                }
            }
        }
    }
}
