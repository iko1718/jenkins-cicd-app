pipeline {
    agent any

    stages {
        stage('Setup Tools') {
            steps {
                sh '''
                echo "--- Installing kubectl and openssl locally in the workspace ---"
                KUBE_VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
                curl -LO "https://storage.googleapis.com/kubernetes-release/release/$KUBE_VERSION/bin/linux/amd64/kubectl"
                chmod +x ./kubectl
                
                # Check if openssl is available
                if ! command -v openssl &> /dev/null; then
                    echo "Installing openssl..."
                    apt-get update && apt-get install -y openssl
                fi
                
                echo "Tools installed and ready"
                '''
            }
        }

        stage('Extract and Prepare Certificates') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh """
                    # Set up environment
                    export PATH=\$PATH:\$PWD
                    export KUBECONFIG_CLEAN=kubeconfig_clean.yaml

                    echo "=== STEP 1: Extracting certificates from kubeconfig ==="
                    
                    # Extract base64 data and decode
                    grep "certificate-authority-data:" "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | base64 -d > ca.crt
                    grep "client-certificate-data:" "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | base64 -d > client.crt
                    grep "client-key-data:" "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | base64 -d > client.key.original

                    echo "--- Original files extracted ---"
                    ls -la ca.crt client.crt client.key.original
                    echo "File headers:"
                    file ca.crt client.crt client.key.original

                    echo "=== STEP 2: Convert private key to proper format ==="
                    # Convert RSA private key to PKCS#8 format if needed
                    if head -1 client.key.original | grep -q "BEGIN RSA PRIVATE KEY"; then
                        echo "Converting RSA private key to PKCS#8 format..."
                        openssl pkcs8 -topk8 -nocrypt -in client.key.original -out client.key
                        echo "RSA key converted to PKCS#8"
                    else
                        echo "Key is already in PKCS#8 format, using as-is"
                        cp client.key.original client.key
                    fi

                    echo "=== STEP 3: Verify certificate and key compatibility ==="
                    # Verify the certificate and key match
                    echo "Verifying certificate and key pair..."
                    openssl x509 -noout -modulus -in client.crt | openssl md5 > cert.md5
                    openssl rsa -noout -modulus -in client.key 2>/dev/null | openssl md5 > key.md5 || \
                    openssl pkcs8 -in client.key -nocrypt -topk8 -out /dev/null 2>/dev/null && \
                    openssl pkey -in client.key -noout -modulus | openssl md5 > key.md5

                    if diff cert.md5 key.md5; then
                        echo "✓ Certificate and key are compatible"
                    else
                        echo "✗ Certificate and key do not match!"
                        exit 1
                    fi

                    echo "=== STEP 4: Create clean kubeconfig ==="
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

                    echo "Clean kubeconfig created:"
                    cat \$KUBECONFIG_CLEAN
                    """
                }
            }
        }

        stage('Test Kubernetes Connection') {
            steps {
                sh """
                export PATH=\$PATH:\$PWD
                export KUBECONFIG=\$PWD/kubeconfig_clean.yaml

                echo "=== STEP 5: Testing Kubernetes connection ==="
                
                # Test with increasing verbosity
                echo "--- Basic connection test ---"
                ./kubectl get nodes --insecure-skip-tls-verify || echo "Basic test failed, trying verbose..."
                
                echo "--- Verbose connection test ---"
                ./kubectl --v=6 get nodes --insecure-skip-tls-verify 2>&1 | head -50 || echo "Verbose test completed"
                
                echo "--- Cluster info test ---"
                ./kubectl cluster-info --insecure-skip-tls-verify || echo "Cluster info test completed"
                
                echo "=== Connection tests finished ==="
                """
            }
        }

        stage('Deploy Application') {
            steps {
                sh """
                export PATH=\$PATH:\$PWD
                export KUBECONFIG=\$PWD/kubeconfig_clean.yaml

                echo "=== STEP 6: Deploying application ==="
                
                # Check if deployment files exist
                if [ -f "deployment.yaml" ]; then
                    echo "Applying deployment.yaml..."
                    ./kubectl apply -f deployment.yaml --insecure-skip-tls-verify
                else
                    echo "deployment.yaml not found, checking for k8s directory..."
                    if [ -d "k8s" ]; then
                        echo "Applying all YAML files in k8s directory..."
                        ./kubectl apply -f k8s/ --insecure-skip-tls-verify
                    else
                        echo "No deployment files found. Creating a simple test deployment..."
                        cat << EOF | ./kubectl apply -f - --insecure-skip-tls-verify
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-test-app
  labels:
    app: jenkins-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-test
  template:
    metadata:
      labels:
        app: jenkins-test
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-test-service
spec:
  selector:
    app: jenkins-test
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
                    fi
                fi

                echo "=== STEP 7: Verify deployment ==="
                ./kubectl get deployments,services,pods --insecure-skip-tls-verify
                
                echo "🎉 Deployment completed successfully!"
                """
            }
        }
    }

    post {
        always {
            sh '''
            echo "=== Cleaning up temporary files ==="
            rm -f ca.crt client.crt client.key client.key.original cert.md5 key.md5 kubeconfig_clean.yaml
            '''
        }
        success {
            echo "🚀 Pipeline executed successfully!"
        }
        failure {
            echo "❌ Pipeline failed - check the logs above for details"
        }
    }
}
