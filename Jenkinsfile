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
                        echo "Docker image built: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    }
                }
            }
        }
        
        stage('Verify Credentials Exist') {
            steps {
                script {
                    echo "Attempting to load credentials..."
                    try {
                        withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                            bat '''
                                @echo off
                                echo ================================
                                echo Credential Check:
                                echo Username: %DOCKER_USER%
                                if "%DOCKER_USER%"=="" (
                                    echo ERROR: Username is EMPTY!
                                    exit /b 1
                                )
                                if "%DOCKER_PASS%"=="" (
                                    echo ERROR: Password is EMPTY!
                                    exit /b 1
                                )
                                echo Username is set correctly
                                echo Password is set (length check)
                                echo %DOCKER_PASS%> temp.txt
                                for %%A in (temp.txt) do (
                                    if %%~zA LSS 10 (
                                        echo WARNING: Password seems too short - only %%~zA bytes
                                    ) else (
                                        echo Password length: %%~zA bytes - OK
                                    )
                                )
                                del temp.txt
                                echo ================================
                            '''
                        }
                    } catch (Exception e) {
                        error "Failed to load credentials: ${e.message}"
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
                            echo Logging in to Docker Hub...
                            echo %DOCKER_PASS%| docker login -u %DOCKER_USER% --password-stdin
                            if errorlevel 1 (
                                echo Docker login failed!
                                echo Please check your Docker Hub credentials
                                exit /b 1
                            )
                            echo Docker login successful
                            echo.
                            echo Pushing image with tag %DOCKER_TAG%...
                            docker push %DOCKER_IMAGE%:%DOCKER_TAG%
                            if errorlevel 1 (
                                echo Failed to push image with tag %DOCKER_TAG%
                                exit /b 1
                            )
                            echo.
                            echo Pushing image with latest tag...
                            docker push %DOCKER_IMAGE%:latest
                            if errorlevel 1 (
                                echo Failed to push image with latest tag
                                exit /b 1
                            )
                            echo.
                            echo Docker images pushed successfully
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
                        echo Applying Kubernetes configurations...
                        kubectl apply -f k8s/configmap.yaml
                        if errorlevel 1 (
                            echo Failed to apply configmap
                            exit /b 1
                        )
                        
                        kubectl apply -f k8s/deployment.yaml
                        if errorlevel 1 (
                            echo Failed to apply deployment
                            exit /b 1
                        )
                        
                        kubectl apply -f k8s/service.yaml
                        if errorlevel 1 (
                            echo Failed to apply service
                            exit /b 1
                        )
                        
                        echo.
                        echo Updating deployment image...
                        kubectl set image deployment/flask-app flask-app=%DOCKER_IMAGE%:%DOCKER_TAG% --record
                        if errorlevel 1 (
                            echo Failed to set image
                            exit /b 1
                        )
                        
                        echo.
                        echo Waiting for rollout to complete...
                        kubectl rollout status deployment/flask-app
                        if errorlevel 1 (
                            echo Rollout failed
                            exit /b 1
                        )
                        
                        echo.
                        echo Deployment completed successfully
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                bat '''
                    @echo off
                    echo Logging out from Docker Hub...
                    docker logout
                '''
            }
        }
        success {
            echo '✓ Pipeline executed successfully!'
            echo "Image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
        }
        failure {
            echo '✗ Pipeline failed - check logs above'
        }
    }
}