pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USERNAME = 'wolfie8935'
        DOCKER_IMAGE_NAME = 'flask-app'
        DOCKER_CREDENTIALS = credentials('docker-credentials')
        KUBECONFIG = credentials('kubeconfig-credentials')
        NAMESPACE = 'default'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out repository..."
                    checkout scm
                    echo "Repository checked out successfully"
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    dir('app') {
                        echo "Building Docker image..."
                        bat '''
                            docker build -t ${DOCKER_USERNAME}/${DOCKER_IMAGE_NAME}:${BUILD_NUMBER} .
                            docker tag ${DOCKER_USERNAME}/${DOCKER_IMAGE_NAME}:${BUILD_NUMBER} ${DOCKER_USERNAME}/${DOCKER_IMAGE_NAME}:latest
                        '''
                        echo "Docker image built: ${DOCKER_USERNAME}/${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                    }
                }
            }
        }
        
        stage('Verify Credentials Exist') {
            steps {
                script {
                    echo "Attempting to load credentials..."
                    withCredentials([usernamePassword(credentialsId: 'docker-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        bat '''
                            @echo off
                            echo ================================
                            echo Credential Check:
                            echo Username: %DOCKER_USER%
                            if "%DOCKER_USER%"=="" (
                                echo Username is NOT set
                                exit /b 1
                            ) else (
                                echo Username is set correctly
                            )
                            
                            if "%DOCKER_PASS%"=="" (
                                echo Password is NOT set
                                exit /b 1
                            ) else (
                                echo Password is set (length check)
                                for /F %%A in ('powershell -Command "$pass='%DOCKER_PASS%'; $pass.Length"') do set PASSLEN=%%A
                                echo Password length: %PASSLEN% bytes - OK
                            )
                            echo ================================
                        '''
                    }
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        bat '''
                            @echo off
                            echo Logging in to Docker Hub...
                            echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
                            if errorlevel 1 (
                                echo Docker login failed
                                exit /b 1
                            )
                            echo Docker login successful
                            
                            echo.
                            echo Pushing image with tag %BUILD_NUMBER%...
                            docker push %DOCKER_USER%/%DOCKER_IMAGE_NAME%:%BUILD_NUMBER%
                            if errorlevel 1 (
                                echo Docker push failed
                                exit /b 1
                            )
                            
                            echo.
                            echo Pushing image with latest tag...
                            docker push %DOCKER_USER%/%DOCKER_IMAGE_NAME%:latest
                            if errorlevel 1 (
                                echo Docker push latest failed
                                exit /b 1
                            )
                            
                            echo Docker images pushed successfully
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG')]) {
                        bat '''
                            @echo off
                            setlocal enabledelayedexpansion
                            
                            echo Checking Kubernetes connection...
                            kubectl cluster-info
                            if errorlevel 1 (
                                echo ERROR: Cannot connect to Kubernetes cluster
                                echo Please configure kubectl on Jenkins server
                                exit /b 1
                            )
                            
                            echo.
                            echo Deploying to Kubernetes cluster...
                            
                            echo Applying ConfigMap...
                            kubectl apply -f k8s/configmap.yaml -n %NAMESPACE%
                            if errorlevel 1 (
                                echo ERROR: Failed to apply ConfigMap
                                exit /b 1
                            )
                            
                            echo Applying Deployment...
                            kubectl set image deployment/flask-app flask-app=%DOCKER_USER%/%DOCKER_IMAGE_NAME%:%BUILD_NUMBER% -n %NAMESPACE% --record 2>nul
                            if errorlevel 1 (
                                echo Deployment does not exist, creating new deployment...
                                kubectl apply -f k8s/deployment.yaml -n %NAMESPACE%
                            ) else (
                                echo Deployment updated with new image
                            )
                            
                            echo Applying Service...
                            kubectl apply -f k8s/service.yaml -n %NAMESPACE%
                            if errorlevel 1 (
                                echo ERROR: Failed to apply Service
                                exit /b 1
                            )
                            
                            echo.
                            echo Checking deployment status...
                            kubectl rollout status deployment/flask-app -n %NAMESPACE% --timeout=5m
                            
                            echo.
                            echo Getting service information...
                            kubectl get svc flask-app -n %NAMESPACE%
                            
                            echo Kubernetes deployment completed successfully!
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Pipeline execution finished"
                withCredentials([usernamePassword(credentialsId: 'docker-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    bat '''
                        @echo off
                        echo Logging out from Docker Hub...
                        docker logout
                    '''
                }
            }
        }
        success {
            echo "✓ Pipeline succeeded - Deployment successful!"
        }
        failure {
            echo "✗ Pipeline failed - check logs above"
        }
    }
}