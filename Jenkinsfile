pipeline {
    agent any

    environment {
        DOCKERHUB_REPOSITORY = 'denisber1984/polybot_app'
        IMAGE_NAME = 'denisber1984/polybot_app'
        IMAGE_TAG = 'latest'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/denisber1984/Jenkins-Containers.git'
            }
        }

        stage('Build Bot app') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD')]) {
                        sh '''
                            # Log in to DockerHub
                            echo $DOCKERHUB_PASSWORD | docker login --username $DOCKERHUB_USERNAME --password-stdin

                            # Build the Docker image
                            docker build -t ${IMAGE_NAME} -f polybot/Dockerfile polybot/

                            # Tag the Docker image
                            docker tag ${IMAGE_NAME}:latest ${DOCKERHUB_REPOSITORY}:${IMAGE_TAG}

                            # Push the Docker image to DockerHub
                            docker push ${DOCKERHUB_REPOSITORY}:${IMAGE_TAG}
                        '''
                    }
                }
            }
        }

        stage('Trigger Deploy') {
            steps {
                build job: 'PolyBot_Deploy', wait: false, parameters: [
                    string(name: 'polybot_URL', value: "${DOCKERHUB_REPOSITORY}:${IMAGE_TAG}")
                ]
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh '''
                    echo "Deploying to production..."
                    # Add your deployment steps here
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Build succeeded!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
