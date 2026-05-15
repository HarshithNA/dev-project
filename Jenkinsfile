pipeline {
    agent any

    tools {
        maven 'Maven-3.9'
        jdk 'JDK17'
    }

    environment {
        NEXUS_IP = '13.49.138.137'
        DEPLOY_IP = '56.228.29.94'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/HarshithNA/dev-project.git',
                    credentialsId: 'git-credentials'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
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
                    nexusVersion:  'nexus3',
                    protocol:      'http',
                    nexusUrl:      "${NEXUS_IP}:8081",
                    groupId:       'com.demo',
                    version:       '1.0.0',
                    repository:    'maven-releases',
                    credentialsId: 'nexus-credentials',
                    artifacts: [
                        [
                            artifactId: 'demo-app',
                            classifier: '',
                            file:       'target/demo-app-1.0.0.jar',
                            type:       'jar'
                        ]
                    ]
                )
            }
        }

        stage('Deploy') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                        scp -o StrictHostKeyChecking=no target/demo-app-1.0.0.jar ubuntu@${DEPLOY_IP}:/home/ubuntu/

                        ssh -o StrictHostKeyChecking=no ubuntu@${DEPLOY_IP} "bash -s" << 'ENDSSH'
                            pkill -f demo-app-1.0.0.jar || true
                            sleep 2
                            nohup java -jar /home/ubuntu/demo-app-1.0.0.jar > /home/ubuntu/app.log 2>&1 &
                            echo "App started"
ENDSSH
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}

