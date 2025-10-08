pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "C:\\Apps\\cloud-config"
        SERVICE_NAME = "cloud-config"
        SERVICE_PORT = "8888"   // Adjust if your Cloud Config runs on a different port
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out Cloud Config repo..."
                git branch: 'main', url: 'https://github.com/Pransquare/ConfigService.git'
            }
        }

        stage('Build') {
            steps {
                echo "Building Cloud Config..."
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying Cloud Config..."
                // Kill any previous Cloud Config process
                bat 'taskkill /F /IM java.exe || exit 0'
                // Create deployment folder if not exists
                bat "if not exist %DEPLOY_DIR% mkdir %DEPLOY_DIR%"
                // Copy JAR to deploy folder
                bat "xcopy target\\*.jar %DEPLOY_DIR% /Y /Q"
                // Start Cloud Config server
                bat "start cmd /c java -jar %DEPLOY_DIR%\\%SERVICE_NAME%.jar --server.port=%SERVICE_PORT%"
            }
        }
    }

    post {
        always {
            echo "Cloud Config deployment finished."
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
