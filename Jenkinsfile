pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub')
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
        NEXUS_URL = 'ec2-3-76-72-36.eu-central-1.compute.amazonaws.com:8082'
        NEXUS_REPO = 'nexus-repo'
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
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
                    def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    docker.build("${NEXUS_URL}/repository/${NEXUS_REPO}:${commitId}-${env.BUILD_NUMBER}", "-f polybot/Dockerfile polybot")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    docker.withRegistry("http://${NEXUS_URL}", 'nexus-credentials') {
                        docker.image("${NEXUS_URL}/repository/${NEXUS_REPO}:${commitId}-${env.BUILD_NUMBER}").push()
                        docker.image("${NEXUS_URL}/repository/${NEXUS_REPO}:${commitId}-${env.BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }

        stage('Unittest') {
            steps {
                script {
                    def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    docker.image("${NEXUS_URL}/repository/${NEXUS_REPO}:${commitId}-${env.BUILD_NUMBER}").inside {
                        sh 'python3 -m pytest --junitxml=results.xml tests/test.py'
                    }
                }
                junit 'results.xml'
            }
        }

        stage('Static code linting') {
            steps {
                script {
                    def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    docker.image("${NEXUS_URL}/repository/${NEXUS_REPO}:${commitId}-${env.BUILD_NUMBER}").inside {
                        sh 'python3 -m pylint -f parseable --reports=no polybot/app.py polybot/bot.py polybot/img_proc.py > pylint.log || true'
                    }
                }
                recordIssues tools: [pylint(pattern: '**/pylint.log')]
            }
        }

        stage('Snyk Security Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                    script {
                        def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                        sh "snyk auth ${SNYK_TOKEN}"
                        sh "snyk container test ${NEXUS_URL}/repository/${NEXUS_REPO}:${commitId}-${env.BUILD_NUMBER} --severity-threshold=high --file=polybot/Dockerfile --exclude-base-image-vulns --policy-path=./snyk-ignore.json"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sshagent(['ec2-ssh-credentials']) {
                    script {
                        def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                        sh """
                            ssh -o StrictHostKeyChecking=no ec2-user@ec2-3-76-253-200.eu-central-1.compute.amazonaws.com '
                                docker login -u admin -p ${NEXUS_CREDENTIALS_PSW} http://${NEXUS_URL}
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

    post {
        always {
            script {
                def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                sh """
                    docker rmi ${NEXUS_URL}/repository/${NEXUS_REPO}:${commitId}-${env.BUILD_NUMBER} || true
                    docker rmi ${NEXUS_URL}/repository/${NEXUS_REPO}:latest || true
                """
            }
            cleanWs()
        }
        success {
            script {
                if (fileExists("${WORKSPACE}/results.xml")) {
                    junit allowEmptyResults: true, testResults: '**/results.xml'
                } else {
                    echo "No results.xml file found, skipping JUnit reporting."
                }
            }
        }
    }
}
