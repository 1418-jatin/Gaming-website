pipeline {
    agent { label 'linux agent' }          // keep your agent's label

    environment {
        IMAGE_NAME  = "19901418/prt-oct-25th"
        IMAGE_TAG   = "${BUILD_NUMBER}"
        APP_NAME    = "gaming-website"
        K8S_NODE_IP = "172.31.29.248"
        NODE_PORT   = "30080"
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

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'ubuntu-cred', variable: 'KUBECONFIG')]) {
                    sh '''
                      kubectl get nodes
                      sed "s|IMAGE_PLACEHOLDER|$IMAGE_NAME:$IMAGE_TAG|g" k8s/deployment.yaml | kubectl apply -f -
                      kubectl apply -f k8s/service.yaml
                      kubectl rollout status deployment/$APP_NAME --timeout=120s || {
                        echo "Rollout failed - rolling back"
                        kubectl rollout undo deployment/$APP_NAME
                        exit 1
                      }
                      kubectl get pods,svc -l app=$APP_NAME -o wide
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh 'curl -fsS --retry 5 --retry-delay 3 http://$K8S_NODE_IP:$NODE_PORT > /dev/null'
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
