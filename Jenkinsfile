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
                    powershell """
                        \$env:KUBECONFIG='${env.KUBECONFIG_PATH}'
                        Write-Host 'Applying Kubernetes manifests...'
                        & '${env.KUBECTL_BIN}' apply -f k8s/deployment.yaml
                        & '${env.KUBECTL_BIN}' apply -f k8s/service.yaml
                        Write-Host 'Updating deployment image...'
                        & '${env.KUBECTL_BIN}' set image deployment/event-portal-deployment event-portal=${deployImage}
                        
                        try {
                            & '${env.KUBECTL_BIN}' rollout status deployment/event-portal-deployment --timeout=90s
                            Write-Host 'EVENT PORTAL DEPLOYMENT SUCCESSFUL'
                        } catch {
                            Write-Host 'FAILED ROLLOUT DETECTED. Rolling back...'
                            & '${env.KUBECTL_BIN}' rollout undo deployment/event-portal-deployment
                            & '${env.KUBECTL_BIN}' rollout status deployment/event-portal-deployment --timeout=120s
                            throw 'Rollback completed successfully after failed deployment.'
                        }
                    """
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
