pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub') // Docker Hub credentials
        GITHUB_CREDENTIALS = credentials('github-credentials') // GitHub credentials
        NEXUS_CREDENTIALS = credentials('nexus-credentials') // Nexus credentials
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
                    docker.build("ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${gitCommitShort}-${env.BUILD_NUMBER}", "-f polybot/Dockerfile polybot").inside {
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
                            docker.image("ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${gitCommitShort}-${env.BUILD_NUMBER}").inside {
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
                            docker.image("ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${gitCommitShort}-${env.BUILD_NUMBER}").inside {
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
                        def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        sh """
                            snyk auth ${SNYK_TOKEN}
                            snyk container test ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${gitCommitShort}-${env.BUILD_NUMBER} --severity-threshold=high --file=polybot/Dockerfile --exclude-base-image-vulns --policy-path=./snyk-ignore.json
                        """
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD'),
                        usernamePassword(credentialsId: 'nexus-credentials-id', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')
                    ]) {
                        def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        docker.withRegistry('http://ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082', 'nexus-credentials-id') {
                            sh """
                                docker login -u ${DOCKER_USER} -p ${DOCKER_PASSWORD}
                                docker tag ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${gitCommitShort}-${env.BUILD_NUMBER} ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${gitCommitShort}-${env.BUILD_NUMBER}
                                docker push ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${gitCommitShort}-${env.BUILD_NUMBER}
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
                                docker pull ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:latest
                                docker stop mypolybot-app || true
                                docker rm mypolybot-app || true
                                docker run -d --name mypolybot-app -p 80:80 ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:latest
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
                    docker rmi ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:${gitCommitShort}-${env.BUILD_NUMBER} || true
                    docker rmi ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo:latest || true
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
