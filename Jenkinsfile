pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "C:\\Deployments"
        SERVICE_NAME = "configserver"
        EC2_HOST = "13.60.54.132"
        EC2_USER = "Administrator"
        EC2_PASSWORD = "d)I*hG!qVf2@YB95bxN=(oungEA5!M$S"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Cloning Cloud Config repository..."
                git branch: 'main', url: 'https://github.com/Pransquare/ConfigService.git', credentialsId: 'Git_credentials'
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

        stage('Deploy to EC2 via WinRM') {
            steps {
                echo "Deploying Cloud Config to EC2 via WinRM..."
                powershell """
                    \$username = '${EC2_USER}'
                    \$password = '${EC2_PASSWORD}'
                    \$secPassword = ConvertTo-SecureString \$password -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential(\$username, \$secPassword)

                    \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
                    \$session = New-PSSession -ComputerName '${EC2_HOST}' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption

                    # Copy JAR from Jenkins workspace to EC2
                    Copy-Item -Path 'Config\\target\\${SERVICE_NAME}.jar' -Destination "\${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -ToSession \$session -Force

                    # Run the JAR on EC2
                    Invoke-Command -Session \$session -ScriptBlock {
                        param (\$deployDir, \$serviceName)
                        
                        # Stop existing Java process
                        Get-Process -Name java -ErrorAction SilentlyContinue | Stop-Process -Force

                        # Start the new JAR
                        Start-Process -FilePath 'java' -ArgumentList "-jar \$deployDir\\\$serviceName.jar" -WindowStyle Hidden
                    } -ArgumentList '${DEPLOY_DIR}', '${SERVICE_NAME}'

                    Remove-PSSession \$session
                """
            }
        }
    }

    post {
        success {
            echo "Cloud Config deployed successfully!"
        }
        failure {
            echo "Cloud Config deployment failed!"
        }
    }
}
