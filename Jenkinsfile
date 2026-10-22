
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'DEPLOY_TO_K8S',
            defaultValue: true,
            description: 'Deploy the application to Kubernetes'
        )
    }

    environment {
        GIT_REPO_URL = 'https://github.com/Poojitha3007/agileFat.git'
        DOCKER_IMAGE = 'kurellapoojitha/foodwaste'
        DOCKER_CREDENTIALS = 'dockerhub-poojitha'
        KUBE_CREDENTIALS = 'kuberconfig'
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: "${GIT_REPO_URL}"

                bat 'git log -1 --oneline'
            }
        }

        stage('Parallel Validation and Testing') {
            parallel {
                stage('Validate HTML') {
                    steps {
                        bat '''
                            if not exist index.html exit /b 1
                            findstr /I /C:"Food Waste Tracking" index.html >nul
                            if errorlevel 1 exit /b 1
                            findstr /I /C:"Redistribution Status Dashboard" index.html >nul
                            if errorlevel 1 exit /b 1
                            findstr /I /C:"Vegetable Fried Rice" index.html >nul
                            if errorlevel 1 exit /b 1
                            echo HTML validation passed.
                        '''
                    }
                }

                stage('Validate Deployment Files') {
                    steps {
                        bat '''
                            if not exist Dockerfile exit /b 1
                            if not exist deployment.yaml exit /b 1
                            findstr /I /C:"foodwaste-portal" deployment.yaml >nul
                            if errorlevel 1 exit /b 1
                            findstr /I /C:"nodePort: 30015" deployment.yaml >nul
                            if errorlevel 1 exit /b 1
                            echo Deployment file validation passed.
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    docker build -t %DOCKER_IMAGE%:latest .
                    if errorlevel 1 exit /b 1

                    docker build -t %DOCKER_IMAGE%:%BUILD_NUMBER% .
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                bat '''
                    docker image inspect %DOCKER_IMAGE%:latest
                    if errorlevel 1 exit /b 1
                    echo Docker image verification passed.
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DOCKER_PASSWORD%| docker login -u "%DOCKER_USERNAME%" --password-stdin
                        if errorlevel 1 exit /b 1

                        docker push %DOCKER_IMAGE%:latest
                        if errorlevel 1 exit /b 1

                        docker push %DOCKER_IMAGE%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        docker logout
                    '''
                }
            }
        }

        stage('Archive Build Artifacts') {
            steps {
                archiveArtifacts(
                    artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                    fingerprint: true
                )
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression {
                    return params.DEPLOY_TO_K8S
                }
            }

            steps {
                withCredentials([
                    file(
                        credentialsId: "${KUBE_CREDENTIALS}",
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    script {
                        def deployStatus = bat(
                            script: '''
                                @echo off
                                set "KUBECONFIG=%KUBECONFIG%"

                                kubectl apply -f deployment.yaml
                                if errorlevel 1 exit /b 1

                                kubectl set image deployment/foodwaste-portal foodwaste-portal=%DOCKER_IMAGE%:%BUILD_NUMBER%
                                if errorlevel 1 exit /b 1

                                kubectl rollout status deployment/foodwaste-portal --timeout=120s
                            ''',
                            returnStatus: true
                        )

                        if (deployStatus != 0) {
                            echo 'Deployment failed. Attempting to roll back.'

                            def rollbackStatus = bat(
                                script: '''
                                    @echo off
                                    set "KUBECONFIG=%KUBECONFIG%"
                                    kubectl rollout undo deployment/foodwaste-portal
                                    kubectl rollout status deployment/foodwaste-portal --timeout=120s
                                ''',
                                returnStatus: true
                            )

                            if (rollbackStatus != 0) {
                                error('Deployment failed and automatic rollback was unsuccessful. Check Kubernetes status.')
                            }

                            error('Deployment failed. Previous deployment restored.')
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Food Waste Portal pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Review the Jenkins Console Output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
