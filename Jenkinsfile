pipeline {
    agent any
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "C:\\Deployments"
        SERVICE_NAME = "config-service"
        SERVICE_PORT = "8888"
        S3_BUCKET = "eureka-deployment-2025"
        REGION = "eu-north-1"
        EC2_HOST = "13.53.193.215"
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
                        file: "config\\target\\config-service.jar" // correct path
                    )
                }
            }
        }

        stage('Deploy to EC2 via WinRM') {
            steps {
                powershell """
                    # Credentials
                    \$username = 'Administrator'
                    \$password = 'd8%55Ir.%Z!hNR%VgUe-07OYX0ujLy;S'
                    \$secPassword = ConvertTo-SecureString \$password -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential(\$username, \$secPassword)

                    # Session options
                    \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                    # Create WinRM session
                    \$session = New-PSSession -ComputerName '${EC2_HOST}' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption

                    # Commands to deploy JAR
                    Invoke-Command -Session \$session -ScriptBlock {
                        if (-Not (Test-Path '${DEPLOY_DIR}')) {
                            New-Item -ItemType Directory -Path '${DEPLOY_DIR}'
                        }

                        # Download JAR from S3
                        aws s3 cp s3://${S3_BUCKET}/${SERVICE_NAME}.jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar

                         $javaProcesses = Get-WmiObject Win32_Process | Where-Object { $_.CommandLine -match '${SERVICE_NAME}.jar' }
    foreach ($proc in $javaProcesses) {
        Stop-Process -Id $proc.ProcessId -Force
    }

                        # Start the new JAR
                        Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar --server.port=${SERVICE_PORT}" -WindowStyle Hidden
                    }

                    # Close session
                    Remove-PSSession \$session
                """
            }
        }
    }
}
