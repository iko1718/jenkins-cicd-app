// Jenkinsfile to deploy an application to a Kubernetes cluster using a Secret File credential.

pipeline {
    // FIX: Switched to a known stable image for kubectl (lachlanevenson/k8s-kubectl:v1.29.0)
    agent {
        docker {
            image 'lachlanevenson/k8s-kubectl:v1.29.0'
            args '-u root' // Helps ensure permissions inside the container
        }
    }

    stages {
        stage('Checkout Code') {
            steps {
                // Assuming your source code (including nginx-deployment.yaml) is checked out here.
                echo 'Source code checked out.'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // 1. Use the Secret File credential named 'kubeconfig'
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE_PATH')]) {
                    
                    // 2. Set the KUBECONFIG environment variable. This is the key step.
                    // kubectl automatically looks for this environment variable to find the configuration file.
                    sh 'export KUBECONFIG=${KUBECONFIG_FILE_PATH}'
                    
                    echo "KUBECONFIG set to: ${KUBECONFIG_FILE_PATH}"

                    // 3. Run kubectl commands using the embedded configuration
                    sh """
                    echo "--- Applying Kubernetes deployment (nginx-deployment.yaml) ---"
                    
                    # Check cluster connection (optional but good for debugging)
                    kubectl cluster-info

                    # Apply the deployment YAML
                    kubectl apply -f nginx-deployment.yaml
                    
                    echo "Deployment applied successfully."
                    
                    # Wait for the deployment to become ready
                    kubectl wait --for=condition=Available deployment/nginx-web --timeout=120s

                    # Apply a simple NodePort service to expose the app (Minikube access)
                    kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-web-service
spec:
  type: NodePort
  selector:
    app: nginx-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF

                    echo "Service exposed via NodePort."
                    """
                }
            }
        }
    }
    
    post {
        always {
            // Clean up the deployment regardless of stage success/failure (optional)
            // sh 'kubectl delete deployment nginx-web || true'
            // sh 'kubectl delete service nginx-web-service || true'
            echo "Pipeline finished."
        }
    }
}
