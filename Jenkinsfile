pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "C:\\Deployments"
        SERVICE_NAME = "cloud-config"
        SERVICE_PORT = "8888"
        S3_BUCKET = "cloud-config-deployment-2025"
        REGION = "ap-south-1"
        EC2_HOST = "13.53.193.215"
        EC2_USER = "Administrator"
        EC2_PASS = "d)I*hG!qVf2@YB95bxN=(oungEA5!M$S"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Cloning Cloud Config repository..."
                git branch: 'main', url: 'https://github.com/Pransquare/ConfigService.git'
            }
        }

        stage('Build') {
            steps {
                echo "Building Cloud Config..."
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Upload JAR to S3') {
            steps {
                echo "Uploading JAR to S3..."
                powershell """
                    # Find the built JAR without hardcoding version
                    \$jar = Get-ChildItem -Path '${env.WORKSPACE}\\Config\\target\\${SERVICE_NAME}*.jar' | Select-Object -First 1
                    if (-Not \$jar) { throw "JAR not found!" }
                    
                    # Upload to S3
                    aws s3 cp \$jar.FullName s3://${S3_BUCKET}/${SERVICE_NAME}.jar
                """
            }
        }

        stage('Deploy to EC2 via WinRM') {
            steps {
                echo "Deploying Cloud Config to EC2..."
                powershell """
                    # Credentials
                    \$secPassword = ConvertTo-SecureString '${EC2_PASS}' -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential('${EC2_USER}', \$secPassword)

                    # Skip certificate checks
                    \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                    # Create WinRM session
                    \$session = New-PSSession -ComputerName '${EC2_HOST}' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption

                    # Deploy commands
                    Invoke-Command -Session \$session -ScriptBlock {
                        if (-Not (Test-Path '${DEPLOY_DIR}')) {
                            New-Item -ItemType Directory -Path '${DEPLOY_DIR}'
                        }

                        # Download latest JAR from S3
                        aws s3 cp s3://${S3_BUCKET}/${SERVICE_NAME}.jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar

                        # Stop existing process
                        Get-Process -Name java -ErrorAction SilentlyContinue | Stop-Process -Force

                        # Start new JAR
                        Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar --server.port=${SERVICE_PORT}" -WindowStyle Hidden
                    }

                    # Close session
                    Remove-PSSession \$session
                """
            }
        }
    }

    post {
        always {
            echo "Cloud Config pipeline finished."
        }
        failure {
            echo "Cloud Config deployment failed!"
        }
    }
}
