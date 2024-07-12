pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub') // Docker Hub credentials
        GITHUB_CREDENTIALS = credentials('github-credentials') // GitHub credentials
        NEXUS_CREDENTIALS = credentials('nexus-credentials') // Nexus credentials

        NEXUS_CREDENTIALS_ID = 'nexus-credentials'
        NEXUS_REPO = "nexus-repo"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "3.76.72.36:8082"
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
                    docker.build("${NEXUS_PROTOCOL}://${NEXUS_URL}/repository/${NEXUS_REPO}:${gitCommitShort}-${env.BUILD_NUMBER}", "-f polybot/Dockerfile polybot").inside {
                        sh 'echo Docker image built successfully'
                    }
                }
            }
        }

        stage('Parallel Stages') {
            parallel {
                stage('Unittest') {
                    steps {
                        script {
                            def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                            docker.image("${NEXUS_PROTOCOL}://${NEXUS_URL}/repository/${NEXUS_REPO}:${gitCommitShort}-${env.BUILD_NUMBER}").inside {
                                sh 'python3 -m pytest --junitxml=${WORKSPACE}/results.xml tests/test.py'
                            }
                        }
                    }
                    post {
                        always {
                            script {
                                junit allowEmptyResults: true, testResults: '**/results.xml'
                            }
                        }
                    }
                }

                stage('Static code linting') {
                    steps {
                        script {
                            def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                            docker.image("${NEXUS_PROTOCOL}://${NEXUS_URL}/repository/${NEXUS_REPO}:${gitCommitShort}-${env.BUILD_NUMBER}").inside {
                                sh 'python3 -m pylint -f parseable --reports=no polybot/*.py > pylint.log'
                            }
                        }
                    }
                    post {
                        always {
                            script {
                                sh 'cat pylint.log'
                                recordI
