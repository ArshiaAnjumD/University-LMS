
pipeline {
    agent any

    parameters {
        string(name: 'VERSION', defaultValue: 'v1.0',
               description: 'Release version')
        booleanParam(name: 'SIMULATE_FAILURE', defaultValue: false,
                     description: 'Demonstrate automatic rollback')
    }

    environment {
        IMAGE = 'arshiaanjumd/university-lms'
        APP = 'university-lms'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t arshiaanjumd/university-lms:%VERSION% .'
            }
        }

        stage('Test Docker Image') {
            steps {
                bat 'docker run --rm arshiaanjumd/university-lms:%VERSION% nginx -t'
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
                        docker push arshiaanjumd/university-lms:%VERSION%
                        set PUSH_RESULT=%ERRORLEVEL%
                        docker logout
                        exit /b %PUSH_RESULT%
                    '''
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                script {
                    def oldImage = bat(
                        script: '@echo off & kubectl get deployment university-lms -o=jsonpath="{.spec.template.spec.containers[0].image}"',
                        returnStdout: true
                    ).trim()

                    if (!oldImage) {
                        error('Could not identify the previous stable image')
                    }

                    try {
                        if (params.SIMULATE_FAILURE) {
                            bat 'kubectl set image deployment/university-lms university-lms=arshiaanjumd/university-lms:%VERSION%-broken'
                        } else {
                            bat 'kubectl set image deployment/university-lms university-lms=arshiaanjumd/university-lms:%VERSION%'
                        }

                        bat 'kubectl rollout status deployment/university-lms --timeout=120s'
                        bat 'kubectl get pods'
                        bat 'kubectl get service university-lms-service'
                    } catch (err) {
                        echo 'Release failed. Restoring the previous stable revision.'

                        bat 'kubectl rollout undo deployment/university-lms'
                        bat 'kubectl rollout status deployment/university-lms --timeout=120s'
                        bat 'kubectl get pods'

                        error('Release failed; rollback executed. Check rollout output.')
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'University LMS pipeline completed.'
        }
    }
}
