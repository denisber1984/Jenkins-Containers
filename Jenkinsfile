pipeline {
    agent any

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        DOCKER_IMAGE = "ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082/repository/nexus-repo"
        GIT_COMMIT_HASH = ''
        SHARED_LIB = 'jenkins-shared-library@main'
    }

    stages {
        stage('Checkout Shared Library') {
            steps {
                script {
                    library SHARED_LIB
                }
            }
        }

        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('Setup Environment') {
            steps {
                script {
                    GIT_COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${GIT_COMMIT_HASH}", "-f polybot/Dockerfile polybot")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withDockerRegistry([ credentialsId: 'nexus-credentials', url: 'http://ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082']) {
                    script {
                        docker.image("${DOCKER_IMAGE}:${GIT_COMMIT_HASH}").push()
                        docker.image("${DOCKER_IMAGE}:${GIT_COMMIT_HASH}").push('latest')
                    }
                }
            }
        }

        stage('Parallel Test and Linting') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        script {
                            docker.image("${DOCKER_IMAGE}:${GIT_COMMIT_HASH}").inside {
                                sh 'python3 -m pytest --junitxml=results.xml tests/test.py'
                            }
                        }
                    }
                    post {
                        always {
                            junit 'results.xml'
                        }
                    }
                }

                stage('Static Code Linting') {
                    steps {
                        script {
                            docker.image("${DOCKER_IMAGE}:${GIT_COMMIT_HASH}").inside {
                                sh 'python3 -m pylint -f parseable --reports=no polybot/app.py polybot/bot.py polybot/img_proc.py > pylint.log'
                            }
                        }
                    }
                    post {
                        always {
                            recordIssues tools: [pylint(pattern: '**/pylint.log')]
                        }
                    }
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                    script {
                        docker.image("${DOCKER_IMAGE}:${GIT_COMMIT_HASH}").inside {
                            sh 'snyk auth $SNYK_TOKEN'
                            sh 'snyk container test ${DOCKER_IMAGE}:${GIT_COMMIT_HASH} --severity-threshold=high --file=polybot/Dockerfile --exclude-base-image-vulns --policy-path=./snyk-ignore.json'
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sshagent(['ec2-ssh-credentials']) {
                    script {
                        def remoteDockerLogin = """
                            docker login -u admin -p ${env.NEXUS_CREDENTIALS_PSW} http://ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082
                            docker pull ${DOCKER_IMAGE}:latest
                            docker stop mypolybot-app || true
                            docker rm mypolybot-app || true
                            docker run -d --name mypolybot-app -p 80:80 ${DOCKER_IMAGE}:latest
                        """
                        sh "ssh -o StrictHostKeyChecking=no ec2-user@ec2-3-125-33-156.eu-central-1.compute.amazonaws.com '${remoteDockerLogin}'"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
