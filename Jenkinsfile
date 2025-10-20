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
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh """
                    # 1. Add the current directory (where ./kubectl is) to the execution PATH.
                    export PATH=\$PATH:\$PWD

                    # 2. Define the path for the new, CLEAN kubeconfig file
                    export KUBECONFIG_CLEAN=kubeconfig_clean.yaml

                    echo "--- Decoding and cleaning up Kubeconfig (Improved Extraction) ---"

                    # --- IMPROVED BASE64 EXTRACTION ---
                    # Use awk to extract the base64 data more reliably

                    # 1. Decode CA Certificate 
                    grep 'certificate-authority-data:' "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | base64 -d > ca.crt

                    # 2. Decode Client Certificate
                    grep 'client-certificate-data:' "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | base64 -d > client.crt

                    # 3. Decode Client Key
                    grep 'client-key-data:' "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | base64 -d > client.key

                    # Verify the files were created and have content
                    echo "--- Verifying certificate files ---"
                    ls -la ca.crt client.crt client.key
                    echo "CA cert size:" \$(stat -c%s ca.crt) "bytes"
                    echo "Client cert size:" \$(stat -c%s client.crt) "bytes" 
                    echo "Client key size:" \$(stat -c%s client.key) "bytes"

                    # Check if files contain proper PEM data
                    echo "--- Checking PEM headers ---"
                    head -1 ca.crt
                    head -1 client.crt
                    head -1 client.key

                    echo "Certificates successfully extracted to ca.crt, client.crt, client.key"

                    # 4. Create a NEW KUBECONFIG using file paths and the VM's TRUE AZURE PRIVATE IP (10.2.0.4:8443).
                    cat << EOF > \$KUBECONFIG_CLEAN
apiVersion: v1
clusters:
- cluster:
    certificate-authority: \$PWD/ca.crt
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
    client-certificate: \$PWD/client.crt
    client-key: \$PWD/client.key
EOF

                    # 5. Set KUBECONFIG environment variable to point to the new, clean file
                    export KUBECONFIG=\$KUBECONFIG_CLEAN

                    # --- TESTING KUBERNETES CONNECTION ---
                    echo "--- Testing Kubernetes Connection ---"

                    # First, let's check if we can connect with verbose output
                    ./kubectl --v=6 cluster-info --insecure-skip-tls-verify || true

                    # Try a simpler test first
                    echo "--- Simple API test ---"
                    ./kubectl get nodes --insecure-skip-tls-verify

                    # --- DEPLOYMENT STEP ---
                    echo "--- Starting Deployment Logic ---"

                    # Apply deployment and service files, skipping TLS verification
                    ./kubectl apply -f deployment.yaml --insecure-skip-tls-verify
                    ./kubectl apply -f service.yaml --insecure-skip-tls-verify

                    echo "Deployment logic completed."
                    """
                }
            }
        }
    }
}
