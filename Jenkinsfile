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
                    
                    # Robust extraction function
                    extract_base64_clean() {
                        local key="\$1"
                        local output="\$2"
                        
                        echo "Extracting \$key..."
                        # Multiple extraction methods to ensure we get clean base64
                        data=\$(grep "\$key" kubeconfig_unix.yaml | sed "s/.*\$key//" | tr -d '[:space:]' | tr -d '\\r')
                        
                        if [ -n "\$data" ]; then
                            echo "Base64 data length: \${#data}"
                            # Try to decode and check if it's valid
                            if echo "\$data" | base64 -d > "\$output" 2>/dev/null; then
                                if [ -s "\$output" ]; then
                                    echo "✓ Successfully extracted \$key"
                                    return 0
                                fi
                            fi
                        fi
                        echo "✗ Failed to extract \$key"
                        return 1
                    }

                    # Extract certificates
                    extract_base64_clean "certificate-authority-data:" ca.crt
                    extract_base64_clean "client-certificate-data:" client.crt
                    extract_base64_clean "client-key-data:" client.key.raw

                    echo "=== ANALYZING EXTRACTED KEY ==="
                    if [ -f "client.key.raw" ]; then
                        echo "Raw key file info:"
                        ls -la client.key.raw
                        echo "First few lines of key:"
                        head -5 client.key.raw
                        echo "File type:"
                        file client.key.raw
                    fi

                    echo "=== CONVERTING PRIVATE KEY FORMAT ==="
                    # Multiple conversion methods
                    
                    # Method 1: Try openssl conversion
                    if [ -f "client.key.raw" ] && head -1 client.key.raw | grep -q "BEGIN RSA PRIVATE KEY"; then
                        echo "Attempting openssl RSA to PKCS#8 conversion..."
                        if openssl pkcs8 -topk8 -nocrypt -in client.key.raw -out client.key 2>/dev/null; then
                            echo "✓ Successfully converted RSA key to PKCS#8 using openssl"
                        else
                            echo "openssl conversion failed, trying alternative methods..."
                            
                            # Method 2: Use Python cryptography library
                            python3 << 'EOF'
import base64
import os

try:
    # Read the raw key file
    with open('client.key.raw', 'rb') as f:
        key_data = f.read()
    
    # Try to decode as PEM
    if b'BEGIN RSA PRIVATE KEY' in key_data:
        print("Found RSA private key, converting to PKCS#8...")
        
        # Import RSA and convert
        from cryptography.hazmat.primitives import serialization
        from cryptography.hazmat.backends import default_backend
        
        private_key = serialization.load_pem_private_key(
            key_data,
            password=None,
            backend=default_backend()
        )
        
        # Convert to PKCS#8
        pkcs8_key = private_key.private_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PrivateFormat.PKCS8,
            encryption_algorithm=serialization.NoEncryption()
        )
        
        with open('client.key', 'wb') as f:
            f.write(pkcs8_key)
        
        print("✓ Successfully converted RSA key to PKCS#8 using Python")
        
    elif b'BEGIN PRIVATE KEY' in key_data:
        print("Key is already in PKCS#8 format")
        os.rename('client.key.raw', 'client.key')
    else:
        print("Unknown key format, using as-is")
        os.rename('client.key.raw', 'client.key')
        
except Exception as e:
    print(f"Error converting key: {e}")
    # Fallback: use the raw key as-is
    try:
        os.rename('client.key.raw', 'client.key')
        print("Using raw key as fallback")
    except:
        pass
EOF
                        fi
                    else
                        echo "Key doesn't appear to be RSA format, using as-is"
                        mv client.key.raw client.key 2>/dev/null || true
                    fi

                    echo "=== FINAL KEY VERIFICATION ==="
                    if [ -f "client.key" ]; then
                        echo "Final key file:"
                        ls -la client.key
                        echo "Key format:"
                        head -2 client.key
                        echo "Key validation:"
                        if openssl pkey -in client.key -noout -check 2>/dev/null; then
                            echo "✓ Key is valid"
                        else
                            echo "✗ Key validation failed"
                        fi
                    else
                        echo "✗ No client key file created"
                    fi

                    echo "=== VERIFYING CERTIFICATE-KEY PAIR ==="
                    if [ -f "client.crt" ] && [ -f "client.key" ]; then
                        echo "Verifying certificate and key compatibility..."
                        openssl x509 -noout -modulus -in client.crt 2>/dev/null | openssl md5 > /tmp/cert.md5 2>/dev/null
                        openssl pkey -in client.key -noout -modulus 2>/dev/null | openssl md5 > /tmp/key.md5 2>/dev/null || \
                        openssl rsa -in client.key -noout -modulus 2>/dev/null | openssl md5 > /tmp/key.md5 2>/dev/null
                        
                        if [ -f "/tmp/cert.md5" ] && [ -f "/tmp/key.md5" ]; then
                            if diff /tmp/cert.md5 /tmp/key.md5 > /dev/null; then
                                echo "✓ Certificate and key are compatible"
                            else
                                echo "✗ Certificate and key do not match!"
                            fi
                        else
                            echo "? Could not verify certificate-key compatibility"
                        fi
                    fi

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
                
                # Test connection with multiple approaches
                echo "--- Testing basic connection ---"
                if ./kubectl get nodes --insecure-skip-tls-verify 2>/dev/null; then
                    echo "✓ Successfully connected to Kubernetes!"
                else
                    echo "Connection failed, trying with more debug info..."
                    
                    # Test with verbose output
                    echo "--- Verbose connection test ---"
                    ./kubectl --v=3 get nodes --insecure-skip-tls-verify 2>&1 | head -30
                    
                    # Check if it's a certificate issue
                    echo "--- Certificate validation ---"
                    if [ -f "client.key" ]; then
                        echo "Client key format: \$(head -1 client.key)"
                    fi
                    
                    exit 1
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
            rm -f ca.crt client.crt client.key client.key.raw client.key.rsa kubeconfig_unix.yaml kubeconfig_clean.yaml /tmp/cert.md5 /tmp/key.md5
            '''
        }
    }
}
