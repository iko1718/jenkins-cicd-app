// This is the file that defines your build, test, and deployment process in Jenkins.
pipeline {
    // Defines where the pipeline will run (e.g., on any available agent)
    agent any

    stages {
        stage('Initialize and Deploy') {
            steps {
                // The crucial step: 'withCredentials' handles the secure secret injection.
                withCredentials([
                    // 1. Credentials Type: file (since you uploaded a Secret File)
                    // 2. credentialsId: MUST match the ID you set in Jenkins Credentials Manager (e.g., "kubeconfig")
                    // 3. variable: The name of the environment variable that will hold the path to the temporary file
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                    echo "Kubeconfig secret file successfully accessed at: $KUBECONFIG_FILE"

                    # Kubernetes tools (like kubectl) are configured by pointing the 
                    # KUBECONFIG environment variable to the file path Jenkins provided.
                    export KUBECONFIG=$KUBECONFIG_FILE

                    # --- SECURITY CHECK ---
                    # Always run a harmless command first to confirm connection.
                    echo "--- Testing Kubernetes Connection ---"
                    kubectl cluster-info

                    # --- DEPLOYMENT STEP (Replace with your actual commands) ---
                    echo "--- Starting Deployment with Kubeconfig ---"
                    # Example: Apply a manifest file using the newly configured kubeconfig
                    # kubectl apply -f my-deployment.yaml

                    echo "Deployment logic completed."
                    '''
                }
            }
        }
    }
}
