@Library('jenkins-shared-library') _

pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

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
        stage('Say Hello') {
            steps {
                script {
                    myLib.sayHello('Den')
                }
            }
        }

        stage('Checkout SCM') {
            steps {
                checkout scm
                script {
                    sh 'git config --global --add safe.directory /var/lib/jenkins/workspace/mypolybot-pipeline'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    myLib.buildDockerImage(NEXUS_URL, NEXUS_REPO, commitId, env.BUILD_NUMBER)
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    myLib.pushDockerImage(NEXUS_URL, NEXUS_REPO, commitId, env.BUILD_NUMBER)
                }
            }
        }

        stage('Unittest') {
            steps {
                script {
                    def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    myLib.runUnitTests(NEXUS_URL, NEXUS_REPO, commitId, env.BUILD_NUMBER)
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                    script {
                        def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                        echo "Running Snyk scan on image: ${NEXUS_URL}/repository/${NEXUS_REPO}:${commitId}-${env.BUILD_NUMBER}"
                        sh "snyk auth ${SNYK_TOKEN}"
                        sh """
                            snyk container test ${NEXUS_URL}/repository/${NEXUS_REPO}:${commitId}-${env.BUILD_NUMBER} \
                            --severity-threshold=high --file=polybot/Dockerfile \
                            --exclude-base-image-vulns --policy-path=./snyk-ignore.json || true
                        """
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sshagent(['ec2-ssh-credentials']) {
                    script {
                        def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                        myLib.deployApplication(NEXUS_URL, NEXUS_REPO, commitId, env.BUILD_NUMBER, NEXUS_CREDENTIALS_PSW, 'ec2-3-125-33-156.eu-central-1.compute.amazonaws.com')
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
