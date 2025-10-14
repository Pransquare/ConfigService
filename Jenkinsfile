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
                echo "📦 Checking out code..."
                git branch: 'main', url: 'https://github.com/Pransquare/ConfigService.git'
            }
        }

        stage('Build') {
            steps {
                echo "🔨 Building JAR..."
                dir('config') { // Directory containing pom.xml
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Deploy to EC2 via WinRM') {
            steps {
                echo "🚀 Deploying to EC2 via WinRM..."
                powershell """
                    # --- Setup Credentials ---
                    \$secPassword = ConvertTo-SecureString '${EC2_PASS}' -AsPlainText -Force
                    \$cred = New-Object System.Management.Automation.PSCredential('${EC2_USER}', \$secPassword)
                    \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                    # --- Create WinRM Session ---
                    \$session = New-PSSession -ComputerName '${EC2_HOST}' -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption

                    # --- Copy JAR from Jenkins to EC2 ---
                    Copy-Item -Path "config\\target\\config-server.jar" -Destination "${DEPLOY_DIR}\\config-server.jar" -ToSession \$session -Force

                    # --- Execute remote commands on EC2 ---
                    Invoke-Command -Session \$session -ScriptBlock {
                        param(\$deployDir, \$serviceName, \$servicePort)

                        Write-Host "Stopping any existing Java process for \$serviceName..."
                        Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
                            Where-Object { \$_.CommandLine -like "*\$serviceName.jar*" } |
                            ForEach-Object { Stop-Process -Id \$_.ProcessId -Force }

                        Write-Host "Starting new service..."
                        Start-Process -FilePath 'java' -ArgumentList "-jar \$deployDir\\\$serviceName.jar --server.port=\$servicePort" -WindowStyle Hidden
                    } -ArgumentList '${DEPLOY_DIR}', '${SERVICE_NAME}', '${SERVICE_PORT}'

                    # --- Close session ---
                    Remove-PSSession \$session
                """
            }
        }
    }

    post {
        success {
            echo "✅ Cloud Config deployed successfully!"
        }
        failure {
            echo "❌ Cloud Config deployment failed!"
        }
    }
}
