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
        // Fix: Add --entrypoint='' to bypass the bitnami/kubectl image's default entrypoint
        docker.image('bitnami/kubectl:latest').inside("--entrypoint='' -v /var/run/docker.sock:/var/run/docker.sock") {
            withCredentials([string(
                credentialsId: 'minikube-config',
                variable: 'KUBECFG_CONTENT'
            )]) {
                def kubeconfigFileEncoded = 'kubeconfig-encoded.b64'
                def deploymentFile = 'kubernetes-deployment.yaml'

                echo "Deploying image ${imageTag} to Kubernetes..."

                // 1. Convert the raw secret text (KUBECFG_CONTENT) into a base64 encoded string
                def kubeconfigBase64 = KUBECFG_CONTENT.trim().encodeBase64().toString()

                // 2. Write the SINGLE-LINE base64 string to a temporary file. This is immune to YAML corruption.
                writeFile file: kubeconfigFileEncoded, text: kubeconfigBase64

                // Replace image placeholder with the new image tag
                sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${imageTag}|g' ${deploymentFile}"

                // 3. Decode the file and pipe the content directly into kubectl's stdin using the '--kubeconfig=-' flag.
                // This is the most reliable way to handle Kubeconfig secrets.
                sh "base64 -d ${kubeconfigFileEncoded} | kubectl --kubeconfig=- apply -f ${deploymentFile}"
                echo "Deployment triggered successfully for image: ${imageTag}"

                // Cleanup
                sh "git checkout ${deploymentFile}"
                sh "rm ${kubeconfigFileEncoded}"
            }
        }
    }
}
