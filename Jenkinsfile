pipeline {
    agent any

    tools {
        maven 'Maven-3.9'
        jdk   'JDK-17'
    }

    environment {
        // ── Update these 3 values only ──
        NEXUS_IP   = '44.204.255.105'
        DEPLOY_IP  = '3.91.3.65'
        NEXUS_PASS = 'YOUR_NEXUS_JENKINS_PASSWORD'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: '',
                    credentialsId: 'git-credentials'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=demo-app'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Push to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUS_IP}:8081",
                    groupId: 'com.demo',
                    version: '1.0.0',
                    repository: 'maven-releases',
                    credentialsId: 'nexus-credentials',
                    artifacts: [[
                        artifactId: 'demo-app',
                        classifier: '',
                        file: 'target/demo-app-1.0.0.jar',
                        type: 'jar'
                    ]]
                )
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${DEPLOY_IP} '
                            mkdir -p /opt/demo-app

                            curl -u jenkins:${NEXUS_PASS} \
                                 -o /opt/demo-app/app.jar \
                                 http://${NEXUS_IP}:8081/repository/maven-releases/com/demo/demo-app/1.0.0/demo-app-1.0.0.jar

                            pkill -f "app.jar" || true
                            sleep 2

                            nohup java -jar /opt/demo-app/app.jar \
                                --server.port=9001 \
                                > /opt/demo-app/app.log 2>&1 &

                            echo "App started!"
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployed! Visit: http://${DEPLOY_IP}:9001"
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
        always {
            cleanWs()
        }
    }
}
