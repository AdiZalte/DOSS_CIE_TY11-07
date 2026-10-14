
pipeline {
    agent any

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                bat 'dir'
            }
        }

        stage('Build') {
            steps {
                bat 'docker build -t devops-cie-app:%BUILD_NUMBER% .'
                bat 'docker tag devops-cie-app:%BUILD_NUMBER% devops-cie-app:latest'
                bat 'docker images | findstr devops-cie-app'
            }
        }

        stage('Test') {
            steps {
                // Remove any container left from a previous run
                bat 'docker rm -f cie-test 2>nul || exit /b 0'

                bat 'docker run -d --name cie-test -p 8082:80 devops-cie-app:%BUILD_NUMBER%'
                bat 'ping -n 11 127.0.0.1 > nul'
                bat 'curl --fail --show-error http://localhost:8082'
            }

            post {
                always {
                    // Clean up even if the test fails
                    bat 'docker rm -f cie-test 2>nul || exit /b 0'
                }
            }
        }

        
        stage('Deploy') {
            steps {
                bat '''
                    set "KUBECONFIG=C:\\ProgramData\\Jenkins\\.kube\\config"

                    echo Checking Kubernetes context...
                    kubectl config current-context
                    if errorlevel 1 exit /b 1

                    echo Checking Kubernetes connection...
                    kubectl get nodes
                    if errorlevel 1 exit /b 1

                    echo Validating Kubernetes manifests...
                   
                    kubectl apply -f deployment.yaml
                    if errorlevel 1 exit /b 1

                    kubectl apply -f service.yaml
                    if errorlevel 1 exit /b 1

                    kubectl rollout status deployment/devops-app --timeout=120s
                    if errorlevel 1 exit /b 1

                    kubectl get pods
                    kubectl get services


                    echo Manifests validated successfully.
                '''
            }
        }

    }

    post {
        success {
            echo 'CI/CD SUCCESS: Git -> Build -> Test -> Deploy done'
        }
        failure {
            echo 'CI/CD FAILED: check Console Output'
        }
        always {
            echo 'Pipeline execution finished.'
        }
    }
}
