pipeline {
    agent { label 'slave-agent1' }

    environment {
        IMAGE_NAME  = "19901418/prt-oct-25th"
        IMAGE_TAG   = "${BUILD_NUMBER}"
        APP_NAME    = "gaming-website"
        K8S_NODE_IP = "172.31.29.248"
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
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .'
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

        stage('Deploy to Minikube') {
            steps {
                sh '''
                  SSH="ssh -o StrictHostKeyChecking=accept-new -o BatchMode=yes ubuntu@$K8S_NODE_IP"

                  $SSH "kubectl get nodes" < /dev/null

                  sed "s|IMAGE_PLACEHOLDER|$IMAGE_NAME:$IMAGE_TAG|g" k8s/deployment.yaml | $SSH "kubectl apply -f -"
                  $SSH "kubectl apply -f -" < k8s/service.yaml

                  $SSH "kubectl rollout status deployment/$APP_NAME --timeout=120s" < /dev/null || {
                    echo "Rollout failed - rolling back"
                    $SSH "kubectl rollout undo deployment/$APP_NAME" < /dev/null
                    exit 1
                  }

                  $SSH "kubectl get pods,svc -l app=$APP_NAME -o wide" < /dev/null
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                  ssh -o BatchMode=yes ubuntu@$K8S_NODE_IP \
                    'curl -fsS --retry 5 --retry-delay 3 http://$(minikube ip):30080 > /dev/null' < /dev/null
                '''
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
