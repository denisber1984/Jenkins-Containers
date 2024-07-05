pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        GITHUB_CREDENTIALS = credentials('github-credentials')
    }
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKERHUB_CREDENTIALS) {
                        def app = docker.build("denisber1984/mypolybot-app:latest", 'polybot')
                        app.push("latest")
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
