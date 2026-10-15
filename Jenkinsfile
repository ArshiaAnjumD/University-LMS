
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t arshiaanjumd/university-lms:v1.0 .'
            }
        }

        stage('Test Docker Image') {
            steps {
                bat 'docker run --rm arshiaanjumd/university-lms:v1.0 nginx -t'
            }
        }

        stage('Publish to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    bat '''
                        docker login -u "%DOCKER_USER%" -p "%DOCKER_TOKEN%"
                        docker push arshiaanjumd/university-lms:v1.0
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat '''
                        kubectl apply -f deployment.yaml
                        kubectl rollout status deployment/university-lms --timeout=120s
                        kubectl get pods
                        kubectl get service university-lms-service
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution finished.'
        }
    }
}
