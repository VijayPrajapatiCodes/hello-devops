pipeline {

    agent any

    environment {
        IMAGE_NAME = 'hello-devops'
    }

    stages {

        // =====================================================
        // 1. CHECKOUT
        // =====================================================

        stage('Checkout') {
            steps {

                echo 'Checking out source code...'

                checkout scm

                sh '''
                    set -e

                    git fetch --tags --force

                    echo "================================="
                    echo "        GIT INFORMATION"
                    echo "================================="

                    echo "Branch:"
                    git branch --show-current

                    echo "Commit:"
                    git rev-parse --short HEAD

                    echo "Existing Tags:"
                    git tag --sort=-v:refname | head -10 || true
                '''
            }
        }


        // =====================================================
        // 2. WEBHOOK PROTECTION
        // =====================================================

        stage('Webhook Protection') {
            steps {

                script {

                    def tagBuild = sh(
                        script: '''
                            if [ -n "${GIT_TAG_NAME:-}" ]; then
                                echo "true"
                            elif git describe --exact-match --tags HEAD >/dev/null 2>&1; then
                                echo "true"
                            else
                                echo "false"
                            fi
                        ''',
                        returnStdout: true
                    ).trim()

                    if (tagBuild == 'true') {

                        echo """
=================================
       WEBHOOK PROTECTION
=================================

This build was triggered by a Git tag.

Tag builds are not allowed to
start another CI/CD release.

Stopping pipeline safely.

=================================
"""

                        currentBuild.result = 'NOT_BUILT'

                        error(
                            'Tag build detected. Pipeline stopped to prevent webhook loop.'
                        )

                    } else {

                        echo """
=================================
         NORMAL PUSH BUILD
=================================

This is a normal branch build.

CI/CD will continue.

=================================
"""
                    }
                }
            }
        }


        // =====================================================
        // 3. CALCULATE VERSION
        // =====================================================

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

                    // First release

                    if (!latestTag) {

                        env.VERSION = '1.0.0'
                        env.PREVIOUS_VERSION = ''

                    } else {

                        // Example:
                        // v1.0.5
                        //   ↓
                        // 1.0.5

                        def version = latestTag.substring(1)

                        def parts = version.tokenize('.')

                        def major = parts[0].toInteger()
                        def minor = parts[1].toInteger()
                        def patch = parts[2].toInteger() + 1

                        env.VERSION = "${major}.${minor}.${patch}"

                        // Previous production version
                        env.PREVIOUS_VERSION = version
                    }

                    // Git tag
                    env.GIT_TAG = "v${env.VERSION}"

                    // Docker tag
                    env.IMAGE_TAG = env.VERSION

                    echo """
=================================
       VERSION INFORMATION
=================================

Previous Git Tag:
${latestTag ?: 'NONE'}

Previous Deploy Version:
${env.PREVIOUS_VERSION ?: 'NONE'}

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


        // =====================================================
        // 4. MAVEN BUILD
        // =====================================================

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


        // =====================================================
        // 5. DOCKER BUILD
        // =====================================================

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


        // =====================================================
        // 6. DOCKER PUSH
        // =====================================================

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

                        docker logout

                        echo "================================="
                        echo "     DOCKER PUSH SUCCESSFUL"
                        echo "================================="
                    '''
                }
            }
        }


        // =====================================================
        // 7. CREATE GIT TAG
        // =====================================================

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


        // =====================================================
        // 8. DEPLOY + HEALTH CHECK + AUTOMATIC ROLLBACK
        // =====================================================

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

                        echo "Previous Version:"
                        echo "${PREVIOUS_VERSION:-NONE}"


                        # =====================================
                        # DOCKER LOGIN
                        # =====================================

                        echo "=== Docker Login ==="

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin


                        # =====================================
                        # DEPLOYMENT DIRECTORY
                        # =====================================

                        cd "$WORKSPACE"

                        echo "=== Deployment Directory ==="

                        pwd


                        # =====================================
                        # CHECK COMPOSE FILE
                        # =====================================

                        echo "=== Checking Compose File ==="

                        test -f docker-compose.yml

                        echo "docker-compose.yml found"


                        # =====================================
                        # SET VERSION
                        # =====================================

                        echo "=== Setting Deployment Version ==="

                        export IMAGE_TAG="$IMAGE_TAG"

                        export DOCKER_USERNAME="$DOCKER_USERNAME"

                        echo "Docker Image:"

                        echo "$DOCKER_USERNAME/$IMAGE_NAME:$IMAGE_TAG"


                        # =====================================
                        # COMPOSE CONFIG VALIDATION
                        # =====================================

                        echo "=== Docker Compose Config ==="

                        docker compose config


                        # =====================================
                        # STOP EXISTING APPLICATION
                        # =====================================

                        echo "=== Stopping Existing Compose Application ==="

                        docker compose down \
                            --remove-orphans \
                            2>/dev/null || true


                        # =====================================
                        # REMOVE EXISTING CONTAINER
                        # =====================================

                        echo "=== Removing Existing hello-devops Container ==="

                        docker rm -f hello-devops \
                            2>/dev/null || true


                        # =====================================
                        # CHECK DOCKER PORT
                        # =====================================

                        echo "=== Checking Docker Containers Using Port 8081 ==="

                        PORT_CONTAINERS=$(docker ps -aq --filter "publish=8081")

                        if [ -n "$PORT_CONTAINERS" ]; then

                            echo "Containers using port 8081:"

                            echo "$PORT_CONTAINERS"

                            docker rm -f $PORT_CONTAINERS

                        else

                            echo "No Docker container is using port 8081"

                        fi


                        # =====================================
                        # CHECK HOST PORT
                        # =====================================

                        echo "=== Checking HOST Port 8081 ==="

                        if sudo -n fuser 8081/tcp >/dev/null 2>&1; then

                            sudo -n fuser -v 8081/tcp || true

                            sudo -n fuser -k 8081/tcp

                            sleep 3

                        fi


                        # =====================================
                        # FINAL PORT CHECK
                        # =====================================

                        echo "=== Final Port Verification ==="

                        if sudo -n fuser 8081/tcp >/dev/null 2>&1; then

                            echo "ERROR: PORT 8081 STILL OCCUPIED"

                            sudo -n fuser -v 8081/tcp || true

                            exit 1

                        fi

                        echo "Port 8081 is completely free"


                        # =====================================
                        # PULL NEW IMAGE
                        # =====================================

                        echo "=== Pulling Versioned Image ==="

                        docker compose pull


                        # =====================================
                        # START NEW VERSION
                        # =====================================

                        echo "=== Starting Version $IMAGE_TAG ==="

                        docker compose up -d \
                            --force-recreate \
                            --remove-orphans


                        # =====================================
                        # DEPLOYMENT STATUS
                        # =====================================

                        echo "=== Deployment Status ==="

                        docker compose ps


                        # =====================================
                        # WAIT FOR APPLICATION
                        # =====================================

                        echo "=== Waiting For Application ==="

                        sleep 10


                        # =====================================
                        # HEALTH CHECK
                        # =====================================

                        echo "=== Application Health Check ==="

                        if curl -f --max-time 10 \
                            http://localhost:8081/api/hello
                        then

                            echo ""

                            echo "================================="

                            echo "     HEALTH CHECK PASSED"

                            echo "================================="

                            echo "Deployed Version:"

                            echo "$IMAGE_TAG"

                            echo ""

                            echo "================================="

                            echo "     DEPLOYMENT SUCCESSFUL"

                            echo "================================="

                            docker logout

                        else

                            # =================================
                            # HEALTH CHECK FAILED
                            # =================================

                            echo ""

                            echo "================================="

                            echo "     HEALTH CHECK FAILED"

                            echo "================================="

                            echo "New Version:"

                            echo "$IMAGE_TAG"

                            echo "Previous Version:"

                            echo "${PREVIOUS_VERSION:-NONE}"


                            # =================================
                            # NO PREVIOUS VERSION
                            # =================================

                            if [ -z "${PREVIOUS_VERSION:-}" ]; then

                                echo ""

                                echo "No previous version available."

                                echo "Rollback cannot be performed."

                                docker logout

                                exit 1

                            fi


                            # =================================
                            # START ROLLBACK
                            # =================================

                            echo ""

                            echo "================================="

                            echo "       STARTING ROLLBACK"

                            echo "================================="

                            echo "Rolling back to:"

                            echo "$PREVIOUS_VERSION"


                            # =================================
                            # STOP FAILED VERSION
                            # =================================

                            echo "=== Stopping Failed Version ==="

                            docker compose down \
                                --remove-orphans \
                                2>/dev/null || true


                            # =================================
                            # SET PREVIOUS VERSION
                            # =================================

                            echo "=== Setting Previous Version ==="

                            export IMAGE_TAG="$PREVIOUS_VERSION"

                            export DOCKER_USERNAME="$DOCKER_USERNAME"

                            echo "Rollback Image:"

                            echo "$DOCKER_USERNAME/$IMAGE_NAME:$IMAGE_TAG"


                            # =================================
                            # PULL PREVIOUS IMAGE
                            # =================================

                            echo "=== Pulling Previous Image ==="

                            docker compose pull


                            # =================================
                            # START PREVIOUS VERSION
                            # =================================

                            echo "=== Starting Previous Version ==="

                            docker compose up -d \
                                --force-recreate \
                                --remove-orphans


                            # =================================
                            # WAIT FOR ROLLBACK
                            # =================================

                            echo "=== Waiting For Rollback Application ==="

                            sleep 10


                            # =================================
                            # ROLLBACK HEALTH CHECK
                            # =================================

                            echo "=== Rollback Health Check ==="

                            if curl -f --max-time 10 \
                                http://localhost:8081/api/hello
                            then

                                echo ""

                                echo "================================="

                                echo "       ROLLBACK SUCCESSFUL"

                                echo "================================="

                                echo "Running Version:"

                                echo "$PREVIOUS_VERSION"

                                docker compose ps

                                docker logout

                                # Pipeline remains FAILED
                                # because the new release failed.

                                exit 1

                            else

                                echo ""

                                echo "================================="

                                echo "     ROLLBACK ALSO FAILED"

                                echo "================================="

                                docker compose ps

                                docker logout

                                exit 1

                            fi

                        fi
                    '''
                }
            }
        }
    }


    // =========================================================
    // POST ACTIONS
    // =========================================================

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
Health Check
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

Possible reasons:

- Build failed
- Docker build failed
- Docker push failed
- Git tag failed
- Deployment failed
- Health check failed
- Rollback executed

Check the failed stage
in Jenkins Console Output.

=================================
'''
        }
    }
}
