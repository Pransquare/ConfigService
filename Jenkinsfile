pipeline {
    agent any
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "C:\\Deployments"
        SERVICE_NAME = "config-server"
        SERVICE_PORT = "8888"
        S3_BUCKET = "eureka-deployment-2025"
        REGION = "eu-north-1"
        EC2_HOST = "13.53.193.215"
        EC2_USER = "Administrator"
        EC2_PASS = "d8%55Ir.%Z!hNR%VgUe-07OYX0ujLy;S"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Pransquare/ConfigService.git'
            }
        }

        stage('Build') {
            steps {
                dir('config') { // directory containing pom.xml
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Upload JAR to S3') {
            steps {
                withAWS(credentials: 'aws-jenkins-creds', region: "${REGION}") {
                    s3Upload(
                        bucket: "${S3_BUCKET}",
                        path: "${SERVICE_NAME}.jar",
                        file: "config\\target\\config-server.jar"
                    )
                }
            }
        }

        stage('Deploy to EC2 via WinRM') {
            steps {
                powershell """
                    # Convert password to secure string
                    \$secPassword = ConvertTo-SecureString '${EC2_PASS}' -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential('${EC2_USER}', \$secPassword)

                    # WinRM session options
                    \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                    # Create WinRM session
                    \$session = New-PSSession -ComputerName '${EC2_HOST}' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption

                    # Deploy commands inside session
                    Invoke-Command -Session \$session -ScriptBlock {
                        param(\$deployDir, \$serviceName, \$servicePort, \$s3Bucket)

                        # Create deployment directory if it doesn't exist
                        if (-Not (Test-Path \$deployDir)) {
                            New-Item -ItemType Directory -Path \$deployDir
                        }

                        # Download the JAR from S3
                        aws s3 cp s3://\$s3Bucket/\$serviceName.jar \$deployDir\\\$serviceName.jar

                        # Stop only the Java process running this service
                        Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
                            Where-Object { \$_.CommandLine -like "*\$serviceName.jar*" } |
                            ForEach-Object { Stop-Process -Id \$_.ProcessId -Force }

                        # Start the new JAR
                        Start-Process -FilePath 'java' -ArgumentList "-jar \$deployDir\\\$serviceName.jar --server.port=\$servicePort" -WindowStyle Hidden
                    } -ArgumentList '${DEPLOY_DIR}', '${SERVICE_NAME}', '${SERVICE_PORT}', '${S3_BUCKET}'

                    # Close session
                    Remove-PSSession \$session
                """
            }
        }
    }
}
