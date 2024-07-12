pipeline {
    agent any
    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        APP_IMAGE_NAME = 'app-image'
        WEB_IMAGE_NAME = 'web-image'
        DOCKER_COMPOSE_FILE = 'compose.yaml'
        DOCKER_REPO = 'denisber1984/polybot_app'
        NEXUS_REPO = "nexus-repo"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082"
        AWS_ELASTIC_IP = '3.76.72.36'
        SSH_KEY_PATH = '/path/to/your/aws-key.pem'
        NEXUS_CREDENTIALS_ID = 'nexus-credentials'
        DOCKERHUB_CREDENTIALS = 'dockerhub'
        SNYK_API_TOKEN = 'snyk-api-token'
    }

    stages {
        stage('Checkout and Extract Git Commit Hash') {
            steps {
                checkout scm
                script {
                    def gitCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.GIT_COMMIT = gitCommit
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    // Ensure docker-compose is available
                    sh 'docker-compose --version'
                    // Build Docker image using docker-compose
                    withDockerContainer(image: 'docker/compose:1.29.2', args: '-u root:root --entrypoint=""') {
                        sh """
                            docker-compose -f ${DOCKER_COMPOSE_FILE} build
                        """
                    }
                }
            }
        }
        stage('Install Python Requirements') {
            steps {
                script {
                    sh """
                        pip install --upgrade pip
                        pip install pytest unittest2 pylint flask telebot Pillow loguru matplotlib
                    """
                }
            }
        }
        stage('Static Code Linting and Unittest') {
            parallel {
                stage('Static code linting') {
                    steps {
                        script {
                            sh """
                                python -m pylint -f parseable --reports=no polybot/*.py > pylint.log
                                cat pylint.log
                            """
                        }
                    }
                }
                stage('Unittest') {
                    steps {
                        script {
                            sh 'python -m pytest --junitxml results.xml polybot/test'
                        }
                    }
                }
            }
        }
        stage('Security Vulnerability Scanning') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                        sh """
                            snyk auth $SNYK_TOKEN
                            snyk container test ${APP_IMAGE_NAME}:latest --severity-threshold=high || exit 0
                        """
                    }
                }
            }
        }
        stage('Login, Tag, and Push Images') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'USER', passwordVariable: 'PASS'),
                    usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')
                ]) {
                    script {
                        def gitTag = "${env.GIT_COMMIT}"
                        env.IMAGE_TAG = "v1.0.0-${BUILD_NUMBER}-${gitTag}"
                        sh """
                            cd polybot
                            docker login -u ${USER} -p ${PASS} ${NEXUS_PROTOCOL}://${NEXUS_URL}/repository/${NEXUS_REPO}
                            docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}
                            docker tag ${APP_IMAGE_NAME}:latest ${NEXUS_URL}/${APP_IMAGE_NAME}:${env.IMAGE_TAG}
                            docker tag ${WEB_IMAGE_NAME}:latest ${NEXUS_URL}/${WEB_IMAGE_NAME}:${env.IMAGE_TAG}
                            docker tag ${APP_IMAGE_NAME}:latest ${DOCKER_REPO}:${APP_IMAGE_NAME}-${env.IMAGE_TAG}
                            docker tag ${WEB_IMAGE_NAME}:latest ${DOCKER_REPO}:${WEB_IMAGE_NAME}-${env.IMAGE_TAG}
                            docker push ${NEXUS_URL}/${APP_IMAGE_NAME}:${env.IMAGE_TAG}
                            docker push ${NEXUS_URL}/${WEB_IMAGE_NAME}:${env.IMAGE_TAG}
                            docker push ${DOCKER_REPO}:${APP_IMAGE_NAME}-${env.IMAGE_TAG}
                            docker push ${DOCKER_REPO}:${WEB_IMAGE_NAME}-${env.IMAGE_TAG}
                        """
                    }
                }
            }
        }
        stage('Deployment on EC2') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        sh """
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker pull ${DOCKER_REPO}:${APP_IMAGE_NAME}-${env.IMAGE_TAG}"
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker pull ${DOCKER_REPO}:${WEB_IMAGE_NAME}-${env.IMAGE_TAG}"
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker stop my-app-container my-web-container"
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker rm -f my-app-container my-web-container"
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker run -d -p 843:8443 --name my-app-container ${DOCKER_REPO}:${APP_IMAGE_NAME}-${env.IMAGE_TAG}"
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker run -d -p 844:8444 --name my-web-container ${DOCKER_REPO}:${WEB_IMAGE_NAME}-${env.IMAGE_TAG}"
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                junit 'results.xml'
                recordIssues enabledForFailure: true, aggregatingResults: true
                recordIssues tools: [pyLint(pattern: 'pylint.log')]
                cleanWs(cleanWhenNotBuilt: false, deleteDirs: true, notFailBuild: true)
                sh "docker image prune -f"
            }
        }
    }
}
