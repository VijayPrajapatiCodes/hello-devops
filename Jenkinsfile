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
                sh './mvnw clean package'
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
