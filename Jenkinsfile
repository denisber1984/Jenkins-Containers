pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub')
        GITHUB_CREDENTIALS = credentials('github-credentials')
        SNYK_TOKEN = credentials('snyk-api-token')
        SNYK_API = 'https://snyk.io/api' // Set to your custom endpoint if different
    }
    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }
    tools {
        snyk 'Snyk'  // This should match the name you gave to the Snyk tool installation
    }
    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'github-credentials', url: 'https://github.com/denisber1984/Jenkins-Containers.git'
                script {
                    gitCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_HUB_CREDENTIALS_PSW', usernameVariable: 'DOCKER_HUB_CREDENTIALS_USR')]) {
                    sh '''
                    docker login --username $DOCKER_HUB_CREDENTIALS_USR --password $DOCKER_HUB_CREDENTIALS_PSW
                    docker build -t denisber1984/mypolybot-app:${BUILD_NUMBER}-${gitCommit} -t denisber1984/mypolybot-app:latest -f polybot/Dockerfile ./polybot
                    docker push denisber1984/mypolybot-app:${BUILD_NUMBER}-${gitCommit}
                    docker push denisber1984/mypolybot-app:latest
                    '''
                }
            }
        }
        stage('Snyk Security Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                    sh '''
                    snyk auth ${SNYK_TOKEN}
                    snyk container test denisber1984/mypolybot-app:latest --file=polybot/Dockerfile --severity-threshold=high
                    '''
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
                sh 'docker rmi denisber1984/mypolybot-app:${BUILD_NUMBER}-${gitCommit}'
                sh 'docker rmi denisber1984/mypolybot-app:latest'
            }
        }
    }
}
