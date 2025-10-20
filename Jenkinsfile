node {
    // Define the image tag using the build number
    def imageTag = "sonaliponnappaa/cicd-app:${env.BUILD_NUMBER}"
    def deploymentFile = 'kubernetes-deployment.yaml'

    stage('Checkout SCM') {
        checkout scm
    }

    stage('Build Image') {
        docker.image('docker:latest').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
            withEnv(["DOCKER_CONFIG=${pwd()}/.docker"]) {
                sh 'mkdir -p .docker'
                echo "Building Docker image: ${imageTag}"
                sh "docker build -t ${imageTag} ."
            }
        }
    }

    stage('Push Image') {
        docker.image('docker:latest').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
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
                    // FIX: Tag and push the :latest tag correctly
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
                def kubeconfigFilePath = "${pwd()}/.kube/config"

                // Use try-finally to ensure cleanup runs even if deployment fails
                try {
                    echo "Deploying image ${imageTag} to Kubernetes..."

                    // 1. Explicitly create directory and write the credential content
                    sh 'mkdir -p .kube' 
                    // .trim() removes leading/trailing whitespace, which is crucial for YAML
                    writeFile file: '.kube/config', text: KUBECFG_CONTENT.trim()
                    sh 'chmod 600 .kube/config'

                    // 2. Set KUBECONFIG environment variable and run kubectl
                    withEnv(["KUBECONFIG=${kubeconfigFilePath}"]) {
                        
                        // 3. Replace image placeholder
                        sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${imageTag}|g' ${deploymentFile}"

                        // 4. Apply deployment
                        sh "kubectl apply -f ${deploymentFile}"

                        echo "Deployment triggered successfully for image: ${imageTag}"
                    }
                } finally {
                    // Cleanup: reset the deployment file and remove the config file/directory
                    sh "git checkout ${deploymentFile}"
                    sh "rm -rf .kube"
                }
            }
        }
    }
}
