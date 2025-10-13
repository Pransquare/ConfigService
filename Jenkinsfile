pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        EC2_HOST = "ec2-13-233-123-45.ap-south-1.compute.amazonaws.com"   // Replace with your EC2 public DNS
        EC2_USER = "ec2-user"                                             // or "ubuntu" if your AMI uses that
        DEPLOY_DIR = "/home/ec2-user/cloud-config"
        SERVICE_NAME = "cloud-config"
        SERVICE_PORT = "8888"
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
                dir('Config') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Deploy to AWS') {
            steps {
                echo "Deploying Cloud Config to AWS EC2..."

                // Use direct SSH command instead of Jenkins SSH key
                bat """
                echo Deploying to AWS EC2...
                pscp -i C:\\path\\to\\your\\pemfile.pem Config\\target\\*.jar ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}\\
                plink -i C:\\path\\to\\your\\pemfile.pem ${EC2_USER}@${EC2_HOST} "pkill -f cloud-config.jar || true && nohup java -jar ${DEPLOY_DIR}/cloud-config-*.jar --server.port=${SERVICE_PORT} > ${DEPLOY_DIR}/app.log 2>&1 &"
                """
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
