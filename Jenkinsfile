pipeline {
    agent any

    stages {
        stage('Setup Tools') {
            steps {
                sh '''
                echo "--- Installing kubectl locally in the workspace ---"
                KUBE_VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
                curl -LO "https://storage.googleapis.com/kubernetes-release/release/$KUBE_VERSION/bin/linux/amd64/kubectl"
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
                    # Set up environment
                    export PATH=\$PATH:\$PWD
                    export KUBECONFIG_CLEAN=kubeconfig_clean.yaml

                    echo "--- Debug: Inspecting kubeconfig file ---"
                    echo "Kubeconfig location: \$KUBECONFIG_SOURCE"
                    echo "First few lines of kubeconfig:"
                    head -20 "\$KUBECONFIG_SOURCE"
                    
                    echo "--- Extracting base64 data with robust cleaning ---"
                    
                    # Function to extract and clean base64 data
                    extract_clean_base64() {
                        local key="\$1"
                        local output_file="\$2"
                        
                        # Extract the line, get everything after the first space, and remove ALL whitespace
                        grep "\$key" "\$KUBECONFIG_SOURCE" | \
                        sed "s/.*\$key//" | \
                        sed 's/^[[:space:]]*//' | \
                        tr -d '[:space:]' | \
                        base64 -d > "\$output_file"
                        
                        # Verify the extraction worked
                        if [ ! -s "\$output_file" ]; then
                            echo "ERROR: Failed to extract \$key"
                            return 1
                        fi
                        return 0
                    }

                    # Extract certificates with robust cleaning
                    echo "Extracting CA certificate..."
                    if ! extract_clean_base64 "certificate-authority-data:" ca.crt; then
                        echo "Trying alternative extraction method for CA..."
                        # Alternative method: use awk and remove any non-base64 characters
                        grep "certificate-authority-data" "\$KUBECONFIG_SOURCE" | \
                        awk '{for(i=2;i<=NF;i++) printf \$i}' | \
                        tr -d '[:space:]' | \
                        base64 -d > ca.crt
                    fi

                    echo "Extracting client certificate..."
                    if ! extract_clean_base64 "client-certificate-data:" client.crt; then
                        echo "Trying alternative extraction method for client cert..."
                        grep "client-certificate-data" "\$KUBECONFIG_SOURCE" | \
                        awk '{for(i=2;i<=NF;i++) printf \$i}' | \
                        tr -d '[:space:]' | \
                        base64 -d > client.crt
                    fi

                    echo "Extracting client key..."
                    if ! extract_clean_base64 "client-key-data:" client.key; then
                        echo "Trying alternative extraction method for client key..."
                        grep "client-key-data" "\$KUBECONFIG_SOURCE" | \
                        awk '{for(i=2;i<=NF;i++) printf \$i}' | \
                        tr -d '[:space:]' | \
                        base64 -d > client.key
                    fi

                    # Verify the files
                    echo "--- Verifying extracted files ---"
                    ls -la ca.crt client.crt client.key
                    echo "File sizes:"
                    wc -c ca.crt client.crt client.key
                    
                    echo "--- Checking file contents ---"
                    echo "CA cert first line:"
                    head -1 ca.crt || echo "Cannot read ca.crt"
                    echo "Client cert first line:"
                    head -1 client.crt || echo "Cannot read client.crt" 
                    echo "Client key first line:"
                    head -1 client.key || echo "Cannot read client.key"

                    # If extraction still failed, try a completely different approach
                    if [ ! -s ca.crt ] || [ ! -s client.crt ] || [ ! -s client.key ]; then
                        echo "--- Using YAML parsing approach ---"
                        # Install yq if not available
                        if ! command -v yq &> /dev/null; then
                            echo "Installing yq..."
                            wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O yq
                            chmod +x yq
                            export PATH=\$PATH:\$PWD
                        fi
                        
                        # Use yq to extract the base64 data
                        ./yq eval '.clusters[0].cluster."certificate-authority-data"' "\$KUBECONFIG_SOURCE" | base64 -d > ca.crt
                        ./yq eval '.users[0].user."client-certificate-data"' "\$KUBECONFIG_SOURCE" | base64 -d > client.crt
                        ./yq eval '.users[0].user."client-key-data"' "\$KUBECONFIG_SOURCE" | base64 -d > client.key
                    fi

                    # Final verification
                    if [ ! -s ca.crt ] || [ ! -s client.crt ] || [ ! -s client.key ]; then
                        echo "ERROR: Failed to extract all required certificates"
                        echo "Debug info:"
                        echo "Kubeconfig content around certificate data:"
                        grep -A 2 -B 2 "certificate-authority-data\\|client-certificate-data\\|client-key-data" "\$KUBECONFIG_SOURCE"
                        exit 1
                    fi

                    echo "--- Creating clean kubeconfig ---"
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

                    export KUBECONFIG=\$KUBECONFIG_CLEAN

                    echo "--- Testing connection ---"
                    ./kubectl cluster-info --insecure-skip-tls-verify

                    echo "--- Deploying application ---"
                    ./kubectl apply -f deployment.yaml --insecure-skip-tls-verify
                    ./kubectl apply -f service.yaml --insecure-skip-tls-verify

                    echo "Deployment completed successfully!"
                    """
                }
            }
        }
    }
}
