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
                        echo "=== Docker Login ==="
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "=== Pull Latest Image ==="
                        docker pull $DOCKER_USERNAME/hello-devops:latest

                        echo "=== Stop Old Container ==="
                        docker stop hello-devops || true

                        echo "=== Remove Old Container ==="
                        docker rm hello-devops || true

                        echo "=== Start New Container ==="
                        docker run -d \
                            --name hello-devops \
                            -p 8081:8080 \
                            $DOCKER_USERNAME/hello-devops:latest

                        echo "=== Running Containers ==="
                        docker ps

                        echo "=== Docker Logout ==="
                        docker logout
                    '''
                }
            }
        }
