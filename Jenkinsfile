pipeline {
    agent any

    stages {
        stage('Debug Kubeconfig') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_SOURCE')
                ]) {
                    sh """
                    echo "=== COMPLETE KUBECONFIG ANALYSIS ==="
                    echo "1. File location: \$KUBECONFIG_SOURCE"
                    echo "2. File size:"
                    wc -c "\$KUBECONFIG_SOURCE"
                    echo "3. File permissions:"
                    ls -la "\$KUBECONFIG_SOURCE"
                    echo "4. First 30 lines:"
                    head -30 "\$KUBECONFIG_SOURCE"
                    echo "5. Certificate data lines with line numbers:"
                    grep -n "certificate-authority-data\\|client-certificate-data\\|client-key-data" "\$KUBECONFIG_SOURCE"
                    echo "6. Let's see the exact content of certificate-authority-data line:"
                    grep "certificate-authority-data" "\$KUBECONFIG_SOURCE" | cat -A
                    echo "7. Character count of base64 data:"
                    grep "certificate-authority-data" "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | wc -c
                    echo "8. First 100 characters of base64 data:"
                    grep "certificate-authority-data" "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | head -c 100
                    echo ""
                    echo "9. Last 100 characters of base64 data:"
                    grep "certificate-authority-data" "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | tail -c 100
                    echo ""
                    echo "10. Let's try different extraction methods:"
                    echo "Method 1 (awk):"
                    grep "certificate-authority-data" "\$KUBECONFIG_SOURCE" | awk '{print \$2}' | head -c 50
                    echo ""
                    echo "Method 2 (cut):"
                    grep "certificate-authority-data" "\$KUBECONFIG_SOURCE" | cut -d: -f2 | head -c 50
                    echo ""
                    echo "Method 3 (sed):"
                    grep "certificate-authority-data" "\$KUBECONFIG_SOURCE" | sed 's/.*certificate-authority-data://' | head -c 50
                    echo ""
                    """
                }
            }
        }
    }
}
