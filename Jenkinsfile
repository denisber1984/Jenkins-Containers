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

    stages {
        stage('Checkout') {
            steps {
                // Checkout code from GitHub
                git url: "${GITHUB_REPO}", branch: 'main', credentialsId: 'github-credentials'
                script {
                    sh 'git config --global --add safe.directory /var/lib/jenkins/workspace/mypolybot-pipeline'
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    def imageTag = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                    def latestTag = "latest"

                    sh """
                        docker login --username ${env.DOCKER_HUB_CREDENTIALS_USR} --password ${env.DOCKER_HUB_CREDENTIALS_PSW}
                        docker build -t ${DOCKER_IMAGE}:${imageTag} -t ${DOCKER_IMAGE}:${latestTag} -f polybot/Dockerfile ./polybot
                        docker push ${DOCKER_IMAGE}:${imageTag}
                        docker push ${DOCKER_IMAGE}:${latestTag}
                    """
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
        }
    }
}
