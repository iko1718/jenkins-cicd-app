pipeline {
    agent {
        // Assuming you used the 'ubuntu:latest' image based on the logs
        docker {
            image 'ubuntu:latest'
            args '-u root' // Use root to ensure apt installation permissions
        }
    }
    environment {
        // Define any environment variables you need here
        K8S_VERSION = '1.29' // Using this variable for consistency
    }
    stages {
        stage('Checkout Code') {
            steps {
                // Assuming your checkout step is fine, keeping it minimal here
                checkout scm
            }
        }

        stage('Setup Tools') {
            steps {
                sh """
                echo "Installing dependencies..."
                apt-get update -y
                # Install base tools for repository setup
                apt-get install -y curl gnupg apt-transport-https

                # 1. Add Kubernetes GPG Key and Repository
                install -m 0755 -d /etc/apt/keyrings
                curl -fsSL https://pkgs.k8s.io/core:/stable:/v\${K8S_VERSION}/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
                echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v\${K8S_VERSION}/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list

                # 2. Update and Install kubectl
                apt-get update -y

                # FIX: Install the generic kubectl package.
                # Since the repository is already filtered to v1.29, this installs the latest 1.29.x version.
                # Removed the problematic '=1.29.0-00' version constraint.
                apt-get install -y kubectl
                
                # Optional: Verify installation
                kubectl version --client
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo "Deployment logic would go here..."
                // Example:
                // kubectl apply -f kubernetes/deployment.yaml
            }
        }
    }
}
