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
        S3_BUCKET = "cloud-config-deployment-2025"   // Your S3 bucket for Cloud Config
        REGION = "eu-north-1"
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
                echo "Building Cloud Config service..."
                dir('Config') {  // Adjust this if your pom.xml is in a subfolder
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Upload JAR to S3') {
            steps {
                echo "Uploading JAR to S3 bucket..."
                withAWS(credentials: 'aws-jenkins-creds', region: "${REGION}") {
                    s3Upload(bucket: "${S3_BUCKET}", path: "${SERVICE_NAME}.jar", file: "Config\\target\\${SERVICE_NAME}.jar")
                }
            }
        }

        stage('Deploy to EC2 via WinRM') {
            steps {
                echo "Deploying Cloud Config to EC2 via WinRM..."

                powershell """
                    # Temporary credentials (use Jenkins credentials plugin in production)
                    \$username = 'Administrator'
                    \$password = 'd)I*hG!qVf2@YB95bxN=(oungEA5!M$S'
                    \$secPassword = ConvertTo-SecureString \$password -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential(\$username, \$secPassword)

                    # Skip certificate validation
                    \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                    # Create secure WinRM session
                    \$session = New-PSSession -ComputerName '13.53.193.215' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption

                    # Execute deployment commands on remote EC2
                    Invoke-Command -Session \$session -ScriptBlock {
                        if (-Not (Test-Path '${DEPLOY_DIR}')) {
                            New-Item -ItemType Directory -Path '${DEPLOY_DIR}'
                        }

                        Write-Host "Downloading latest Cloud Config JAR from S3..."
                        aws s3 cp s3://${S3_BUCKET}/${SERVICE_NAME}.jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar --region ${REGION}

                        Write-Host "Stopping existing Cloud Config process..."
                        Get-Process -Name java -ErrorAction SilentlyContinue | Stop-Process -Force

                        Write-Host "Starting new Cloud Config service..."
                        Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar --server.port=${SERVICE_PORT}" -WindowStyle Hidden
                    }

                    # Close the remote session
                    Remove-PSSession \$session
                """
            }
        }
    }

    post {
        always {
            echo "Cloud Config pipeline completed."
        }
        success {
            echo "✅ Cloud Config deployed successfully!"
        }
        failure {
            echo "❌ Cloud Config deployment failed."
        }
    }
}
