// This is the file that defines your build, test, and deployment process in Jenkins.
pipeline {
    agent any

    stages {
        stage('Setup Tools') {
            steps {
                sh '''
                echo "--- Installing kubectl locally in the workspace ---"

                # 1. Get the latest stable Kubernetes version number (FIXED TYPO HERE)
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
                withCredentials([
                    // ID must match the Secret File ID you set in Jenkins
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh '''
                    # 1. Add the current directory (where ./kubectl is) to the execution PATH.
                    export PATH=$PATH:$PWD

                    # 2. Define the path for the new, CLEAN kubeconfig file
                    export KUBECONFIG_CLEAN=kubeconfig_clean.yaml

                    echo "--- Decoding and cleaning up Kubeconfig (Final Escaped Quoting Fix) ---"

                    # --- OPTIMIZED DECODING STEPS ---
                    # Uses backslash-escaped single quotes (\\') to guarantee the script is passed correctly to sed.

                    # 1. Decode CA Certificate 
                    sed -n \' /certificate-authority-data:/s/.*: //p \' $KUBECONFIG_SOURCE | base64 -d > ca.crt

                    # 2. Decode Client Certificate
                    sed -n \' /client-certificate-data:/s/.*: //p \' $KUBECONFIG_SOURCE | base64 -d > client.crt

                    # 3. Decode Client Key
                    sed -n \' /client-key-data:/s/.*: //p \' $KUBECONFIG_SOURCE | base64 -d > client.key

                    echo "Certificates successfully extracted to ca.crt, client.crt, client.key"

                    # 4. Create a NEW KUBECONFIG using file paths and the VM's TRUE AZURE PRIVATE IP (10.2.0.4:8443).
                    cat << EOF > $KUBECONFIG_CLEAN
apiVersion: v1
clusters:
- cluster:
    certificate-authority: $PWD/ca.crt
    server: https://10.2.0.4:8443
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

                    # 5. Set KUBECONFIG environment variable to point to the new, clean file
                    export KUBECONFIG=$KUBECONFIG_CLEAN

                    # --- TESTING KUBERNETES CONNECTION ---
                    echo "--- Testing Kubernetes Connection ---"

                    # Flag to skip TLS verification
                    kubectl cluster-info --insecure-skip-tls-verify 

                    # --- DEPLOYMENT STEP ---
                    echo "--- Starting Deployment Logic ---"

                    # Apply deployment and service files, skipping TLS verification
                    kubectl apply -f deployment.yaml --insecure-skip-tls-verify
                    kubectl apply -f service.yaml --insecure-skip-tls-verify

                    echo "Deployment logic completed."
                    '''
                }
            }
        }
    }
}
