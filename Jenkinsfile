
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
                bat '''
                    docker rm -f cie-test 2>nul || exit /b 0
                    docker run -d --name cie-test -p 8082:80 devops-cie-app:%BUILD_NUMBER%

                    ping -n 11 127.0.0.1 >nul

                    docker ps -a --filter "name=cie-test"
                    docker logs cie-test

                    curl.exe --retry 5 --retry-connrefused --retry-delay 2 --connect-timeout 5 --fail --show-error http://127.0.0.1:8082
                '''
            }
            post {
                always {
                    bat 'docker rm -f cie-test 2>nul || exit /b 0'
                }
            }
        }


        stage('Transfer Image') {
            steps {
                bat '''
                    echo Exporting Docker image...
                    docker save devops-cie-app:%BUILD_NUMBER% -o devops-cie-app.tar
                    if errorlevel 1 exit /b 1

                    echo Importing image into Kubernetes containerd...
                    docker exec -i desktop-control-plane ctr -n k8s.io images import - < devops-cie-app.tar
                    if errorlevel 1 exit /b 1

                    echo Image transfer completed.
                '''
            }
        }

        stage('Deploy') {
            steps {
                bat '''
                    set "KUBECONFIG=C:\\ProgramData\\Jenkins\\.kube\\config"

                    echo Checking Kubernetes connection...
                    kubectl get nodes
                    if errorlevel 1 exit /b 1

                    echo Deploying Kubernetes manifests...
                    kubectl apply -f deployment.yaml
                    if errorlevel 1 exit /b 1

                    kubectl apply -f service.yaml
                    if errorlevel 1 exit /b 1

                    echo Updating deployment to the newly built image...
                    kubectl set image deployment/devops-app web=devops-cie-app:%BUILD_NUMBER%
                    if errorlevel 1 exit /b 1

                    kubectl rollout status deployment/devops-app --timeout=120s
                    if errorlevel 1 exit /b 1

                    kubectl get pods
                    kubectl get services
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD SUCCESS: Git -> Build -> Test -> Transfer -> Deploy'
        }
        failure {
            echo 'CI/CD FAILED: check Console Output'
        }
        always {
            echo 'Pipeline execution finished.'
        }
    }
}
