pipeline {
    agent any
    environment {
        NEXUS_CREDENTIALS_ID = 'nexus-credentials'
        NEXUS_URL = 'ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8081'
        NEXUS_REPOSITORY = 'repository/nexus-repo'
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
                    env.DOCKER_IMAGE_NAME = "${NEXUS_URL}/${NEXUS_REPOSITORY}:${gitCommitShort}"
                    docker.build("${env.DOCKER_IMAGE_NAME}", "-f polybot/Dockerfile polybot").inside {
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
                            docker.image(env.DOCKER_IMAGE_NAME).inside {
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
                            docker.image(env.DOCKER_IMAGE_NAME).inside {
                                sh 'python3 -m pylint -f parseable --reports=no polybot/*.py > pylint.log'
                            }
                        }
                    }
                    post {
                        always {
                            script {
                                sh 'cat pylint.log'
                                recordIssues(
                                    enabledForFailure: true,
                                    aggregatingResults: true,
                                    tools: [pyLint(name: 'Pylint', pattern: '**/pylint.log')]
                                )
                            }
                        }
                    }
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                        sh """
                            snyk auth ${SNYK_TOKEN}
                            snyk container test ${env.DOCKER_IMAGE_NAME} --severity-threshold=high --file=polybot/Dockerfile --exclude-base-image-vulns --policy-path=./snyk-ignore.json
                        """
                    }
                }
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                script {
                    docker.withRegistry("http://${NEXUS_URL}", NEXUS_CREDENTIALS_ID) {
                        sh 'docker push ${DOCKER_IMAGE_NAME}'
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sshagent(['ec2-ssh-credentials']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ec2-user@ec2-3-76-253-200.eu-central-1.compute.amazonaws.com '
                                docker pull ${DOCKER_IMAGE_NAME}
                                docker stop mypolybot-app || true
                                docker rm mypolybot-app || true
                                docker run -d --name mypolybot-app -p 80:80 ${DOCKER_IMAGE_NAME}
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
                    docker rmi ${env.DOCKER_IMAGE_NAME} || true
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
