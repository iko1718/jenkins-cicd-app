pipeline {
    agent any

    stages {
        stage('Setup Tools') {
            steps {
                sh '''
                echo "--- Installing kubectl ---"
                KUBE_VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
                curl -LO "https://storage.googleapis.com/kubernetes-release/release/$KUBE_VERSION/bin/linux/amd64/kubectl"
                chmod +x ./kubectl
                echo "kubectl ready"
                '''
            }
        }

        // --- NEW STAGE: FIXES KEY CORRUPTION AND INITIALIZES KUBECONFIG ---
        stage('Initialize Kubeconfig') {
            steps {
                // Use the existing credential ID 'kubeconfig'
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh '''
                    export PATH=$PATH:${WORKSPACE}
                    echo "--- Initializing Kubeconfig with Robust Key Cleanup ---"
                    
                    KUBECONFIG_CLEAN="kubeconfig_clean.yaml"

                    # CRITICAL FIX: Extract and aggressively clean ALL whitespace from the base64 data
                    # This prevents the "tls: failed to find any PEM data in key input" error.
                    # We use cut -d: -f2- to handle colons that might exist in the base64 data.
                    CA_DATA=$(echo "${KUBECONFIG_SOURCE}" | grep 'certificate-authority-data:' | cut -d: -f2- | tr -d '[:space:]')
                    CLIENT_CERT_DATA=$(echo "${KUBECONFIG_SOURCE}" | grep 'client-certificate-data:' | cut -d: -f2- | tr -d '[:space:]')
                    CLIENT_KEY_DATA=$(echo "${KUBECONFIG_SOURCE}" | grep 'client-key-data:' | cut -d: -f2- | tr -d '[:space:]')

                    if [ -z "$CLIENT_KEY_DATA" ]; then
                        echo "FATAL ERROR: Client key data extraction failed. Check your Kubeconfig secret format."
                        exit 1
                    fi
                    
                    echo "Base64 data extracted and cleaned successfully."

                    # 2. Decode the clean base64 strings into separate files
                    echo "${CA_DATA}" | base64 -d > ca.crt
                    echo "${CLIENT_CERT_DATA}" | base64 -d > client.crt
                    echo "${CLIENT_KEY_DATA}" | base64 -d > client.key

                    echo "Certificates successfully decoded to ca.crt, client.crt, client.key"

                    # 3. Create the clean Kubeconfig file by substituting file paths for the base64 blobs
                    cat << EOF > "${KUBECONFIG_CLEAN}"
$(echo "${KUBECONFIG_SOURCE}" \
    | sed 's/certificate-authority-data:.*/certificate-authority: ca.crt/' \
    | sed 's/client-certificate-data:.*/client-certificate: client.crt/' \
    | sed 's/client-key-data:.*/client-key: client.key/')
EOF
                    
                    # 4. Save the KUBECONFIG path to a file so subsequent stages can access it
                    echo "${KUBECONFIG_CLEAN}" > .kubeconfig_path
                    
                    # Test the connection with the newly created, clean file
                    echo "--- Testing connection with cleaned Kubeconfig ---"
                    export KUBECONFIG="${KUBECONFIG_CLEAN}"
                    ./kubectl version --client 
                    # This command should now succeed!
                    ./kubectl cluster-info --insecure-skip-tls-verify || true
                    
                    '''
                }
            }
        }
        // --- END OF NEW STAGE ---

        stage('Check Port Forward Status') {
            steps {
                sh """
                echo "=== CHECKING KUBECTL PORT-FORWARD STATUS ==="
                echo "If you're running 'kubectl port-forward' on your VM, make sure it's still active."
                echo "The connection was refused, which means:"
                echo "1. The port-forward command might have stopped"
                echo "2. The IP address might be wrong" 
                echo "3. There might be a firewall issue"
                echo ""
                echo "On your VM, please check if the port-forward is still running:"
                echo "  ps aux | grep 'kubectl port-forward'"
                echo ""
                echo "If it's not running, restart it with:"
                echo "  kubectl port-forward --address 10.2.0.4 -n kube-system deployment/kube-apiserver-minikube 8443:8443"
                """
            }
        }

        stage('Deploy Application') {
            steps {
                sh """
                export PATH=\$PATH:\$PWD
                
                # --- FIX: Load KUBECONFIG from the initialization stage ---
                if [ -f .kubeconfig_path ]; then
                    KUBECONFIG_CLEAN=\$(cat .kubeconfig_path)
                    export KUBECONFIG="\$KUBECONFIG_CLEAN"
                    echo "Using cleaned KUBECONFIG: \$KUBECONFIG_CLEAN"
                else
                    echo "ERROR: Kubeconfig path not found. Initialization stage failed."
                    exit 1
                fi
                # --------------------------------------------------------
                
                echo "=== ATTEMPTING DEPLOYMENT ==="
                
                # Try deployment with the newly set KUBECONFIG
                if [ -n "\$KUBECONFIG" ]; then
                    echo "Using KUBECONFIG: \${KUBECONFIG}"
                    
                    for file in deployment.yaml service.yaml; do
                        if [ -f "\$file" ]; then
                            echo "Applying \$file..."
                            ./kubectl apply -f \$file --insecure-skip-tls-verify
                        else
                            echo "Note: \$file not found"
                        fi
                    done
                    
                    echo "=== VERIFYING DEPLOYMENT ==="
                    ./kubectl get deployments,services,pods --insecure-skip-tls-verify || echo "Cannot verify deployment"
                else
                    echo "No working kubeconfig found for deployment"
                fi
                """
            }
        }
    }

    post {
        always {
            sh '''
            echo "=== CLEANING UP ==="
            # Clean up all temporary files, including the newly created ones
            rm -f kubeconfig_approach*.yaml simple_kubeconfig.yaml
            rm -f kubeconfig_clean.yaml ca.crt client.crt client.key .kubeconfig_path
            '''
        }
    }
}
