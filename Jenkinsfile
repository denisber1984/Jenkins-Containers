pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub') // Updated to use the correct credentials ID
        GITHUB_REPO = 'https://github.com/denisber1984/Jenkins-Containers.git' // Your GitHub repo URL
        DOCKER_IMAGE = 'denisber1984/mypolybot-app:latest'
    }

    stages {
        stage('Checkout') {
            steps {
                // Checkout code from GitHub
                git url: "${GITHUB_REPO}"
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}", '-f Polybot/Dockerfile ./Polybot')
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        docker.image("${DOCKER_IMAGE}").push()
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploy stage - customize as needed'
                // Add deployment steps here if necessary
            }
        }
    }

    post {
        always {
            node {
                cleanWs()
            }
        }
    }
}
