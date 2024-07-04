pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'denisber1984/mypolybot-app'
    }
    tools {
        snyk 'Snyk'  // Ensure this matches the name you gave to the Snyk tool installation
    }
    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/denisber1984/Jenkins-Containers.git'
                script {
                    gitCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh """
                        echo ${DOCKER_PASSWORD} | docker login --username ${DOCKER_USERNAME} --password-stdin
                        docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER}-${gitCommit} -t ${DOCKER_IMAGE}:latest -f polybot/Dockerfile ./polybot
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}-${gitCommit}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }
        stage('Verify Snyk Installation') {
            steps {
                sh '/usr/local/bin/snyk --version'
            }
        }
        stage('Snyk Security Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                    sh """
                        /usr/local/bin/snyk auth ${SNYK_TOKEN}
                        /usr/local/bin/snyk container test ${DOCKER_IMAGE}:latest --file=polybot/Dockerfile --severity-threshold=high
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
            cleanWs()
            script {
                try {
                    sh 'docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER}-${gitCommit}'
                    sh 'docker rmi ${DOCKER_IMAGE}:latest'
                } catch (Exception e) {
                    echo "Error during cleanup: ${e}"
                }
            }
        }
    }
}
