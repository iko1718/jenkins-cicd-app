node {
    // Define the image tag using the build number
    def imageTag = "sonaliponnappaa/cicd-app:${env.BUILD_NUMBER}"

    stage('Checkout SCM') {
        checkout scm
    }

    stage('Build Image') {
        // Run docker client inside a separate container, mounting the socket for communication
        docker.image('docker:latest').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
            // Fix: Use DOCKER_CONFIG in the workspace to prevent "permission denied" error
            withEnv(["DOCKER_CONFIG=${pwd()}/.docker"]) {
                sh 'mkdir -p .docker'
                echo "Building Docker image: ${imageTag}"
                sh "docker build -t ${imageTag} ."
            }
        }
    }

    stage('Push Image') {
        docker.image('docker:latest').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
            // Fix: Use DOCKER_CONFIG in the workspace for secure login
            withEnv(["DOCKER_CONFIG=${pwd()}/.docker"]) {
                sh 'mkdir -p .docker'
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    echo "Logging into Docker Hub and pushing image: ${imageTag}"
                    sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USER --password-stdin"
                    
                    sh "docker push ${imageTag}"
                    sh "docker tag ${imageTag} sonaliponnappaa/cicd-app:latest"
                    sh "docker push sonaliponnappaa/cicd-app:latest"
                    sh "docker logout"
                }
            }
        }
    }

    stage('Deploy to K8s') {
        // Using bitnami/kubectl image for Kubernetes operations
        docker.image('bitnami/kubectl:latest').inside("--entrypoint='' -v /var/run/docker.sock:/var/run/docker.sock") {
            withCredentials([string(
                credentialsId: 'minikube-config',
                variable: 'KUBECFG_CONTENT' // This is the raw kubeconfig content
            )]) {
                def deploymentFile = 'kubernetes-deployment.yaml'
                def kubeconfigFilePath = "${pwd()}/.kube/config"

                echo "Deploying image ${imageTag} to Kubernetes..."

                // 1. Write the raw KUBECFG_CONTENT directly to a temporary file
                writeFile file: '.kube/config', text: KUBECFG_CONTENT.trim()

                // 2. Set KUBECONFIG environment variable to use the file.
                withEnv(["KUBECONFIG=${kubeconfigFilePath}"]) {
                    
                    // 3. Replace image placeholder
                    sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${imageTag}|g' ${deploymentFile}"

                    // 4. Apply deployment (kubectl automatically uses $KUBECONFIG)
                    sh """
                    kubectl apply -f ${deploymentFile}

                    echo "Deployment triggered successfully for image: ${imageTag}"

                    # Cleanup
                    git checkout ${deploymentFile}
                    rm -rf .kube
                    """
                }
            }
        }
    }
}
