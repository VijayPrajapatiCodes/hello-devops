pipeline {

    agent any

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
            }
        }

        stage('Build') {
            steps {
                sh '''
                    echo "=== Java Version ==="
                    java -version

                    echo "=== Maven Version ==="
                    ./mvnw -version

                    echo "=== Maven Build ==="
                    ./mvnw clean package
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "=== Docker Version ==="
                    docker --version

                    echo "=== Building Docker Image ==="
                    docker build -t hello-devops:latest .
                '''
            }
        }
    }

    post {
        success {
            echo 'CI + Docker Image Build Successful! 🚀'
        }

        failure {
            echo 'Pipeline Failed! ❌'
        }
    }
}
