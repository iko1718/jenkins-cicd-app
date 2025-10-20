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
                    
                    # Extract and decode the CA cert from the Jenkins-provided file
                    CA_DATA=$(grep 'certificate-authority-data:' $KUBECONFIG_SOURCE | awk '{print $2}')
                    echo $CA_DATA | base64 -d > ca.crt
                    
                    # Extract and decode the Client Certificate
                    CERT_DATA=$(grep 'client-certificate-data:' $KUBECONFIG_SOURCE | awk '{print $2}')
                    echo $CERT_DATA | base64 -d > client.crt
                    
                    # Extract and decode the Client Key
                    KEY_DATA=$(grep 'client-key-data:' $KUBECONFIG_SOURCE | awk '{print $2}')
                    echo $KEY_DATA | base64 -d > client.key
                    
                    # 3. Create a NEW KUBECONFIG using file paths instead of Base64 data.
                    # This ensures the PEM data is clean and correctly formatted for kubectl.
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

                    # 4. Set KUBECONFIG to point to the new, clean file
                    export KUBECONFIG=$KUBECONFIG_CLEAN

                    # --- SECURITY CHECK ---
                    echo "--- Testing Kubernetes Connection (Should succeed now) ---"
                    kubectl cluster-info

                    # --- DEPLOYMENT STEP (Replace with your actual commands) ---
                    echo "--- Starting Deployment with Kubeconfig ---"
                    # kubectl apply -f my-deployment.yaml

                    echo "Deployment logic completed."
                    '''
                }
            }
        }
    }
}
