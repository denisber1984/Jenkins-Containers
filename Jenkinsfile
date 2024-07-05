pipeline {
    agent {
        docker {
            image 'denisber1984/jenkins-agent:latest'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        SNYK_TOKEN = credentials('snyk-api-token')
        SNYK_ORG_ID = 'cce0d520-322a-4751-a5e3-106da15b2776'
    }
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        def app = docker.build("denisber1984/mypolybot-app:latest", 'polybot')
                        app.push("latest")
                    }
                }
            }
        }
        stage('Snyk Security Scan') {
            steps {
                script {
                    withEnv(["SNYK_TOKEN=${SNYK_TOKEN}", "SNYK_ORG_ID=${SNYK_ORG_ID}"]) {
                        sh 'snyk auth $SNYK_TOKEN --api=https://snyk.io/api'
                        sh 'snyk test --org=$SNYK_ORG_ID --docker denisber1984/mypolybot-app:latest --file=polybot/Dockerfile'
                    }
                }
            }
        }
    }
}
