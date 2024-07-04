pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        GITHUB_CREDENTIALS = credentials('github-credentials')
        SNYK_TOKEN = credentials('snyk-api-token')
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout([$class: 'GitSCM', branches: [[name: '*/main']],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [[$class: 'CleanCheckout']],
                        submoduleCfg: [],
                        userRemoteConfigs: [[credentialsId: 'github-credentials', url: 'https://github.com/denisber1984/Jenkins-Containers.git']]])
                }
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        sh 'docker build -t denisber1984/mypolybot-app:latest polybot'
                        sh 'docker push denisber1984/mypolybot-app:latest'
                    }
                }
            }
        }
        stage('Verify Snyk Installation') {
            steps {
                sh 'snyk --version'
            }
        }
        stage('Snyk Security Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                    script {
                        def snykEndpoint = 'https://snyk.io/api'
                        sh 'snyk auth ${SNYK_TOKEN} --api=' + snykEndpoint + ' -d'
                        sh 'snyk test'
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying...'
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
