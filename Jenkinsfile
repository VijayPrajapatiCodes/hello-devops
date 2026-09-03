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

                        echo "=== Javac Version ==="
                        javac -version

                        echo "=== JAVA_HOME ==="
                        echo $JAVA_HOME

                        echo "=== Maven Version ==="
                        ./mvnw -version

                        echo "=== Maven Build ==="
                        ./mvnw clean package
                    '''
                }
            }
        }

       stage('Docker Push') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'dockerhub-credentials',
                usernameVariable: 'vijayprajapaticodes',
                passwordVariable: 'dckr_pat_DsjnvgemKWkkP0DWpZi0aotxl6w'
            )
        ]) {
            sh '''
                echo "=== Docker Login ==="
                echo "dckr_pat_DsjnvgemKWkkP0DWpZi0aotxl6w" | docker login \
                    -u "vijayprajapaticodes" \
                    --password-stdin

                echo "=== Docker Tag ==="
                docker tag hello-devops:latest \
                    vijayprajapaticodes/hello-devops:latest

                echo "=== Docker Push ==="
                docker push \
                    vijayprajapaticodes/hello-devops:latest

                echo "=== Docker Logout ==="
                docker logout
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
