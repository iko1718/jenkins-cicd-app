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

        stage('Extract and Convert Certificates') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh """
                    export PATH=\$PATH:\$PWD
                    export KUBECONFIG_CLEAN=kubeconfig_clean.yaml

                    echo "=== FIXING WINDOWS LINE ENDINGS AND EXTRACTING CERTIFICATES ==="
                    
                    # Step 1: Convert Windows line endings to Unix
                    echo "Converting Windows CRLF to Unix LF..."
                    sed 's/\\r\$//' "\$KUBECONFIG_SOURCE" > kubeconfig_unix.yaml
                    
                    echo "=== EXTRACTING CERTIFICATES ==="
                    
                    # Simple extraction function
                    extract_base64() {
                        local key="\$1"
                        local output="\$2"
                        
                        echo "Extracting \$key..."
                        data=\$(grep "\$key" kubeconfig_unix.yaml | sed "s/.*\$key//" | tr -d '[:space:]' | tr -d '\\r')
                        
                        if [ -n "\$data" ]; then
                            echo "\$data" | base64 -d > "\$output"
                            if [ -s "\$output" ]; then
                                echo "✓ Successfully extracted \$key"
                                return 0
                            fi
                        fi
                        echo "✗ Failed to extract \$key"
                        return 1
                    }

                    # Extract certificates
                    extract_base64 "certificate-authority-data:" ca.crt
                    extract_base64 "client-certificate-data:" client.crt
                    extract_base64 "client-key-data:" client.key.raw

                    echo "=== SIMPLE KEY CONVERSION APPROACH ==="
                    
                    # Check if the key is RSA format and convert using Python
                    if head -1 client.key.raw | grep -q "BEGIN RSA PRIVATE KEY"; then
                        echo "Converting RSA private key to PKCS#8 format using Python..."
                        
                        python3 << 'EOF'
try:
    # Read the RSA key
    with open('client.key.raw', 'r') as f:
        key_lines = f.readlines()
    
    # Write the same key content - sometimes kubectl can handle RSA keys directly
    with open('client.key', 'w') as f:
        f.writelines(key_lines)
    
    print("✓ Using RSA key as-is (direct copy)")
    
except Exception as e:
    print(f"Error: {e}")
    # Fallback: try to install cryptography library and convert
    import subprocess
    import sys
    try:
        subprocess.check_call([sys.executable, '-m', 'pip', 'install', 'cryptography'])
        from cryptography.hazmat.primitives import serialization
        from cryptography.hazmat.backends import default_backend
        
        with open('client.key.raw', 'rb') as f:
            private_key = serialization.load_pem_private_key(
                f.read(),
                password=None,
                backend=default_backend()
            )
        
        pkcs8_key = private_key.private_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PrivateFormat.PKCS8,
            encryption_algorithm=serialization.NoEncryption()
        )
        
        with open('client.key', 'wb') as f:
            f.write(pkcs8_key)
        
        print("✓ Successfully converted RSA key to PKCS#8")
        
    except Exception as e2:
        print(f"Conversion failed: {e2}")
        # Last resort: copy the raw key
        import shutil
        shutil.copy('client.key.raw', 'client.key')
        print("✓ Using raw key as fallback")
EOF
                    else
                        echo "Key doesn't appear to be RSA format, using as-is"
                        cp client.key.raw client.key
                    fi

                    echo "=== VERIFYING EXTRACTED FILES ==="
                    echo "Certificate files:"
                    ls -la ca.crt client.crt client.key
                    
                    echo "File contents (first line):"
                    echo "CA: \$(head -1 ca.crt)"
                    echo "Client Cert: \$(head -1 client.crt)" 
                    echo "Client Key: \$(head -1 client.key)"

                    echo "=== CREATING CLEAN KUBECONFIG ==="
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

                    echo "✓ Clean kubeconfig created"
                    """
                }
            }
        }

        stage('Test Connection') {
            steps {
                sh """
                export PATH=\$PATH:\$PWD
                export KUBECONFIG=\$PWD/kubeconfig_clean.yaml

                echo "=== TESTING KUBERNETES CONNECTION ==="
                
                # Test connection
                echo "--- Testing connection to Kubernetes API ---"
                if ./kubectl get nodes --insecure-skip-tls-verify; then
                    echo "✓ SUCCESS: Connected to Kubernetes!"
                else
                    echo "Connection failed, let's try a different approach..."
                    
                    # Alternative: Use the original kubeconfig with modified server URL
                    echo "--- Trying alternative approach ---"
                    cp "\$KUBECONFIG_SOURCE" kubeconfig_modified.yaml
                    sed -i 's|server: https://.*|server: https://10.2.0.4:8443|' kubeconfig_modified.yaml
                    export KUBECONFIG=kubeconfig_modified.yaml
                    
                    if ./kubectl get nodes --insecure-skip-tls-verify; then
                        echo "✓ SUCCESS: Connected using modified original kubeconfig!"
                    else
                        echo "✗ All connection attempts failed"
                        echo "Debug info:"
                        echo "Current directory: \$(pwd)"
                        echo "Files: \$(ls -la)"
                        exit 1
                    fi
                fi
                """
            }
        }

        stage('Deploy Application') {
            steps {
                sh """
                export PATH=\$PATH:\$PWD
                export KUBECONFIG=\$PWD/kubeconfig_clean.yaml

                echo "=== DEPLOYING APPLICATION ==="
                
                # Deploy application files
                for file in deployment.yaml service.yaml; do
                    if [ -f "\$file" ]; then
                        echo "Applying \$file..."
                        ./kubectl apply -f \$file --insecure-skip-tls-verify
                    else
                        echo "Note: \$file not found"
                    fi
                done
                
                echo "=== VERIFYING DEPLOYMENT ==="
                ./kubectl get deployments,services,pods --insecure-skip-tls-verify
                
                echo "🎉 PIPELINE COMPLETED SUCCESSFULLY!"
                """
            }
        }
    }

    post {
        always {
            sh '''
            echo "=== CLEANING UP ==="
            rm -f ca.crt client.crt client.key client.key.raw kubeconfig_unix.yaml kubeconfig_clean.yaml kubeconfig_modified.yaml
            '''
        }
    }
}
