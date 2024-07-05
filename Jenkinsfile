pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        GITHUB_CREDENTIALS = credentials('github-credentials')
        SNYK_API = '4e793fd6-8afb-49dc-bf3f-a3170cd08483'
    }
    stages {
        stage('Checkout SCM') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: 'https://github.com/denisber1984/Jenkins-Containers.git', credentialsId: 'github-credentials']]
                ])
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        def customImage = docker.build('denisber1984/mypolybot-app:latest', 'polybot')
                        customImage.push()
                    }
                }
            }
        }
        stage('Verify Snyk Installation') {
            steps {
                sh 'snyk --version'
            }
        }
        stage('Snyk Security Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                    script {
                        def snykScan = sh(script: 'snyk auth ${SNYK_TOKEN} --api=https://snyk.io/api -d', returnStatus: true)
                        if (snykScan != 0) {
                            error 'Snyk authentication failed'
                        } else {
                            sh 'snyk test'
                        }
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                // Add your deployment steps here
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
