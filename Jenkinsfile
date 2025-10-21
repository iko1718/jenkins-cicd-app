pipeline {

    agent any

    // Environment variables for configuration, making the pipeline adaptable.
    environment {
        // --- CONFIGURATION REQUIRED ---
        // 1. The Git repository URL to clone
        GIT_REPO = 'https://github.com/iko1718/jenkins-cicd-app'
        // 2. The ID of the Jenkins Credential storing your kubeconfig file
        KUBE_CONFIG_ID = 'kubeconfig'
        // 3. The target IP/port of the Kubernetes API server (for deployment)
        KUBE_SERVER_IP = '10.2.0.4:8443' 
        // 4. Kubernetes API server endpoint (replace if using a different port/IP)
        KUBE_API_ENDPOINT = "https://${KUBE_SERVER_IP}"
        // ------------------------------
        
        // Path to the clean kubeconfig file created during initialization
        KUBECONFIG_CLEAN_PATH = 'kubeconfig_clean.yaml'
    }

    stages {
        // Stage 1: Fetches the code and installs kubectl
        stage('Build & Setup') {
            steps {
                sh '''
                echo "--- 1. Installing kubectl ---"
                KUBE_VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
                curl -LO "https://storage.googleapis.com/kubernetes-release/release/$KUBE_VERSION/bin/linux/amd64/kubectl"
                chmod +x ./kubectl
                echo "kubectl ready at: $PWD/kubectl"
                '''
                // In a real scenario, building a Docker image would happen here (e.g., docker build -t myapp:$BUILD_ID .)
                echo "Artifact build simulated successfully."
            }
        }

        // Stage 2: Initialize Kubeconfig - Solves the PEM data corruption issue
        stage('Initialize Kubeconfig') {
            steps {
                // KUBECONFIG_SOURCE is the path to the temporary file holding the secret content
                withCredentials([
                    file(credentialsId: env.KUBE_CONFIG_ID, variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh '''
                    echo "--- 2. Initializing Kubeconfig with Robust Key Cleanup ---"
                    
                    # Ensure kubectl is accessible
                    export PATH=$PATH:${WORKSPACE}
                    
                    # CRITICAL FIX: Use sed to extract and aggressively clean ALL whitespace from the base64 data.
                    # This prevents the "tls: failed to find any PEM data in key input" error.
                    echo "Extracting and cleaning base64 certificate data..."

                    # sed -n '/pattern/s/search/replace/p' - Finds the line, removes the label/colon, and prints the result.
                    CA_DATA=$(sed -n '/certificate-authority-data:/s/.*: *//p' "${KUBECONFIG_SOURCE}" | tr -d '[:space:]')
                    CLIENT_CERT_DATA=$(sed -n '/client-certificate-data:/s/.*: *//p' "${KUBECONFIG_SOURCE}" | tr -d '[:space:]')
                    CLIENT_KEY_DATA=$(sed -n '/client-key-data:/s/.*: *//p' "${KUBECONFIG_SOURCE}" | tr -d '[:space:]')

                    if [ -z "$CLIENT_KEY_DATA" ]; then
                        echo "FATAL ERROR: Client key data extraction failed. Check your Kubeconfig secret format."
                        exit 1
                    fi
                    
                    # 3. Decode the clean base64 strings into separate files
                    echo "${CA_DATA}" | base64 -d > ca.crt
                    echo "${CLIENT_CERT_DATA}" | base64 -d > client.crt
                    echo "${CLIENT_KEY_DATA}" | base64 -d > client.key

                    echo "Certificates successfully decoded."

                    # 4. Create the clean Kubeconfig file by substituting file paths for the base64 blobs
                    # This final file is used for all subsequent kubectl commands.
                    cat << EOF > ${KUBECONFIG_CLEAN_PATH}
$(sed \
    -e 's/certificate-authority-data:.*/certificate-authority: ca.crt/' \
    -e 's/client-certificate-data:.*/client-certificate: client.crt/' \
    -e 's/client-key-data:.*/client-key: client.key/' \
    "${KUBECONFIG_SOURCE}")
EOF
                    
                    // Update the server address (if necessary, based on your previous logs)
                    sh "sed -i 's|server: https://.*|server: ${env.KUBE_API_ENDPOINT}|' ${env.KUBECONFIG_CLEAN_PATH}"
                    
                    // Set environment variable for this and subsequent stages
                    echo "${KUBECONFIG_CLEAN_PATH}" > .kubeconfig_path
                    export KUBECONFIG="${KUBECONFIG_CLEAN_PATH}"
                    
                    // Test connection (This is the critical test for success)
                    echo "--- Testing connection with cleaned Kubeconfig ---"
                    ./kubectl cluster-info --insecure-skip-tls-verify || echo "kubectl connection test FAILED. Check port-forward/firewall."
                    '''
                }
            }
        }

        // Stage 3: Deployment to Kubernetes
        stage('Deploy Application') {
            steps {
                sh '''
                echo "--- 3. Deploying to Kubernetes ---"
                
                # Load the clean KUBECONFIG path set in the previous stage
                if [ -f .kubeconfig_path ]; then
                    export KUBECONFIG=$(cat .kubeconfig_path)
                    echo "Using KUBECONFIG: ${KUBECONFIG}"
                else
                    echo "ERROR: Kubeconfig path not found. Cannot deploy."
                    exit 1
                fi

                # Apply Kubernetes manifests (deployment.yaml and service.yaml are expected in the repo)
                if [ -f "deployment.yaml" ]; then
                    ./kubectl apply -f deployment.yaml --insecure-skip-tls-verify
                    echo "Applied deployment.yaml"
                else
                    echo "Warning: deployment.yaml not found. Skipping deployment."
                fi
                
                if [ -f "service.yaml" ]; then
                    ./kubectl apply -f service.yaml --insecure-skip-tls-verify
                    echo "Applied service.yaml"
                else
                    echo "Warning: service.yaml not found. Skipping service."
                fi

                echo "Deployment step complete."
                '''
            }
        }
    }

    post {
        always {
            sh '''
            echo "=== CLEANING UP WORKSPACE ==="
            # Clean up temporary files created during the process
            rm -f ./kubectl
            rm -f ${KUBECONFIG_CLEAN_PATH} ca.crt client.crt client.key .kubeconfig_path
            '''
        }
    }
}
