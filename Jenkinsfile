pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user"
        EC2_HOST = "13.60.24.160"
        SERVICE_NAME = "config-server"
        SERVER_PORT = "8888"
        LOG_FILE = "config-server.log"
        SSH_CREDENTIALS_ID = "ec2-linux-key"  // Must match Jenkins SSH credential ID
    }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out Config Service source code..."
                git branch: 'main', url: 'https://github.com/Pransquare/ConfigService.git'
            }
        }

        stage('Build') {
            steps {
                echo "Building Config Server JAR..."
                dir('config') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    echo "Deploying Config Server to EC2 (${EC2_HOST})..."

                    // Define remote connection
                    def remote = [
                        name: "ec2-configserver",
                        host: "${EC2_HOST}",
                        user: "ec2-user",
                        allowAnyHosts: true,
                        credentialsId: "${SSH_CREDENTIALS_ID}"
                    ]

                    // Copy the JAR
                    echo "Copying JAR to EC2..."
                    sshPut remote: remote, from: "config/target/${SERVICE_NAME}.jar", into: "${DEPLOY_DIR}/"

                    // Stop old process safely
                    echo "Stopping old Config Server instance..."
                    sshCommand remote: remote, command: """
                        pkill -f ${SERVICE_NAME}.jar || echo 'No existing process found'
                    """

                    // Start new process in background
                    echo "Starting new Config Server..."
                    sshCommand remote: remote, command: """
                        nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar \
                        --server.port=${SERVER_PORT} > ${DEPLOY_DIR}/${LOG_FILE} 2>&1 &
                        echo \$! > ${DEPLOY_DIR}/${SERVICE_NAME}.pid
                    """

                    // Wait a few seconds to let the service start
                    echo "Waiting for the service to start..."
                    sshCommand remote: remote, command: "sleep 5"

                    // Check running process
                    echo "Checking running process..."
                    sshCommand remote: remote, command: "ps -ef | grep ${SERVICE_NAME}.jar | grep -v grep || echo 'Process not found'"

                    // Show last 10 log lines
                    echo "Showing last 10 log lines..."
                    sshCommand remote: remote, command: "tail -n 10 ${DEPLOY_DIR}/${LOG_FILE}"
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Config Server deployment failed. Check Jenkins console logs for details."
        }
        success {
            echo "✅ Config Server deployed successfully! Accessible at: http://${EC2_HOST}:${SERVER_PORT}/"
        }
    }
}
