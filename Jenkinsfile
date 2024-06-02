pipeline {
    agent any

    environment {
        DOCKERHUB_REPOSITORY = 'denisber1984/polybot_app'
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
                            docker build -t polybot-app -f polybot/Dockerfile polybot/

                            # Tag the Docker image
                            docker tag polybot-app:latest $DOCKERHUB_REPOSITORY:latest

                            # Push the Docker image to DockerHub
                            docker push $DOCKERHUB_REPOSITORY:latest
                        '''
                    }
                }
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
