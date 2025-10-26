pipeline {
    agent any

    environment {
        GIT_REPO = 'https://github.com/iko1718/jenkins-cicd-app'
        KUBE_CONFIG_ID = 'kubeconfig'
        KUBE_SERVER_IP = '10.2.0.4:8443'
        KUBE_API_ENDPOINT = "https://${KUBE_SERVER_IP}"
        KUBECONFIG_CLEAN_PATH = 'kubeconfig_clean.yaml'
        BUILD_ID = "${env.BUILD_ID ?: 'latest'}"
    }

    stages {
        stage('Build & Setup') {
            steps {
                checkout scm
                sh '''
                echo "Installing kubectl..."
                KUBE_VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
                curl -LO "https://storage.googleapis.com/kubernetes-release/release/$KUBE_VERSION/bin/linux/amd64/kubectl"
                chmod +x ./kubectl
                echo "kubectl installed."
                '''
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                    echo "Building Docker image..."
                    docker build -t $DOCKER_USER/ci-cd-app:$BUILD_ID .

                    echo "Logging into DockerHub..."
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    echo "Pushing image to DockerHub..."
                    docker push $DOCKER_USER/ci-cd-app:$BUILD_ID

                    echo "Updating deployment.yaml with image tag..."
                    sed -i "s|PLACEHOLDER_IMAGE_URL|$DOCKER_USER/ci-cd-app:$BUILD_ID|" deployment.yaml
                    '''
                }
            }
        }

        stage('Initialize Kubeconfig') {
            steps {
                withCredentials([file(credentialsId: env.KUBE_CONFIG_ID, variable: 'KUBECONFIG_SOURCE')]) {
                    sh '''
                    export PATH=$PATH:${WORKSPACE}

                    CA_DATA=$(sed -n '/certificate-authority-data:/s/.*: *//p' "${KUBECONFIG_SOURCE}" | tr -d '[:space:]')
                    CLIENT_CERT_DATA=$(sed -n '/client-certificate-data:/s/.*: *//p' "${KUBECONFIG_SOURCE}" | tr -d '[:space:]')
                    CLIENT_KEY_DATA=$(sed -n '/client-key-data:/s/.*: *//p' "${KUBECONFIG_SOURCE}" | tr -d '[:space:]')

                    if [ -z "$CLIENT_KEY_DATA" ]; then
                        echo "Client key data missing."
                        exit 1
                    fi

                    echo "${CA_DATA}" | base64 -d > ca.crt
                    echo "${CLIENT_CERT_DATA}" | base64 -d > client.crt
                    echo "${CLIENT_KEY_DATA}" | base64 -d > client.key

                    cat << EOF > ${KUBECONFIG_CLEAN_PATH}
$(sed \
    -e 's/certificate-authority-data:.*/certificate-authority: ca.crt/' \
    -e 's/client-certificate-data:.*/client-certificate: client.crt/' \
    -e 's/client-key-data:.*/client-key: client.key/' \
    "${KUBECONFIG_SOURCE}")
EOF

                    sed -i 's|server: https://.*|server: ${KUBE_API_ENDPOINT}|' ${KUBECONFIG_CLEAN_PATH}

                    echo "${KUBECONFIG_CLEAN_PATH}" > .kubeconfig_path
                    export KUBECONFIG="${KUBECONFIG_CLEAN_PATH}"

                    echo "Testing Kubernetes connection..."
                    ./kubectl cluster-info --insecure-skip-tls-verify || echo "Connection test failed."
                    '''
                }
            }
        }

        stage('Deploy Application') {
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                sh '''
                if [ -f .kubeconfig_path ]; then
                    export KUBECONFIG=$(cat .kubeconfig_path)
                    echo "Using kubeconfig: ${KUBECONFIG}"
                else
                    echo "Kubeconfig not found."
                    exit 1
                fi

                if [ -f "deployment.yaml" ]; then
                    ./kubectl apply -f deployment.yaml --insecure-skip-tls-verify
                    echo "Deployment applied."
                else
                    echo "deployment.yaml missing."
                fi

                if [ -f "service.yaml" ]; then
                    ./kubectl apply -f service.yaml --insecure-skip-tls-verify
                    echo "Service applied."
                else
                    echo "service.yaml missing."
                fi
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
        always {
            sh '''
            echo "Cleaning up workspace..."
            rm -f ./kubectl
            rm -f ${KUBECONFIG_CLEAN_PATH} ca.crt client.crt client.key .kubeconfig_path

            echo "Removing local Docker image..."
            docker rmi $DOCKER_USER/ci-cd-app:$BUILD_ID || true
            '''
        }
    }
}
