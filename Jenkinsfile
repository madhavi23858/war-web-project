pipeline {
    agent any

    tools {
        maven "maven"
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "43.205.233.59:8081"
        NEXUS_REPOSITORY = "demo-release"
        NEXUS_CREDENTIAL_ID = "nexus_credentials"
        ARTVERSION = "1.0.0"
        TOMCAT_URL = "http://13.126.128.217:8080"
        TOMCAT_CREDENTIAL_ID = "tomcat_credentials"
        TOMCAT_USERNAME = "tomcat-user"
        TOMCAT_PASSWORD = "secure-password" // Update this to the correct password
    }

    stages {
        stage('Build WAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war'
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    if (!fileExists(warFile)) {
                        error("WAR file not found at ${warFile}")
                    }
                    nexusArtifactUploader(
                        nexusVersion: "${NEXUS_VERSION}",
                        protocol: "${NEXUS_PROTOCOL}",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "com.example.warwebproject",
                        version: "${ARTVERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: "wwp", classifier: '', file: warFile, type: "war"],
                            [artifactId: "wwp", classifier: '', file: "pom.xml", type: "pom"]
                        ]
                    )
                }
            }
        }

        stage('Deploy WAR') {
            steps {
                script {
                    // Dynamically find the latest WAR file in the target directory
                    def warFilePath = sh(script: "find ${workspace}/target -name '*.war' -type f -print0 | xargs -0 ls -t | head -n 1", returnStdout: true).trim()

                    if (warFilePath) {
                        echo "WAR file located at: ${warFilePath}"

                        // Deploy the WAR file to the remote server
                        sh """
                            # Copy the WAR file to a temporary directory on the remote server
                            scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/jenkins_key ${warFilePath} ubuntu@13.126.128.217:/tmp/wwp.war

                            # Move the WAR file to the correct directory and restart Tomcat
                            ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/jenkins_key ubuntu@13.126.128.217 <<EOF
                                set -e
                                echo "Moving WAR file to Tomcat directory..."
                                sudo mv /tmp/wwp.war /opt/tomcat/webapps/wwp.war
                                echo "WAR file deployed successfully to /opt/tomcat/webapps/wwp.war"
                                
                                echo "Restarting Tomcat..."
                                sudo systemctl restart tomcat
                                echo "Tomcat restarted."
                            EOF
                        """
                    } else {
                        error "No WAR file found to deploy."
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed. Check logs for details.'
        }
    }
}
