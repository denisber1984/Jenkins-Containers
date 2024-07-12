pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub')
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.build("ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${commitId}-${BUILD_NUMBER}", "-f polybot/Dockerfile polybot")
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.withRegistry('http://ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082', 'nexus-credentials') {
                        docker.image("ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${commitId}-${BUILD_NUMBER}").push()
                        docker.image("ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${commitId}-${BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }
        stage('Unittest') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.image("ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${commitId}-${BUILD_NUMBER}").inside('-u root') {
                        sh 'python3 -m pytest --junitxml=results.xml tests/test.py'
                    }
                }
                junit 'results.xml'
            }
        }
        stage('Static code linting') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.image("ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${commitId}-${BUILD_NUMBER}").inside('-u root') {
                        sh 'python3 -m pylint -f parseable --reports=no polybot/app.py polybot/bot.py polybot/img_proc.py > pylint-report.txt'
                    }
                }
                recordIssues tools: [pylint(pattern: 'pylint-report.txt')]
            }
        }
        stage('Snyk Security Scan') {
            steps {
                snykSecurity failOnIssues: true, snykTokenId: 'snyk-api-token', severity: 'high'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying the application...'
                // Add your deployment steps here
            }
        }
    }
    post {
        always {
            cleanWs()
            script {
                def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                sh "docker rmi ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${commitId}-${BUILD_NUMBER} || true"
                sh "docker rmi ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:latest || true"
            }
        }
    }
}
