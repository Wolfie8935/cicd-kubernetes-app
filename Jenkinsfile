pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE = 'wolfie8935/flask-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Repository checked out successfully"
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    dir('app') {
                        bat 'docker build -t %DOCKER_IMAGE%:%DOCKER_TAG% .'
                        bat 'docker tag %DOCKER_IMAGE%:%DOCKER_TAG% %DOCKER_IMAGE%:latest'
                        echo "Docker image built: %DOCKER_IMAGE%:%DOCKER_TAG%"
                    }
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        bat '''
                            @echo off
                            echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
                            docker push %DOCKER_IMAGE%:%DOCKER_TAG%
                            docker push %DOCKER_IMAGE%:latest
                            echo Docker image pushed successfully
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    bat '''
                        @echo off
                        kubectl apply -f k8s/configmap.yaml
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        kubectl set image deployment/flask-app flask-app=%DOCKER_IMAGE%:%DOCKER_TAG% --record
                        kubectl rollout status deployment/flask-app
                        echo Deployment completed successfully
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                bat 'docker logout'
            }
        }
        success {
            echo '✓ Pipeline executed successfully!'
        }
        failure {
            echo '✗ Pipeline failed - check logs above'
        }
    }
}