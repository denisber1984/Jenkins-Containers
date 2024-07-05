pipeline {
    agent any

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("denisber1984/mypolybot-app:${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}", "polybot/Dockerfile").inside {
                        sh 'echo Docker image built successfully'
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('', 'dockerhub') {
                        docker.image("denisber1984/mypolybot-app:${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}").push()
                        docker.image("denisber1984/mypolybot-app:${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}").push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploy stage - customize as needed'
            }
        }
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub') // Docker Hub credentials
        GITHUB_CREDENTIALS = credentials('github-credentials') // GitHub credentials
    }

    post {
        always {
            script {
                // Clean up the built Docker images from the disk
                sh """
                    docker rmi denisber1984/mypolybot-app:${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT} || true
                    docker rmi denisber1984/mypolybot-app:latest || true
                """
            }
            // Clean the workspace
            cleanWs()
        }
    }
}
