pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "kurellapoojitha/foodwaste:latest"
        GIT_REPO_URL = "https://github.com/Poojitha3007/agileFat.git"
    }
    
    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: "${env.GIT_REPO_URL}"
            }
        }
        
        stage('Build Docker Image') {
            steps {
                bat "docker build -t %DOCKER_IMAGE% ."
            }
        }
        
        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-poojitha', 
                    usernameVariable: 'USER', 
                    passwordVariable: 'PASS'
                )]) {
                    bat "docker login -u %USER% -p %PASS%"
                    bat "docker push %DOCKER_IMAGE%"
                }
            }
        }
        
        // Requirement 2: Conditional Stage Execution example
        stage('Deploy to Kubernetes') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig', 
                    variable: 'KUBECONFIG'
                )]) {
                    bat '''
                    set KUBECONFIG=%KUBECONFIG%
                    kubectl apply -f deployment.yaml --validate=false
                    '''
                }
            }
        }
    }
}
