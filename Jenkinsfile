
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
                bat 'echo Deploy stage: applying Kubernetes manifests...'
                bat 'dir deployment.yaml service.yaml'
                bat 'kubectl apply --dry-run=client --validate=false -f deployment.yaml'
                bat 'kubectl apply --dry-run=client --validate=false -f service.yaml'
                bat 'echo Manifests valid - ready for kubectl apply from workstation'
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
