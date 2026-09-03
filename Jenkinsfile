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
                withEnv([
                    'JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64',
                    'PATH+JAVA=/usr/lib/jvm/java-17-openjdk-amd64/bin'
                ]) {
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
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "=== Docker Version ==="
                    docker --version

                    echo "=== Docker Build ==="
                    docker build -t hello-devops:latest .
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "=== Docker Login ==="

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "=== Docker Tag ==="

                        docker tag hello-devops:latest \
                            $DOCKER_USERNAME/hello-devops:latest

                        echo "=== Docker Push ==="

                        docker push \
                            $DOCKER_USERNAME/hello-devops:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "=== Deploying ==="

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        docker stop hello-devops || true
                        docker rm hello-devops || true

                        docker pull \
                            $DOCKER_USERNAME/hello-devops:latest

                        docker run -d \
                            --name hello-devops \
                            -p 8081:8080 \
                            $DOCKER_USERNAME/hello-devops:latest

                        docker logout

                        echo "=== Deployment Successful ==="
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '🚀 CI/CD Pipeline Successful!'
        }

        failure {
            echo '❌ Pipeline Failed!'
        }
    }
}
