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
                        set -e

                        echo "================================="
                        echo "        JAVA VERSION"
                        echo "================================="

                        java -version

                        echo "================================="
                        echo "        MAVEN VERSION"
                        echo "================================="

                        ./mvnw -version

                        echo "================================="
                        echo "        MAVEN BUILD"
                        echo "================================="

                        ./mvnw clean package

                        echo "================================="
                        echo "        BUILD SUCCESSFUL"
                        echo "================================="
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {

                sh '''
                    set -e

                    echo "================================="
                    echo "        DOCKER VERSION"
                    echo "================================="

                    docker --version

                    echo "================================="
                    echo "        DOCKER BUILD"
                    echo "================================="

                    docker build -t hello-devops:latest .

                    echo "================================="
                    echo "   DOCKER BUILD SUCCESSFUL"
                    echo "================================="
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
                        set -e

                        echo "================================="
                        echo "        DOCKER LOGIN"
                        echo "================================="

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "================================="
                        echo "        DOCKER TAG"
                        echo "================================="

                        docker tag \
                            hello-devops:latest \
                            "$DOCKER_USERNAME/hello-devops:latest"

                        echo "================================="
                        echo "        DOCKER PUSH"
                        echo "================================="

                        docker push \
                            "$DOCKER_USERNAME/hello-devops:latest"

                        echo "================================="
                        echo "        DOCKER LOGOUT"
                        echo "================================="

                        docker logout

                        echo "================================="
                        echo "     DOCKER PUSH SUCCESSFUL"
                        echo "================================="
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

                        echo "docker-compose.yml found"


                        echo "=== Docker Compose Config ==="

                        docker compose config


                        echo "=== Stopping Existing Compose Application ==="

                        docker compose down \
                            --remove-orphans \
                            2>/dev/null || true


                        echo "=== Removing Existing hello-devops Container ==="

                        docker rm -f hello-devops \
                            2>/dev/null || true

                        echo "hello-devops cleanup completed"


                        echo "=== Checking Docker Containers Using Port 8081 ==="

                        PORT_CONTAINERS=$(docker ps -aq --filter "publish=8081")

                        if [ -n "$PORT_CONTAINERS" ]; then

                            echo "Containers using port 8081:"
                            echo "$PORT_CONTAINERS"

                            echo "=== Removing Containers Using Port 8081 ==="

                            docker rm -f $PORT_CONTAINERS

                            echo "Docker port cleanup completed"

                        else

                            echo "No Docker container is using port 8081"

                        fi


                        echo "=== Checking HOST Port 8081 ==="

                        if sudo -n fuser 8081/tcp >/dev/null 2>&1; then

                            echo "WARNING: Host port 8081 is occupied."

                            echo "Process using port 8081:"

                            sudo -n fuser -v 8081/tcp || true

                            echo "=== Freeing HOST Port 8081 ==="

                            sudo -n fuser -k 8081/tcp

                            sleep 3

                            echo "Host port cleanup completed"

                        else

                            echo "Host port 8081 is already free"

                        fi


                        echo "=== Final Port Verification ==="

                        if sudo -n fuser 8081/tcp >/dev/null 2>&1; then

                            echo "================================="
                            echo " ERROR: PORT 8081 STILL OCCUPIED"
                            echo "================================="

                            sudo -n fuser -v 8081/tcp || true

                            exit 1

                        fi

                        echo "Port 8081 is completely free"


                        echo "=== Pulling Latest Docker Image ==="

                        docker compose pull


                        echo "=== Starting Latest Application ==="

                        docker compose up -d \
                            --force-recreate \
                            --remove-orphans


                        echo "=== Deployment Status ==="

                        docker compose ps


                        echo "=== Waiting For Application ==="

                        sleep 10


                        echo "=== Application Health Check ==="

                        curl -f \
                            http://localhost:8081/api/hello


                        echo ""

                        echo "================================="
                        echo "     DEPLOYMENT SUCCESSFUL"
                        echo "================================="


                        echo "=== Docker Logout ==="

                        docker logout
                    '''
                }
            }
        }
    }

    post {

        success {

            echo '''
=================================
      CI/CD SUCCESSFUL
=================================

GitHub
   ↓
Jenkins
   ↓
Maven Build
   ↓
Docker Build
   ↓
Docker Hub
   ↓
EC2 Deployment
   ↓
Application Running
=================================
'''
        }

        failure {

            echo '''
=================================
       CI/CD FAILED
=================================

Check the failed stage
in Jenkins Console Output.
=================================
'''
        }
    }
}
