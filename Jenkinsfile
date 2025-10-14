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
        EC2_HOST = "13.53.132.62"
        EC2_USER = "Administrator"
        EC2_PASS = "zcriOxjGlLg0q*LJ$oaLnyB4ZII$RkpS"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out code..."
                git branch: 'main', url: 'https://github.com/Pransquare/ConfigService.git'
            }
        }

        stage('Build') {
            steps {
                echo "Building JAR..."
                dir('config') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Deploy to EC2 via WinRM') {
            steps {
                echo "Deploying to EC2 via WinRM (HTTP) as specific user..."
                powershell """
                    try {
                        # --- Credentials ---
                        \$secPassword = ConvertTo-SecureString '${EC2_PASS}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential('${EC2_USER}', \$secPassword)

                        # --- Run all WinRM commands as the EC2 user ---
                        Start-Process powershell -Credential \$cred -ArgumentList {
                            try {
                                # Add EC2 host to TrustedHosts
                                Set-Item WSMan:\\localhost\\Client\\TrustedHosts -Value '${EC2_HOST}' -Force

                                Write-Host "Creating WinRM session to ${EC2_HOST}..."
                                \$session = New-PSSession -ComputerName '${EC2_HOST}' -Credential \$cred -Authentication Basic

                                Write-Host "Copying JAR to EC2..."
                                Copy-Item -Path 'config\\target\\config-server.jar' -Destination '${DEPLOY_DIR}\\config-server.jar' -ToSession \$session -Force

                                Write-Host "Stopping any existing Java process for ${SERVICE_NAME}..."
                                Invoke-Command -Session \$session -ScriptBlock {
                                    param(\$deployDir, \$serviceName, \$servicePort)
                                    Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
                                        Where-Object { \$_.CommandLine -like "*\$serviceName.jar*" } |
                                        ForEach-Object { Stop-Process -Id \$_.ProcessId -Force }

                                    Write-Host "Starting new service..."
                                    Start-Process -FilePath 'java' -ArgumentList "-jar \$deployDir\\\$serviceName.jar --server.port=\$servicePort" -WindowStyle Hidden
                                } -ArgumentList '${DEPLOY_DIR}', '${SERVICE_NAME}', '${SERVICE_PORT}'

                                Write-Host "Deployment completed successfully."
                            }
                            catch {
                                Write-Host "Deployment failed: \$($_.Exception.Message)"
                                exit 1
                            }
                            finally {
                                if (\$session) { Remove-PSSession \$session }
                            }
                        }
                    }
                    catch {
                        Write-Host "Pipeline failed: \$($_.Exception.Message)"
                        exit 1
                    }
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
