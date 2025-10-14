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
                echo "📦 Checking out code..."
                git branch: 'main', url: 'https://github.com/Pransquare/ConfigService.git'
            }
        }

        stage('Clean old JAR if locked') {
            steps {
                echo "🧹 Cleaning up any locked JAR..."
                bat """
                    if exist config\\target\\config-server.jar (
                        echo Killing any process using config-server.jar...
                        for /f "tokens=2" %%a in ('handle.exe config-server.jar ^| findstr "pid:"') do taskkill /PID %%a /F
                        echo Removing old jar...
                        del /F /Q config\\target\\config-server.jar
                    )
                """
            }
        }

        stage('Build') {
            steps {
                echo "🏗️ Building JAR..."
                dir('config') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Deploy to EC2 via WinRM HTTPS') {
            steps {
                echo "🚀 Deploying to EC2 (${EC2_HOST}) via WinRM (HTTPS)..."

                powershell """
                    try {
                        # --- Credentials ---
                        \$secPassword = ConvertTo-SecureString '${EC2_PASS}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential('${EC2_USER}', \$secPassword)
                        \$sessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                        # --- Verify EC2 connection ---
                        Write-Host "Testing WinRM connection to ${EC2_HOST}..."
                        Test-WSMan -ComputerName '${EC2_HOST}' -Port 5986 -UseSSL | Out-Null

                        # --- Create session ---
                        Write-Host "Connecting securely to EC2..."
                        \$session = New-PSSession -ComputerName '${EC2_HOST}' -Credential \$cred -Authentication Basic -UseSSL -Port 5986 -SessionOption \$sessionOption

                        if (-not \$session) {
                            throw "Failed to create WinRM session to ${EC2_HOST}"
                        }

                        # --- Ensure deployment directory exists ---
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$deployDir)
                            if (-not (Test-Path \$deployDir)) {
                                New-Item -ItemType Directory -Force -Path \$deployDir | Out-Null
                            }
                        } -ArgumentList '${DEPLOY_DIR}'

                        # --- Copy the JAR file ---
                        Write-Host "Copying new JAR to EC2..."
                        Copy-Item -Path 'config\\target\\config-server.jar' -Destination '${DEPLOY_DIR}\\config-server.jar' -ToSession \$session -Force

                        # --- Restart the Java process on EC2 ---
                        Write-Host "Restarting Java service on EC2..."
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$deployDir, \$serviceName, \$servicePort)

                            Write-Host "Stopping old Java process..."
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
            echo "✅ Cloud Config deployed successfully to EC2 via HTTPS!"
        }
        failure {
            echo "❌ Cloud Config deployment failed!"
        }
    }
}
