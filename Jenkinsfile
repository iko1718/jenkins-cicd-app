node {
    def imageTag = "sonaliponnappaa/cicd-app:${env.BUILD_NUMBER}"

    stage('Checkout SCM') {
        checkout scm
    }

    stage('Build Image') {
        docker.image('docker:latest').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
            echo "Building Docker image: ${imageTag}"
            sh "docker build -t ${imageTag} ."
        }
    }

    stage('Push Image') {
        docker.image('docker:latest').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
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

    // Optional: Uncomment this stage once kubectl is installed and configured
    /*
    stage('Deploy to K8s') {
        docker.image('bitnami/kubectl:latest').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
            withCredentials([string(
                credentialsId: 'minikube-config',
                variable: 'KUBECFG_CONTENT'
            )]) {
                def kubeconfigFile = 'kubeconfig-temp.yaml'
                def deploymentFile = 'kubernetes-deployment.yaml'

                echo "Deploying image ${imageTag} to Kubernetes..."

                // Write kubeconfig to file
                writeFile file: kubeconfigFile, text: "${KUBECFG_CONTENT}"

                // Replace image placeholder
                sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${imageTag}|g' ${deploymentFile}"

                // Apply deployment
                sh "kubectl --kubeconfig=${kubeconfigFile} apply -f ${deploymentFile}"

                // Revert and cleanup
                sh "git checkout ${deploymentFile}"
                sh "rm ${kubeconfigFile}"
            }
        }
    }
    */
}
