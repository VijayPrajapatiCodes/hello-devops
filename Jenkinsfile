pipeline {

    agent any

    stages {

        // =========================
        // 1. CHECKOUT
        // =========================
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
            }
        }

        // =========================
        // 2. BUILD SPRING BOOT
        // =========================
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

        // =========================
        // 3. DOCKER BUILD
        // =========================
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

        // =========================
        // 4. DOCKER PUSH
        // =========================
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
                        set -e

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

                        echo "=== Docker Logout ==="

                        docker logout
                    '''
                }
            }
        }

        // =========================
        // 5. DEPLOY
        // =========================
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
                        set -e

                        echo "=== Docker Login ==="

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "=== Deploying with Docker Compose ==="

                        cd "$WORKSPACE"

                        echo "=== Current Directory ==="
                        pwd

                        echo "=== Files ==="
                        ls -la

                        echo "=== Docker Compose Config ==="
                        docker compose config

                        echo "=== Pulling Latest Image ==="
                        docker compose pull

                        echo "=== Starting/Recreating Container ==="
                        docker compose up -d --force-recreate

                        echo "=== Deployment Status ==="
                        docker compose ps

                        echo "=== Testing Application ==="
                        curl -f http://localhost:8081/api/hello

                        echo "=== Docker Logout ==="
                        docker logout
                    '''
                }
            }
        }
    }

    // =========================
    // PIPELINE RESULT
    // =========================
    post {

        success {
            echo '🚀 CI/CD Pipeline Successful!'
        }

        failure {
            echo '❌ Pipeline Failed!'
        }
    }
}
