
pipeline {
    agent any

    parameters {
        string(name: 'VERSION', defaultValue: 'v1.0',
               description: 'Version tag for this release')
        booleanParam(name: 'SIMULATE_FAILURE', defaultValue: false,
                     description: 'Test automatic rollback')
    }

    environment {
        IMAGE_NAME = 'arshiaanjumd/university-lms'
        DEPLOYMENT = 'university-lms'
        CONTAINER_NAME = 'university-lms'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE_NAME%:%VERSION% .'
            }
        }

        stage('Test Docker Image') {
            steps {
                bat 'docker run --rm %IMAGE_NAME%:%VERSION% nginx -t'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    bat '''
                        @echo off
                        echo %DH_TOKEN%| docker login -u %DH_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push %IMAGE_NAME%:%VERSION%
                        set PUSH_RESULT=%ERRORLEVEL%
                        docker logout
                        exit /b %PUSH_RESULT%
                    '''
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    script {
                        try {
                            if (params.SIMULATE_FAILURE) {
                                bat '''
                                    kubectl set image deployment/%DEPLOYMENT% %CONTAINER_NAME%=%IMAGE_NAME%:%VERSION%-broken
                                    kubectl rollout status deployment/%DEPLOYMENT% --timeout=90s
                                '''
                            } else {
                                bat '''
                                    kubectl set image deployment/%DEPLOYMENT% %CONTAINER_NAME%=%IMAGE_NAME%:%VERSION%
                                    kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                                    kubectl get pods
                                    kubectl get service university-lms-service
                                '''
                            }
                        } catch (err) {
                            echo 'Deployment failed. Rolling back to the previous revision.'
                            bat '''
                                kubectl rollout undo deployment/%DEPLOYMENT%
                                kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                                kubectl get pods
                            '''
                            error('Release failed. Rollback attempted; check the rollout output.')
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'University LMS pipeline finished.'
        }
    }
}
