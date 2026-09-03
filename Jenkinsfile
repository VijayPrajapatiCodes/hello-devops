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
                        echo "=== Java used by Jenkins build ==="
                        java -version

                        echo "=== Javac used by Jenkins build ==="
                        javac -version

                        echo "=== JAVA_HOME ==="
                        echo $JAVA_HOME

                        echo "=== Maven Java ==="
                        ./mvnw -version

                        echo "=== Maven Build ==="
                        ./mvnw clean package
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Build Successful! 🚀'
        }

        failure {
            echo 'Build Failed! ❌'
        }
    }
}
