node {
    // Define the image tag using the build number
    def imageTag = "sonaliponnappaa/cicd-app:${env.BUILD_NUMBER}"

    stage('Checkout SCM') {
        checkout scm
    }
// ... (Build and Push stages)

    stage('Deploy to K8s') {
        // Run kubectl commands inside a container with kubectl installed
        docker.image('bitnami/kubectl:latest').inside('-v /var/run/docker.sock:/var/run/docker.sock') {
            withCredentials([string(
                credentialsId: 'minikube-config',
                variable: 'KUBECFG_CONTENT'
            )]) {
                def kubeconfigFile = 'kubeconfig-temp.yaml'
                def deploymentFile = 'kubernetes-deployment.yaml'

                echo "Deploying image ${imageTag} to Kubernetes..."

                // Write kubeconfig to a temporary file using shell
                sh "echo \"\${KUBECFG_CONTENT}\" > ${kubeconfigFile}"

                // Replace image placeholder with the new image tag
                sh "sed -i 's|PLACEHOLDER_IMAGE_URL|${imageTag}|g' ${deploymentFile}"

                // Apply deployment to Kubernetes cluster using the temporary kubeconfig
                sh "kubectl --kubeconfig=${kubeconfigFile} apply -f ${deploymentFile}"
                echo "Deployment triggered successfully for image: ${imageTag}"

                // Revert the deployment file and cleanup the temporary kubeconfig file
                sh "git checkout ${deploymentFile}"
                sh "rm ${kubeconfigFile}"
            }
        }
    }
}
