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
                        echo "=== Java used for build ==="
                        java -version

                        echo "=== Javac used for build ==="
                        javac -version

                        echo "=== JAVA_HOME ==="
                        echo $JAVA_HOME

                        echo "=== Maven Java ==="
                        ./mvnw -version

                        echo "=== Building Spring Boot ==="
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
