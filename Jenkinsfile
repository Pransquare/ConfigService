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
                dir('Config') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Deploy to EC2 via WinRM') {
            steps {
                echo "Deploying Cloud Config to EC2 via WinRM..."

                powershell """
                    \$username = 'Administrator'
                    \$password = 'd8%55Ir.%Z!hNR%VgUe-07OYX0ujLy;S'
                    \$secPassword = ConvertTo-SecureString \$password -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential(\$username, \$secPassword)

                    \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                    \$session = New-PSSession -ComputerName '13.53.193.215' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption

                    Invoke-Command -Session \$session -ScriptBlock {
                        if (-Not (Test-Path '${DEPLOY_DIR}')) {
                            New-Item -ItemType Directory -Path '${DEPLOY_DIR}'
                        }

                        Copy-Item -Path "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\cloudconfig\\Config\\target\\configserver-0.0.1-SNAPSHOT.jar" -Destination "${DEPLOY_DIR}\\cloud-config.jar" -Force

                        Get-Process -Name java -ErrorAction SilentlyContinue | Stop-Process -Force

                        Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\cloud-config.jar --server.port=${SERVICE_PORT}" -WindowStyle Hidden
                    }

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
