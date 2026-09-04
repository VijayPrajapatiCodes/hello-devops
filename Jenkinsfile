pipeline {

    agent any

    environment {
        IMAGE_NAME = 'hello-devops'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'

                sh '''
                    set -e

                    git fetch --tags --force

                    echo "================================="
                    echo "        EXISTING GIT TAGS"
                    echo "================================="

                    git tag --sort=-v:refname | head -10 || true
                '''
            }
        }

        stage('Calculate Version') {
            steps {

                script {

                    def latestTag = sh(
                        script: '''
                            git tag --list 'v[0-9]*.[0-9]*.[0-9]*' \
                            --sort=-v:refname | head -1
                        ''',
                        returnStdout: true
                    ).trim()

                    if (!latestTag) {

                        env.VERSION = '1.0.0'

                    } else {

                        def version = latestTag.substring(1)
                        def parts = version.tokenize('.')

                        def major = parts[0].toInteger()
                        def minor = parts[1].toInteger()
                        def patch = parts[2].toInteger() + 1

                        env.VERSION = "${major}.${minor}.${patch}"
                    }

                    env.GIT_TAG = "v${env.VERSION}"
                    env.IMAGE_TAG = env.VERSION

                    echo """
=================================
       VERSION INFORMATION
=================================

Previous Git Tag:
${latestTag ?: 'NONE'}

New Version:
${env.VERSION}

Git Tag:
${env.GIT_TAG}

Docker Tag:
${env.IMAGE_TAG}

=================================
"""
                }
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
                        echo "     BUILD SUCCESSFUL"
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
                    echo "        DOCKER BUILD"
                    echo "================================="

                    docker --version

                    docker build \
                        -t "$IMAGE_NAME:$IMAGE_TAG" \
                        -t "$IMAGE_NAME:latest" \
                        .

                    echo "================================="
                    echo "      DOCKER BUILD SUCCESSFUL"
                    echo "================================="

                    docker images "$IMAGE_NAME"
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
                            "$IMAGE_NAME:$IMAGE_TAG" \
                            "$DOCKER_USERNAME/$IMAGE_NAME:$IMAGE_TAG"

                        docker tag \
                            "$IMAGE_NAME:latest" \
                            "$DOCKER_USERNAME/$IMAGE_NAME:latest"

                        echo "================================="
                        echo "        PUSH VERSION"
                        echo "================================="

                        docker push \
                            "$DOCKER_USERNAME/$IMAGE_NAME:$IMAGE_TAG"

                        echo "================================="
                        echo "        PUSH LATEST"
                        echo "================================="

                        docker push \
                            "$DOCKER_USERNAME/$IMAGE_NAME:latest"

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

        stage('Create Git Tag') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-credentials',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "================================="
                        echo "       CREATING GIT TAG"
                        echo "================================="

                        git config user.name "Jenkins"
                        git config user.email "jenkins@localhost"

                        git tag -a "$GIT_TAG" \
                            -m "Release $GIT_TAG"

                        echo "Git tag created:"
                        git tag -l "$GIT_TAG"

                        echo "================================="
                        echo "       PUSHING GIT TAG"
                        echo "================================="

                        cat > "$WORKSPACE/git-askpass.sh" <<'EOF'
#!/bin/sh
case "$1" in
    *Username*) echo "$GITHUB_USERNAME" ;;
    *Password*) echo "$GITHUB_TOKEN" ;;
esac
EOF

                        chmod 700 "$WORKSPACE/git-askpass.sh"

                        export GIT_ASKPASS="$WORKSPACE/git-askpass.sh"
                        export GIT_TERMINAL_PROMPT=0

                        git push origin "$GIT_TAG"

                        rm -f "$WORKSPACE/git-askpass.sh"

                        echo "================================="
                        echo "       GIT TAG PUSHED"
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

                        echo "Deploying Version:"
                        echo "$IMAGE_TAG"

                        echo "=== Docker Login ==="

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        cd "$WORKSPACE"

                        echo "=== Deployment Directory ==="

                        pwd

                        echo "=== Checking Compose File ==="

                        test -f docker-compose.yml

                        echo "docker-compose.yml found"

                        echo "=== Setting Deployment Version ==="

                        export IMAGE_TAG="$IMAGE_TAG"
                        export DOCKER_USERNAME="$DOCKER_USERNAME"

                        echo "Docker Image:"
                        echo "$DOCKER_USERNAME/$IMAGE_NAME:$IMAGE_TAG"

                        echo "=== Docker Compose Config ==="

                        docker compose config

                        echo "=== Stopping Existing Compose Application ==="

                        docker compose down \
                            --remove-orphans \
                            2>/dev/null || true

                        echo "=== Removing Existing hello-devops Container ==="

                        docker rm -f hello-devops \
                            2>/dev/null || true

                        echo "=== Checking Docker Containers Using Port 8081 ==="

                        PORT_CONTAINERS=$(docker ps -aq --filter "publish=8081")

                        if [ -n "$PORT_CONTAINERS" ]; then

                            echo "Containers using port 8081:"
                            echo "$PORT_CONTAINERS"

                            docker rm -f $PORT_CONTAINERS

                        else

                            echo "No Docker container is using port 8081"

                        fi

                        echo "=== Checking HOST Port 8081 ==="

                        if sudo -n fuser 8081/tcp >/dev/null 2>&1; then

                            sudo -n fuser -v 8081/tcp || true

                            sudo -n fuser -k 8081/tcp

                            sleep 3

                        fi

                        echo "=== Final Port Verification ==="

                        if sudo -n fuser 8081/tcp >/dev/null 2>&1; then

                            echo "ERROR: PORT 8081 STILL OCCUPIED"

                            sudo -n fuser -v 8081/tcp || true

                            exit 1

                        fi

                        echo "Port 8081 is completely free"

                        echo "=== Pulling Versioned Image ==="

                        docker compose pull

                        echo "=== Starting Version $IMAGE_TAG ==="

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

                        echo "Deployed Version:"
                        echo "$IMAGE_TAG"

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
Semantic Version
   ↓
Git Tag
   ↓
Maven Build
   ↓
Docker Build
   ↓
Docker Hub
   ↓
Versioned Image
   ↓
AWS EC2
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
