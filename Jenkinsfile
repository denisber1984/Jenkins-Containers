pipeline {
    agent any
    options {
        // Keeps builds for the last 30 days.
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        // Prevents concurrent builds of the same pipeline.
        disableConcurrentBuilds()
        // Adds timestamps to the log output.
        timestamps()
    }

    environment {
        // Define environment variables
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
        stage('Use Shared Library Code') {
            steps {
                // Use the helloWorld function from the shared library
                helloWorld('DevOps Student')
            }
        }
        stage('Checkout and Extract Git Commit Hash') {
            steps {
                // Checkout code
                checkout scm
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image using docker-compose
                    sh """
                        docker-compose -f ${DOCKER_COMPOSE_FILE} build
                    """
                }
            }
        }
        stage('Install Python Requirements') {
            steps {
                script {
                    // Install Python dependencies
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
                            // Run python code analysis
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
                            // Run unittests
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
                        // Scan the image
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
                        // Retrieve the Git commit hash
                        sh(script: 'git rev-parse --short HEAD > gitCommit.txt')
                        def GITCOMMIT = readFile('gitCommit.txt').trim()
                        def GIT_TAG = "${GITCOMMIT}"
                        // Set IMAGE_TAG as an environment variable
                        env.IMAGE_TAG = "v1.0.0-${BUILD_NUMBER}-${GIT_TAG}"
                        // Login to Dockerhub / Nexus repo ,tag, and push images
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
                        // Login to Dockerhub private repo
                        sh """
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                        """

                        // Pull the app image from Dockerhub
                        sh """
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker pull ${DOCKER_REPO}:${APP_IMAGE_NAME}-${env.IMAGE_TAG}"
                        """

                        // Pull the web image from Dockerhub
                        sh """
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker pull ${DOCKER_REPO}:${WEB_IMAGE_NAME}-${env.IMAGE_TAG}"
                        """

                        // Stop the previous my-app-container
                        sh """
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker stop my-app-container my-web-container"
                        """

                        // Remove the previous my-app-container
                        sh """
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker rm -f my-app-container my-web-container"
                        """

                        // Run the app image at 8443
                        sh """
                            ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${AWS_ELASTIC_IP} "docker run -d -p 843:8443 --name my-app-container ${DOCKER_REPO}:${APP_IMAGE_NAME}-${env.IMAGE_TAG}"
                        """

                        // Run the web image at 8444
                        sh """
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
                // Process the test results using the JUnit plugin
                junit 'results.xml'

                // Process the pylint report using the Warnings Plugin
                recordIssues enabledForFailure: true, aggregatingResults: true
                recordIssues tools: [pyLint(pattern: 'pylint.log')]

                // Clean up workspace after build
                cleanWs(cleanWhenNotBuilt: false,
                        deleteDirs: true,
                        notFailBuild: true)

                // Clean up unused dangling Docker images
                sh """
                    docker image prune -f
                """
            }
        }
    }
}
