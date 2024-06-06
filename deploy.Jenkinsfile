pipeline {
    agent any

    parameters {
        string(name: 'polybot_URL', defaultValue: '', description: 'URL of the Docker image to deploy')
    }

    stages {
        stage('Deploy to EC2') {
            steps {
                echo "Deploying PolyBot app with image: ${params.polybot_URL}"
                sshagent(['ec2-ssh-credentials']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@18.197.90.144 << 'EOF'
                        docker pull ${params.polybot_URL}
                        docker stop polybot || true
                        docker rm polybot || true
                        docker run -d -p 8443:8443 --name polybot ${params.polybot_URL}
                        EOF
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Deployment to EC2 succeeded!'
        }
        failure {
            echo 'Deployment to EC2 failed!'
        }
    }
}
