pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '-u root:root' // Use root user to ensure permissions
        }
    }
    environment {
        SNYK_TOKEN = credentials('snyk-api-token')
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/denisber1984/Jenkins-Containers.git'
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        def app = docker.build("denisber1984/mypolybot-app:${env.BUILD_NUMBER}")
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
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
                    sh 'snyk auth $SNYK_TOKEN'
                    sh 'snyk container test denisber1984/mypolybot-app:latest --file=polybot/Dockerfile --severity-threshold=high'
                }
            }
        }
        stage('Deploy') {
            steps {
                // Your deployment steps here
                echo 'Deploying the application...'
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
