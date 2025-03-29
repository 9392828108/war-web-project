pipeline {
    agent any

    environment {
        ART_VERSION = "1.0.0"
        NEXUS_URL = "3.109.203.221:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        TOMCAT_SERVER = "43.204.112.166"
        TOMCAT_USER = "ubuntu"
        TOMCAT_PASSWORD = "your_password_here" // Replace with actual password
    }

    tools {
        maven "maven"
    }

    stages {
        // Stage 1: Build WAR
        stage('Build WAR') {
            steps {
                echo '🔨 Building WAR file...'
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war'
            }
        }

        // Stage 2: Publish to Nexus
        stage('Publish to Nexus') {
            steps {
                echo '📦 Publishing WAR to Nexus...'
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()

                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "com.example",
                        version: "${ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: "simple-war", classifier: '', file: warFile, type: "war"]
                        ]
                    )
                }
            }
        }

        // Stage 3: Verify SSH Connection
        stage('Verify SSH Connection') {
            steps {
                script {
                    echo '🔗 Verifying connection to Tomcat server...'

                    // Check if sshpass is installed
                    def sshpassInstalled = sh(script: "which sshpass || true", returnStatus: true)
                    if (sshpassInstalled != 0) {
                        error "❌ sshpass is not installed on Jenkins. Please install it to proceed."
                    }

                    // Test SSH connection using password
                    def sshTest = sh(script: "sshpass -p '${TOMCAT_PASSWORD}' ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_SERVER} 'echo connection successful'", returnStatus: true)

                    if (sshTest != 0) {
                        error "❌ Unable to establish SSH connection to Tomcat server at ${TOMCAT_SERVER}."
                    } else {
                        echo "✅ SSH connection established successfully!"
                    }
                }
            }
        }

        // Stage 4: Deploy to Tomcat
        stage('Deploy to Tomcat') {
            steps {
                echo '🚀 Deploying WAR to Tomcat...'
                sh '''
                    sshpass -p '${TOMCAT_PASSWORD}' scp -o StrictHostKeyChecking=no target/*.war ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                    sshpass -p '${TOMCAT_PASSWORD}' ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_SERVER} << 'EOF'
                        sudo mv /tmp/*.war /opt/tomcat/webapps/
                        sudo systemctl restart tomcat
                    EOF
                '''
            }
        }

        // Stage 5: Verify Deployment
        stage('Verify Deployment') {
            steps {
                echo '✅ Verifying deployment...'
                script {
                    def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://${TOMCAT_SERVER}:8080/simple-war", returnStdout: true).trim()
                    if (status == "200") {
                        echo "🎉 Application deployed successfully and is accessible!"
                    } else {
                        error "❌ Deployment verification failed with HTTP code ${status}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs for errors.'
        }
    }
}
