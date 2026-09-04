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
                        set -e

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
                    set -e

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

                        echo "================================="
                        echo "       STARTING DEPLOYMENT"
                        echo "================================="

                        echo "=== Docker Login ==="

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "=== Deployment Directory ==="

                        cd "$WORKSPACE"

                        echo "Current Directory:"
                        pwd

                        echo "=== Checking Compose File ==="

                        test -f docker-compose.yml

                        echo "docker-compose.yml found ✅"

                        echo "=== Docker Compose Config ==="

                        docker compose config

                        echo "=== Pulling Latest Docker Image ==="

                        docker compose pull

                        echo "=== Removing Previous Container ==="

                        docker rm -f hello-devops 2>/dev/null || true

                        echo "Previous container cleanup completed ✅"

                        echo "=== Removing Orphan Containers ==="

                        docker compose down --remove-orphans 2>/dev/null || true

                        echo "=== Starting Latest Application ==="

                        docker compose up -d --force-recreate --remove-orphans

                        echo "=== Deployment Status ==="

                        docker compose ps

                        echo "=== Waiting for Application ==="

                        sleep 10

                        echo "=== Application Health Check ==="

                        curl -f http://localhost:8081/api/hello

                        echo ""
                        echo "================================="
                        echo "     DEPLOYMENT SUCCESSFUL 🚀"
                        echo "================================="

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
            echo 'Application deployed successfully to EC2.'
        }

        failure {
            echo '❌ CI/CD Pipeline Failed!'
            echo 'Check the failed stage in Console Output.'
        }
    }
}
