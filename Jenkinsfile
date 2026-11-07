pipeline {

    agent any

    environment {
        IMAGE_NAME = "YOUR_DOCKERHUB_USERNAME/gaming-website"
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('DockerHub Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                sh """
                docker push ${IMAGE_NAME}:${IMAGE_TAG}

                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest

                docker push ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy') {

            steps {

                sh """

                docker stop gaming-website || true

                docker rm gaming-website || true

                docker pull ${IMAGE_NAME}:latest

                docker run -d \
                --name gaming-website \
                -p 80:80 \
                --restart unless-stopped \
                ${IMAGE_NAME}:latest

                """
            }

        }

    }

}
