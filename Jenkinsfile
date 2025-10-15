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
        echo "Building Cloud Config JAR..."
        // Stop any running java process first
        bat '''
        echo Stopping existing Java processes...
        taskkill /F /IM java.exe || echo No running Java process found.
        '''
        dir('config') {
            bat 'mvn clean package -DskipTests'
        }
    }
}


        stage('Deploy to EC2 via WinRM (HTTPS)') {
            steps {
                echo "Deploying Cloud Config to EC2 (${EC2_HOST}) via WinRM HTTPS..."

                powershell """
                    try {
                        # --- Credentials ---
                        \$secPassword = ConvertTo-SecureString '${EC2_PASS}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential('${EC2_USER}', \$secPassword)

                        # --- Skip SSL certificate checks (since we use self-signed cert) ---
                        \$sessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                        # --- Create secure WinRM session ---
                        Write-Host "Connecting to EC2 via HTTPS..."
                        \$session = New-PSSession -ComputerName '${EC2_HOST}' -Credential \$cred -UseSSL -Authentication Basic -Port 5986 -SessionOption \$sessionOption

                        if (-not \$session) {
                            throw "Failed to establish WinRM HTTPS session to ${EC2_HOST}"
                        }

                        # --- Ensure deployment directory exists on EC2 ---
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$deployDir)
                            if (-not (Test-Path \$deployDir)) {
                                New-Item -ItemType Directory -Force -Path \$deployDir | Out-Null
                            }
                        } -ArgumentList '${DEPLOY_DIR}'

                        # --- Copy new JAR file to EC2 ---
                        Write-Host "Copying JAR to EC2..."
                        Copy-Item -Path 'config\\target\\config-server.jar' -Destination '${DEPLOY_DIR}\\config-server.jar' -ToSession \$session -Force

                        # --- Stop any existing Java process running the same JAR ---
                        Write-Host "Stopping existing Java process..."
                        Invoke-Command -Session \$session -ScriptBlock {
                            Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
                                Where-Object { \$_.CommandLine -like "*config-server.jar*" } |
                                ForEach-Object { Stop-Process -Id \$_.ProcessId -Force }
                        }

                        # --- Start the new JAR on EC2 ---
                        Write-Host "Starting new config-server on port ${SERVICE_PORT}..."
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$deployDir, \$serviceName, \$servicePort)
                            Start-Process "java" "-jar \$deployDir\\\$serviceName.jar --server.port=\$servicePort" -WindowStyle Hidden
                        } -ArgumentList '${DEPLOY_DIR}', '${SERVICE_NAME}', '${SERVICE_PORT}'

                        Write-Host "Cloud Config successfully deployed to EC2!"
                    }
                    catch {
                        Write-Error "Deployment failed: \$($_.Exception.Message)"
                        exit 1
                    }
                    finally {
                        if (\$session) {
                            Write-Host "Closing remote session..."
                            Remove-PSSession \$session
                        }
                    }
                """
            }
        }
    }

    post {
        success {
            echo "Cloud Config deployed successfully on EC2 (HTTPS)!"
        }
        failure {
            echo "Cloud Config deployment failed!"
        }
    }
}
