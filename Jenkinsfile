
pipeline {
    agent any

    parameters {
        string(
            name: 'VERSION',
            defaultValue: 'v1.0',
            description: 'Version tag for the release'
        )
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: 'Test automatic rollback'
        )
    }

    environment {
        IMAGE = 'arshiaanjumd/university-lms'
        DEPLOYMENT = 'university-lms'
        CONTAINER_NAME = 'university-lms'

        DOCKER_EXE = 'C:\\Users\\arshi\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_EXE = 'C:\\Users\\arshi\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
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
                            script: '@echo off & "%KUBECTL_EXE%" get deployment university-lms -o=jsonpath="{.spec.template.spec.containers[0].image}"',
                            returnStdout: true
                        ).trim()

                        if (!previousImage) {
                            error('Could not identify the previous stable image')
                        }

                        try {
                            if (params.SIMULATE_FAILURE) {
                                bat '''
                                    "%KUBECTL_EXE%" set image deployment/university-lms university-lms=%IMAGE%:%VERSION%-broken
                                '''
                            } else {
                                bat '''
                                    "%KUBECTL_EXE%" set image deployment/university-lms university-lms=%IMAGE%:%VERSION%
                                '''
                            }

                            bat '''
                                "%KUBECTL_EXE%" rollout status deployment/university-lms --timeout=120s
                                if errorlevel 1 exit /b 1

                                "%KUBECTL_EXE%" get pods
                                "%KUBECTL_EXE%" get service university-lms-service
                            '''

                            echo 'Deployment and verification successful.'

                        } catch (err) {
                            echo "Deployment failed. Rolling back to previous stable revision: ${previousImage}"

                            bat '''
                                "%KUBECTL_EXE%" rollout undo deployment/university-lms
                                if errorlevel 1 exit /b 1

                                "%KUBECTL_EXE%" rollout status deployment/university-lms --timeout=120s
                                if errorlevel 1 exit /b 1

                                "%KUBECTL_EXE%" get pods
                            '''

                            error('Release failed. Rollback was attempted.')
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
