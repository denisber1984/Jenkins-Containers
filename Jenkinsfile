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
                    def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.build("denisber1984/mypolybot-app:${env.BUILD_NUMBER}-${gitCommitShort}", "-f polybot/Dockerfile polybot").inside {
                        sh 'echo Docker image built successfully'
                    }
                }
            }
        }

        stage('Unittest') {
            steps {
                script {
                    // Install required packages
                    sh 'pip3 install pytest unittest2'
                    // Run unittests
                    sh 'python3 -m pytest --junitxml results.xml tests/*.py'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'results.xml'
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                        def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        sh """
                            snyk auth ${SNYK_TOKEN}
                            snyk container test denisber1984/mypolybot-app:62-62b1591 --severity-threshold=high --file=polybot/Dockerfile --exclude-base-image-vulns
                            snyk ignore --id=SNYK-DEBIAN12-ZLIB-6008963
                            snyk ignore --id=SNYK-DEBIAN12-GIT-6846203
                        """
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.withRegistry('', 'dockerhub') {
                        docker.image("denisber1984/mypolybot-app:${env.BUILD_NUMBER}-${gitCommitShort}").push()
                        docker.image("denisber1984/mypolybot-app:${env.BUILD_NUMBER}-${gitCommitShort}").push('latest')
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
                def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                // Clean up the built Docker images from the disk
                sh """
                    docker rmi denisber1984/mypolybot-app:${env.BUILD_NUMBER}-${gitCommitShort} || true
                    docker rmi denisber1984/mypolybot-app:latest || true
                """
            }
            // Clean the workspace
            cleanWs()
        }
    }
}
