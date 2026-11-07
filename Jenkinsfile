pipeline {
    agent { label 'linux agent' }

    environment {
        IMAGE_NAME = "19901418/prt-oct-25th"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        CONTAINER  = "gaming-website"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker build -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push $IMAGE_NAME:$IMAGE_TAG
                      docker push $IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                  docker stop $CONTAINER || true
                  docker rm $CONTAINER || true
                  docker run -d --name $CONTAINER -p 80:80 \
                    --restart unless-stopped $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh 'sleep 5 && curl -fsS http://localhost:80 > /dev/null'
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            sh 'docker image prune -f || true'
        }
    }
}
