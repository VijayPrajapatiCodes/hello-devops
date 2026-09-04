pipeline {

    agent any

    stages {

        // =========================================
        // 1. CHECKOUT
        // =========================================
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
            }
        }


        // =========================================
        // 2. BUILD SPRING BOOT APPLICATION
        // =========================================
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


        // =========================================
        // 3. BUILD DOCKER IMAGE
        // =========================================
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

                    docker build \
                        -t hello-devops:latest \
                        .


                    echo "================================="
                    echo "   DOCKER BUILD SUCCESSFUL"
                    echo "================================="
                '''
            }
        }


        // =========================================
        // 4. PUSH IMAGE TO DOCKER HUB
        // =========================================
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
                            $DOCKER_USERNAME/hello-devops:latest


                        echo "================================="
                        echo "        DOCKER PUSH"
                        echo "================================="

                        docker push \
                            $DOCKER_USERNAME/hello-devops:latest


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


        // =========================================
        // 5. DEPLOY TO EC2
        // =========================================
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


                        // ---------------------------------
                        // Docker Login
                        // ---------------------------------

                        echo "=== Docker Login ==="

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin


                        // ---------------------------------
                        // Go To Jenkins Workspace
                        // ---------------------------------

                        echo "=== Deployment Directory ==="

                        cd "$WORKSPACE"

                        echo "Current Directory:"

                        pwd


                        // ---------------------------------
                        // Check Compose File
                        // ---------------------------------

                        echo "=== Checking Compose File ==="

                        test -f docker-compose.yml

                        echo "docker-compose.yml found ✅"


                        // ---------------------------------
                        // Validate Compose
                        // ---------------------------------

                        echo "=== Docker Compose Config ==="

                        docker compose config


                        // ---------------------------------
                        // Stop Existing Compose Application
                        // ---------------------------------

                        echo "=== Stopping Existing Compose Application ==="

                        docker compose down \
                            --remove-orphans \
                            2>/dev/null || true


                        // ---------------------------------
                        // Remove OLD hello-devops Container
                        // ---------------------------------

                        echo "=== Removing Existing hello-devops Container ==="

                        docker rm -f hello-devops \
                            2>/dev/null || true

                        echo "hello-devops cleanup completed ✅"


                        // ---------------------------------
                        // Find ANY Container Using Port 8081
                        // ---------------------------------

                        echo "=== Checking Port 8081 ==="

                        PORT_CONTAINERS=$(docker ps -aq --filter "publish=8081")

                        if [ -n "$PORT_CONTAINERS" ]; then

                            echo "⚠️ Containers found using port 8081:"
                            echo "$PORT_CONTAINERS"

                            echo "=== Removing Containers Using Port 8081 ==="

                            docker rm -f $PORT_CONTAINERS

                            echo "Port 8081 Docker containers removed ✅"

                        else

                            echo "Port 8081 is free from Docker containers ✅"

                        fi


                        // ---------------------------------
                        // Verify Port 8081
                        // ---------------------------------

                        echo "=== Verifying Port 8081 ==="

                        if docker ps --format '{{.Ports}}' | grep -q '0.0.0.0:8081->'; then

                            echo "❌ ERROR: Port 8081 is still occupied by Docker."

                            docker ps --format 'table {{.ID}}\\t{{.Names}}\\t{{.Ports}}'

                            exit 1

                        fi

                        echo "Port 8081 is free ✅"


                        // ---------------------------------
                        // Pull Latest Image
                        // ---------------------------------

                        echo "=== Pulling Latest Docker Image ==="

                        docker compose pull


                        // ---------------------------------
                        // Start Application
                        // ---------------------------------

                        echo "=== Starting Latest Application ==="

                        docker compose up -d \
                            --force-recreate \
                            --remove-orphans


                        // ---------------------------------
                        // Deployment Status
                        // ---------------------------------

                        echo "=== Deployment Status ==="

                        docker compose ps


                        // ---------------------------------
                        // Wait For Application
                        // ---------------------------------

                        echo "=== Waiting For Application ==="

                        sleep 10


                        // ---------------------------------
                        // Health Check
                        // ---------------------------------

                        echo "=== Application Health Check ==="

                        curl -f \
                            http://localhost:8081/api/hello


                        echo ""

                        echo "================================="
                        echo "     DEPLOYMENT SUCCESSFUL 🚀"
                        echo "================================="


                        // ---------------------------------
                        // Docker Logout
                        // ---------------------------------

                        echo "=== Docker Logout ==="

                        docker logout
                    '''
                }
            }
        }
    }


    // =========================================
    // POST ACTIONS
    // =========================================
    post {

        success {

            echo '''
=================================
      CI/CD SUCCESSFUL 🚀
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
Application Running ✅
=================================
'''
        }

        failure {

            echo '''
=================================
       CI/CD FAILED ❌
=================================

Check the failed stage
in Jenkins Console Output.
=================================
'''
        }
    }
}
