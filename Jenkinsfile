
pipeline {
    agent any

    parameters {
        string(
            name: 'VERSION',
            defaultValue: 'v1.0',
            description: 'Docker image release version'
        )
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: 'Demonstrate automatic rollback'
        )
    }

    environment {
        IMAGE = 'arshiaanjumd/university-lms'
        DEPLOYMENT = 'university-lms'
        CONTAINER_NAME = 'university-lms'
        DOCKER_EXE = 'C:\\Users\\arshi\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    "%DOCKER_EXE%" build -t %IMAGE%:%VERSION% .
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                bat '''
                    "%DOCKER_EXE%" run --rm %IMAGE%:%VERSION% nginx -t
                '''
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
                        echo %DH_TOKEN%| "%DOCKER_EXE%" login -u %DH_USER% --password-stdin
                        if errorlevel 1 exit /b 1

                        "%DOCKER_EXE%" push %IMAGE%:%VERSION%
                        set PUSH_RESULT=%ERRORLEVEL%

                        "%DOCKER_EXE%" logout
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
                        def previousImage = bat(
                            script: '@echo off & kubectl get deployment university-lms -o=jsonpath="{.spec.template.spec.containers[0].image}"',
                            returnStdout: true
                        ).trim()

                        if (!previousImage) {
                            error('Unable to identify previous deployment image')
                        }

                        try {
                            if (params.SIMULATE_FAILURE) {
                                bat '''
                                    kubectl set image deployment/university-lms university-lms=%IMAGE%:%VERSION%-broken
                                '''
                            } else {
                                bat '''
                                    kubectl set image deployment/university-lms university-lms=%IMAGE%:%VERSION%
                                '''
                            }

                            bat '''
                                kubectl rollout status deployment/university-lms --timeout=120s
                                kubectl get pods
                                kubectl get service university-lms-service
                            '''

                            echo 'Deployment and health verification succeeded.'

                        } catch (err) {
                            echo "Release failed. Rolling back from ${previousImage}"

                            bat '''
                                kubectl rollout undo deployment/university-lms
                                kubectl rollout status deployment/university-lms --timeout=120s
                                kubectl get pods
                            '''

                            error('Release failed. Automatic rollback was attempted.')
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'University LMS CI/CD pipeline finished.'
        }
    }
}
