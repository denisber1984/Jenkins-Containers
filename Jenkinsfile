pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub')
        GITHUB_CREDENTIALS = credentials('github-credentials')
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
        NEXUS_REPO = "nexus-repo"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082"
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
                    def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.build("${NEXUS_URL}/repository/${NEXUS_REPO}:${gitCommitShort}-${env.BUILD_NUMBER}", "-f polybot/Dockerfile polybot")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD'),
                        usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')
                    ]) {
                        def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        docker.withRegistry("${NEXUS_PROTOCOL}://${NEXUS_URL}", 'nexus-credentials') {
                            sh """
                                docker login -u ${NEXUS_USER} -p ${NEXUS_PASSWORD} ${NEXUS_PROTOCOL}://${NEXUS_URL}
                                docker tag ${NEXUS_URL}/repository/${NEXUS_REPO}:${gitCommitShort}-${env.BUILD_NUMBER} ${NEXUS_URL}/repository/${NEXUS_REPO}:${gitCommitShort}-${env.BUILD_NUMBER}
                                docker push ${NEXUS_URL}/repository/${NEXUS_REPO}:${gitCommitShort}-${env.BUILD_NUMBER}
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sshagent(['ec2-ssh-credentials']) {
                        def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        sh """
                            ssh -o StrictHostKeyChecking=no ec2-user@ec2-3-76-253-200.eu-central-1.compute.amazonaws.com '
                                docker pull ${NEXUS_URL}/repository/${NEXUS_REPO}:latest
                                docker stop mypolybot-app || true
                                docker rm mypolybot-app || true
                                docker run -d --name mypolybot-app -p 80:80 ${NEXUS_URL}/repository/${NEXUS_REPO}:latest
                            '
                        """
                    }
                }
            }
        }
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    post {
        always {
            script {
                def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                // Clean up the built Docker images from the disk
                sh """
                    docker rmi ${NEXUS_URL}/repository/${NEXUS_REPO}:${gitCommitShort}-${env.BUILD_NUMBER} || true
                    docker rmi ${NEXUS_URL}/repository/${NEXUS_REPO}:latest || true
                """
            }
            // Clean the workspace
            cleanWs()
        }
        success {
            script {
                // Check if the results.xml file exists before trying to access it
                if (fileExists("${WORKSPACE}/results.xml")) {
                    junit allowEmptyResults: true, testResults: '**/results.xml'
                } else {
                    echo "No results.xml file found, skipping JUnit reporting."
                }
            }
        }
    }
}
