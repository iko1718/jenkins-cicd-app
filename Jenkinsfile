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

        stage('Extract Certificates with Windows Line Ending Fix') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh """
                    export PATH=\$PATH:\$PWD
                    export KUBECONFIG_CLEAN=kubeconfig_clean.yaml

                    echo "=== FIXING WINDOWS LINE ENDINGS AND EXTRACTING CERTIFICATES ==="
                    
                    # Step 1: Convert Windows line endings to Unix and extract
                    echo "Converting Windows CRLF to Unix LF..."
                    sed 's/\\r\$//' "\$KUBECONFIG_SOURCE" > kubeconfig_unix.yaml
                    
                    echo "=== EXTRACTING CERTIFICATES FROM CLEANED FILE ==="
                    
                    # Method 1: Use the cleaned file with proper extraction
                    extract_base64_clean() {
                        local key="\$1"
                        local output="\$2"
                        
                        echo "Extracting \$key..."
                        
                        # Get the base64 data and remove ALL whitespace and control characters
                        data=\$(grep "\$key" kubeconfig_unix.yaml | sed "s/.*\$key//" | tr -d '[:space:]' | tr -d '\\r')
                        
                        if [ -n "\$data" ]; then
                            echo "Data length: \${#data}"
                            echo "\$data" | base64 -d > "\$output"
                            if [ \$? -eq 0 ] && [ -s "\$output" ]; then
                                echo "✓ Successfully extracted \$key"
                                return 0
                            else
                                echo "✗ Failed to decode \$key"
                                return 1
                            fi
                        else
                            echo "✗ No data found for \$key"
                            return 1
                        fi
                    }

                    # Extract certificates using the cleaned method
                    if extract_base64_clean "certificate-authority-data:" ca.crt; then
                        echo "CA certificate extracted"
                    else
                        # Fallback: try awk method on cleaned file
                        echo "Trying fallback extraction..."
                        grep "certificate-authority-data" kubeconfig_unix.yaml | awk '{print \$2}' | tr -d '\\r' | base64 -d > ca.crt
                    fi

                    if extract_base64_clean "client-certificate-data:" client.crt; then
                        echo "Client certificate extracted"
                    else
                        grep "client-certificate-data" kubeconfig_unix.yaml | awk '{print \$2}' | tr -d '\\r' | base64 -d > client.crt
                    fi

                    if extract_base64_clean "client-key-data:" client.key; then
                        echo "Client key extracted"
                    else
                        grep "client-key-data" kubeconfig_unix.yaml | awk '{print \$2}' | tr -d '\\r' | base64 -d > client.key
                    fi

                    echo "=== VERIFYING EXTRACTED FILES ==="
                    echo "File sizes:"
                    ls -la ca.crt client.crt client.key 2>/dev/null || echo "Some files missing"
                    
                    echo "File contents (first line):"
                    [ -f "ca.crt" ] && head -1 ca.crt || echo "ca.crt missing"
                    [ -f "client.crt" ] && head -1 client.crt || echo "client.crt missing"
                    [ -f "client.key" ] && head -1 client.key || echo "client.key missing"

                    # If files are empty, try one more approach
                    if [ ! -s "ca.crt" ] || [ ! -s "client.crt" ] || [ ! -s "client.key" ]; then
                        echo "=== USING ROBUST EXTRACTION METHOD ==="
                        # Most robust method: handle any formatting issues
                        python3 << 'EOF'
import base64
import re

def extract_base64(key, filename):
    with open('kubeconfig_unix.yaml', 'r') as f:
        content = f.read()
    
    # Find the line with the key
    pattern = rf'{key}:\\s*([^\\n]+)'
    match = re.search(pattern, content)
    
    if match:
        b64_data = match.group(1).strip()
        # Remove any remaining whitespace or special characters
        b64_data = re.sub(r'\\s+', '', b64_data)
        
        try:
            decoded = base64.b64decode(b64_data)
            with open(filename, 'wb') as f:
                f.write(decoded)
            print(f"Successfully extracted {key}")
            return True
        except Exception as e:
            print(f"Failed to decode {key}: {e}")
            return False
    else:
        print(f"Key {key} not found")
        return False

extract_base64('certificate-authority-data', 'ca.crt')
extract_base64('client-certificate-data', 'client.crt')  
extract_base64('client-key-data', 'client.key')
EOF
                    fi

                    echo "=== FINAL VERIFICATION ==="
                    for file in ca.crt client.crt client.key; do
                        if [ -f "\$file" ] && [ -s "\$file" ]; then
                            echo "✓ \$file: \$(stat -c%s \$file) bytes, starts with: \$(head -1 \$file)"
                        else
                            echo "✗ \$file: missing or empty"
                        fi
                    done

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

                    echo "Clean kubeconfig created"
                    """
                }
            }
        }

        stage('Test and Deploy') {
            steps {
                sh """
                export PATH=\$PATH:\$PWD
                export KUBECONFIG=\$PWD/kubeconfig_clean.yaml

                echo "=== TESTING KUBERNETES CONNECTION ==="
                
                # Test connection
                if ./kubectl cluster-info --insecure-skip-tls-verify; then
                    echo "✓ Successfully connected to Kubernetes"
                    
                    echo "=== DEPLOYING APPLICATION ==="
                    if [ -f "deployment.yaml" ]; then
                        ./kubectl apply -f deployment.yaml --insecure-skip-tls-verify
                    fi
                    if [ -f "service.yaml" ]; then
                        ./kubectl apply -f service.yaml --insecure-skip-tls-verify
                    fi
                    
                    echo "=== VERIFYING DEPLOYMENT ==="
                    ./kubectl get deployments,services,pods --insecure-skip-tls-verify
                    
                    echo "🎉 PIPELINE COMPLETED SUCCESSFULLY!"
                else
                    echo "✗ Failed to connect to Kubernetes"
                    echo "Debug info:"
                    ./kubectl version --client --insecure-skip-tls-verify
                    exit 1
                fi
                """
            }
        }
    }

    post {
        always {
            sh '''
            echo "=== CLEANING UP ==="
            rm -f ca.crt client.crt client.key kubeconfig_unix.yaml kubeconfig_clean.yaml
            '''
        }
    }
}
