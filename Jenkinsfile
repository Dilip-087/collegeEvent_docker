pipeline {
    agent any
    parameters {
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: 'Deploy a faulty image to test automatic rollback'
        )
    }
    environment {
        IMAGE = "dilip087/college-event:${env.BUILD_NUMBER}"
        DOCKER_BIN = 'C:\\Users\\itzme\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_BIN = 'C:\\Users\\itzme\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
        KUBECONFIG_PATH = 'C:\\Users\\itzme\\.kube\\config'
    }
    stages {
        stage('Clone Code') {
            steps { 
                checkout scm 
            }
        }
        stage('Build Docker Image') {
            steps { 
                bat "\"${env.DOCKER_BIN}\" build -t ${env.IMAGE} -t dilip087/college-event:latest ." 
            }
        }
        stage('Test Application') {
            steps {
                bat """
                    "${env.DOCKER_BIN}" rm -f app-test >nul 2>&1 || true
                    "${env.DOCKER_BIN}" run -d --rm --name app-test -p 18080:80 ${env.IMAGE}
                """
            }
        }
        stage('Push Image') {
            steps {
                script {
                    bat "\"${env.DOCKER_BIN}\" push ${env.IMAGE}"
                    bat "\"${env.DOCKER_BIN}\" push dilip087/college-event:latest"
                }
            }
        }
        stage('Deploy and Auto Rollback') {
            steps {
                script {
                    def deployImage = params.SIMULATE_FAILURE ? 'dilip087/college-event:broken-demo' : env.IMAGE
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        def result = bat(
                            returnStatus: true,
                            script: """
                                @echo off
                                set KUBECONFIG=%KUBECONFIG_FILE%
                                echo Deploying Event Portal image...
                                "${env.KUBECTL_BIN}" set image deployment/event-portal-deployment event-portal=${deployImage}
                                if errorlevel 1 exit /b 1
                                
                                echo Checking rollout status...
                                "${env.KUBECTL_BIN}" rollout status deployment/event-portal-deployment --timeout=120s
                                if errorlevel 1 (
                                    echo FAILED ROLLOUT DETECTED! Rolling back to previous stable version...
                                    "${env.KUBECTL_BIN}" rollout undo deployment/event-portal-deployment
                                    if errorlevel 1 exit /b 1
                                    
                                    "${env.KUBECTL_BIN}" rollout status deployment/event-portal-deployment --timeout=120s
                                    if errorlevel 1 exit /b 1
                                    
                                    echo ROLLBACK SUCCESSFUL
                                    exit /b 1
                                )
                                echo EVENT PORTAL DEPLOYMENT SUCCESSFUL
                            """
                        )
                        if (result != 0 && !params.SIMULATE_FAILURE) {
                            error('Event Portal deployment failed.')
                        }
                        if (result != 0) {
                            echo 'Simulated failure detected and successfully rolled back.'
                        }
                    }
                }
            }
        }
    }
    post {
        always { 
            echo 'Pipeline finished. Check stage logs for the result.' 
        }
    }
}
