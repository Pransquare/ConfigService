pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR   = "C:\\Deployments"
        SERVICE_NAME = "config-server"
        SERVICE_PORT = "8888"
        EC2_HOST     = "13.53.132.62"
        EC2_USER     = "Administrator"
        EC2_PASS     = "zcriOxjGlLg0q*LJ$oaLnyB4ZII$RkpS"
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

        stage('Deploy to EC2 via WinRM HTTPS') {
            steps {
                echo "Deploying to EC2 via WinRM (HTTPS)..."

                powershell """
                    try {
                        # --- Credentials ---
                        \$secPassword = ConvertTo-SecureString '${EC2_PASS}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential('${EC2_USER}', \$secPassword)
                        \$sessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                        # --- Create WinRM HTTPS Session ---
                        Write-Host "Connecting to EC2 over HTTPS..."
                        \$session = New-PSSession -ComputerName '${EC2_HOST}' -Credential \$cred -Authentication Basic -UseSSL -Port 5986 -SessionOption \$sessionOption

                        if (-not \$session) {
                            throw "Failed to create PSSession to ${EC2_HOST}"
                        }

                        # --- Create Deploy Directory ---
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$deployDir)
                            if (-not (Test-Path \$deployDir)) {
                                New-Item -ItemType Directory -Force -Path \$deployDir | Out-Null
                            }
                        } -ArgumentList '${DEPLOY_DIR}'

                        # --- Copy JAR to EC2 ---
                        Write-Host "Copying JAR file to EC2..."
                        Copy-Item -Path 'config\\target\\config-server.jar' -Destination '${DEPLOY_DIR}\\config-server.jar' -ToSession \$session -Force

                        # --- Restart the Spring Boot service ---
                        Write-Host "Restarting Java service..."
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$deployDir, \$serviceName, \$servicePort)

                            Write-Host "Stopping existing process..."
                            Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
                                Where-Object { \$_.CommandLine -like "*\$serviceName.jar*" } |
                                ForEach-Object { Stop-Process -Id \$_.ProcessId -Force }

                            Write-Host "Starting new process..."
                            Start-Process "java" "-jar \$deployDir\\\$serviceName.jar --server.port=\$servicePort" -WindowStyle Hidden
                        } -ArgumentList '${DEPLOY_DIR}', '${SERVICE_NAME}', '${SERVICE_PORT}'

                        Write-Host "✅ Deployment completed successfully!"
                    }
                    catch {
                        Write-Error "❌ Deployment failed: \$($_.Exception.Message)"
                        exit 1
                    }
                    finally {
                        if (\$session) { Remove-PSSession \$session }
                    }
                """
            }
        }
    }

    post {
        success {
            echo "✅ Cloud Config deployed successfully on EC2 (HTTPS)!"
        }
        failure {
            echo "❌ Cloud Config deployment failed!"
        }
    }
}
