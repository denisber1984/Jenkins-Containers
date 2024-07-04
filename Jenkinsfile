pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub')
        GITHUB_CREDENTIALS = credentials('github-credentials')
        GITHUB_REPO = 'https://github.com/denisber1984/Jenkins-Containers.git'
        DOCKER_IMAGE = 'denisber1984/mypolybot-app'
    }

    stages {
        stage('Checkout') {
            steps {
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

        stage('Snyk Security Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                    sh """
                        snyk auth ${SNYK_TOKEN}
                        # Scan the Docker image with severity threshold
                        snyk container test ${DOCKER_IMAGE}:latest --file=polybot/Dockerfile --severity-threshold=high
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploy stage - customize as needed'
            }
        }
    }

    post {
        always {
            script {
                sh """
                    docker rmi ${DOCKER_IMAGE}:${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT} || true
                    docker rmi ${DOCKER_IMAGE}:latest || true
                """
            }
            cleanWs()
        }
    }
}

