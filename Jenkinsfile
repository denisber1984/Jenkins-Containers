pipeline {
    agent any
    environment {
        NEXUS_CREDENTIALS_ID = 'nexus-credentials' // Ensure this ID matches the ID used when adding the Nexus credentials to Jenkins
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
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.DOCKER_IMAGE_NAME = "${NEXUS_URL}/${NEXUS_REPOSITORY}:${commitId}"

                    sh 'docker build -t $DOCKER_IMAGE_NAME -f polybot/Dockerfile polybot'
                }
            }
        }
        stage('Test Docker Image') {
            parallel {
                stage('Unittest') {
                    steps {
                        script {
                            sh 'docker inspect -f . $DOCKER_IMAGE_NAME'
                            docker.image(env.DOCKER_IMAGE_NAME).inside {
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
                stage('Static code linting') {
                    steps {
                        script {
                            sh 'docker inspect -f . $DOCKER_IMAGE_NAME'
                            docker.image(env.DOCKER_IMAGE_NAME).inside {
                                sh 'python3 -m pylint -f parseable --reports=no polybot/app.py polybot/bot.py polybot/img_proc.py | tee pylint.log'
                            }
                        }
                    }
                    post {
                        always {
                            recordIssues tools: [pylint(pattern: 'pylint.log')]
                        }
                    }
                }
            }
        }
        stage('Snyk Security Scan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                        sh '''
                            snyk auth $SNYK_TOKEN
                            snyk container test $DOCKER_IMAGE_NAME --severity-threshold=high --file=polybot/Dockerfile --exclude-base-image-vulns --policy-path=./snyk-ignore.json
                        '''
                    }
                }
            }
        }
        stage('Push Docker Image to Nexus') {
            steps {
                script {
                    docker.withRegistry("http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}", NEXUS_CREDENTIALS_ID) {
                        sh 'docker push $DOCKER_IMAGE_NAME'
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    sshagent(['ec2-ssh-credentials']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ec2-user@ec2-18-199-84-131.eu-central-1.compute.amazonaws.com << EOF
                            docker pull $DOCKER_IMAGE_NAME
                            docker run -d --name polybot_app -p 80:80 $DOCKER_IMAGE_NAME
                            EOF
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
