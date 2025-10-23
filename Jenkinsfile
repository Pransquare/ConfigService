pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user/config-server"
        EC2_HOST = "16.171.155.143"
        SERVICE_NAME = "config-server"
        PEM_PATH = "C:/Users/KRISHNA/Downloads/ec2-linux-key.pem"
    }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/Pransquare/ConfigService.git', branch: 'main'
            }
        }

        stage('Build') {
            steps {
                dir('config') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                bat """
                echo ===== Creating deploy directory on EC2 if not exists =====
                ssh -i "${PEM_PATH}" -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} "mkdir -p ${DEPLOY_DIR}"

                echo ===== Copying JAR to EC2 =====
                scp -i "${PEM_PATH}" -o StrictHostKeyChecking=no config/target/${SERVICE_NAME}.jar ec2-user@${EC2_HOST}:${DEPLOY_DIR}/

                echo ===== Stopping old ${SERVICE_NAME} instance if running =====
                ssh -i "${PEM_PATH}" -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} "pkill -f ${SERVICE_NAME}.jar || true"

                echo ===== Starting new ${SERVICE_NAME} instance =====
                ssh -i "${PEM_PATH}" -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} "nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar --server.port=8888 > ${DEPLOY_DIR}/${SERVICE_NAME}.log 2>&1 &"

                echo  Deployment completed successfully!
                """
            }
        }
    }

    post {
        failure {
            echo " Deployment failed. Check Jenkins console logs for details."
        }
    }
}
