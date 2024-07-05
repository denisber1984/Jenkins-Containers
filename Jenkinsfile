pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub') // Docker Hub credentials
        GITHUB_CREDENTIALS = credentials('github-credentials') // GitHub credentials
        GITHUB_REPO = 'https://github.com/denisber1984/Jenkins-Containers.git' // Your GitHub repo URL
        DOCKER_IMAGE = 'denisber1984/mypolybot-app'
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                // Checkout code from GitHub
                git url: "${GITHUB_REPO}", branch: 'main', credentialsId: 'github-credentials'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    docker.build(imageTag, '-f polybot/Dockerfile ./polybot')
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    def imageTag = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        docker.image(imageTag).push()
                        docker.image(imageTag).push('latest')
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
            cleanWs()
            script {
                try {
                    def imageTag = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    echo "Cleaning up Docker images: ${imageTag} and ${DOCKER_IMAGE}:latest"
                    echo "Checking Docker daemon status..."
                    sh "docker info || true"
                    echo "Removing Docker images..."
                    sh "docker rmi ${imageTag} || true"
                    sh "docker rmi ${DOCKER_IMAGE}:latest || true"
                } catch (Exception e) {
                    echo "Error during cleanup: ${e.getMessage()}"
                    echo "Debug info: ${e}"
                    throw e
                }
            }
        }
    }
}
