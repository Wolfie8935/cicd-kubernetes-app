pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE = 'wolfie8935/flask-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
        KUBECONFIG = '/home/jenkins/.kube/config'
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
                    dir('app') {
                        sh 'docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .'
                        sh 'docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest'
                    }
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh 'echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin'
                        sh 'docker push ${DOCKER_IMAGE}:${DOCKER_TAG}'
                        sh 'docker push ${DOCKER_IMAGE}:latest'
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh '''
                    kubectl apply -f k8s/configmap.yaml
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                    kubectl set image deployment/flask-app flask-app=${DOCKER_IMAGE}:${DOCKER_TAG} --record
                    kubectl rollout status deployment/flask-app
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh 'docker logout'
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
