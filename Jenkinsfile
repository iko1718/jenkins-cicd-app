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
                    
                    # Extract certificates
                    extract_base64_clean() {
                        local key="\$1"
                        local output="\$2"
                        
                        echo "Extracting \$key..."
                        data=\$(grep "\$key" kubeconfig_unix.yaml | sed "s/.*\$key//" | tr -d '[:space:]' | tr -d '\\r')
                        
                        if [ -n "\$data" ]; then
                            echo "\$data" | base64 -d > "\$output"
                            if [ \$? -eq 0 ] && [ -s "\$output" ]; then
                                echo "✓ Successfully extracted \$key"
                                return 0
                            fi
                        fi
                        return 1
                    }

                    # Extract certificates
                    extract_base64_clean "certificate-authority-data:" ca.crt
                    extract_base64_clean "client-certificate-data:" client.crt
                    extract_base64_clean "client-key-data:" client.key.rsa

                    echo "=== CONVERTING RSA PRIVATE KEY TO PKCS#8 FORMAT ==="
                    # Convert RSA key to PKCS#8 format
                    if [ -f "client.key.rsa" ] && head -1 client.key.rsa | grep -q "BEGIN RSA PRIVATE KEY"; then
                        echo "Converting RSA private key to PKCS#8 format..."
                        openssl pkcs8 -topk8 -nocrypt -in client.key.rsa -out client.key
                        echo "✓ RSA key converted to PKCS#8 format"
                        rm -f client.key.rsa
                    else
                        echo "Key is already in PKCS#8 format or conversion not needed"
                        mv client.key.rsa client.key 2>/dev/null || true
                    fi

                    echo "=== VERIFYING CERTIFICATES ==="
                    echo "File verification:"
                    for file in ca.crt client.crt client.key; do
                        if [ -f "\$file" ] && [ -s "\$file" ]; then
                            echo "✓ \$file: \$(stat -c%s \$file) bytes"
                            echo "  First line: \$(head -1 \$file)"
                        else
                            echo "✗ \$file: missing or empty"
                        fi
                    done

                    # Verify the key is in correct format
                    echo "=== KEY FORMAT VERIFICATION ==="
                    if [ -f "client.key" ]; then
                        key_format=\$(head -1 client.key)
                        case "\$key_format" in
                            "-----BEGIN RSA PRIVATE KEY-----")
                                echo "✗ Key is still in RSA format - conversion failed"
                                ;;
                            "-----BEGIN PRIVATE KEY-----")
                                echo "✓ Key is in PKCS#8 format"
                                ;;
                            "-----BEGIN ENCRYPTED PRIVATE KEY-----")
                                echo "✗ Key is encrypted"
                                ;;
                            *)
                                echo "? Key format unknown: \$key_format"
                                ;;
                        esac
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
                
                # Test with verbose output to see what's happening
                echo "--- Testing connection with verbose output ---"
                ./kubectl --v=3 cluster-info --insecure-skip-tls-verify 2>&1 | head -20
                
                # Simple connection test
                echo "--- Simple connection test ---"
                if ./kubectl get nodes --insecure-skip-tls-verify 2>/dev/null; then
                    echo "✓ Successfully connected to Kubernetes!"
                else
                    echo "✗ Connection failed"
                    echo "Debug: Checking certificate and key compatibility..."
                    
                    # Verify certificate and key match
                    openssl x509 -noout -modulus -in client.crt | openssl md5 > /tmp/cert.md5
                    openssl rsa -noout -modulus -in client.key 2>/dev/null | openssl md5 > /tmp/key.md5 || \
                    openssl pkcs8 -in client.key -nocrypt -topk8 -out /dev/null 2>/dev/null && \
                    openssl pkey -in client.key -noout -modulus | openssl md5 > /tmp/key.md5
                    
                    if diff /tmp/cert.md5 /tmp/key.md5 > /dev/null; then
                        echo "✓ Certificate and key are compatible"
                    else
                        echo "✗ Certificate and key do not match!"
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
                if [ -f "deployment.yaml" ]; then
                    echo "Applying deployment.yaml..."
                    ./kubectl apply -f deployment.yaml --insecure-skip-tls-verify
                fi
                
                if [ -f "service.yaml" ]; then
                    echo "Applying service.yaml..."
                    ./kubectl apply -f service.yaml --insecure-skip-tls-verify
                fi
                
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
            rm -f ca.crt client.crt client.key client.key.rsa kubeconfig_unix.yaml kubeconfig_clean.yaml /tmp/cert.md5 /tmp/key.md5
            '''
        }
    }
}
