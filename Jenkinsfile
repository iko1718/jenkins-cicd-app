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

        stage('Test Connection with Different Approaches') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh """
                    export PATH=\$PATH:\$PWD

                    echo "=== TESTING DIFFERENT CONNECTION APPROACHES ==="
                    
                    # Approach 1: Use original kubeconfig with modified server URL
                    echo "--- Approach 1: Modified original kubeconfig ---"
                    cp "\$KUBECONFIG_SOURCE" kubeconfig_approach1.yaml
                    sed -i 's|server: https://.*|server: https://10.2.0.4:8443|' kubeconfig_approach1.yaml
                    
                    echo "Testing Approach 1..."
                    KUBECONFIG=kubeconfig_approach1.yaml ./kubectl get nodes --insecure-skip-tls-verify || echo "Approach 1 failed"
                    
                    # Approach 2: Try with the original server URL from kubeconfig
                    echo "--- Approach 2: Original server URL ---"
                    ORIGINAL_SERVER=\$(grep "server:" "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | head -1)
                    echo "Original server from kubeconfig: \$ORIGINAL_SERVER"
                    
                    cp "\$KUBECONFIG_SOURCE" kubeconfig_approach2.yaml
                    export KUBECONFIG=kubeconfig_approach2.yaml
                    ./kubectl get nodes --insecure-skip-tls-verify || echo "Approach 2 failed"
                    
                    # Approach 3: Check if we can reach any Kubernetes API
                    echo "--- Approach 3: Testing various endpoints ---"
                    echo "Testing localhost..."
                    curl -k https://localhost:8443/healthz 2>/dev/null || echo "localhost:8443 not reachable"
                    
                    echo "Testing 10.2.0.4..."
                    curl -k https://10.2.0.4:8443/healthz 2>/dev/null || echo "10.2.0.4:8443 not reachable"
                    
                    echo "Testing 192.168.49.2..."
                    curl -k https://192.168.49.2:8443/healthz 2>/dev/null || echo "192.168.49.2:8443 not reachable"
                    
                    # Approach 4: Simple file-based kubeconfig
                    echo "--- Approach 4: File-based kubeconfig ---"
                    cat > simple_kubeconfig.yaml << EOF
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://10.2.0.4:8443
  name: local
contexts:
- context:
    cluster: local
    user: local
  name: local
current-context: local
kind: Config
users:
- name: local
  user: {}
EOF
                    
                    KUBECONFIG=simple_kubeconfig.yaml ./kubectl cluster-info || echo "Approach 4 failed"
                    
                    echo "=== NETWORK DIAGNOSTICS ==="
                    echo "Checking network connectivity..."
                    ping -c 2 10.2.0.4 || echo "Cannot ping 10.2.0.4"
                    netstat -tuln | grep 8443 || echo "No local service on 8443"
                    """
                }
            }
        }

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
                
                echo "=== ATTEMPTING DEPLOYMENT ==="
                
                # Try deployment with the most likely working config
                if [ -f "kubeconfig_approach1.yaml" ]; then
                    export KUBECONFIG=kubeconfig_approach1.yaml
                    echo "Using Approach 1 for deployment..."
                    
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
            rm -f kubeconfig_approach*.yaml simple_kubeconfig.yaml
            '''
        }
    }
}
