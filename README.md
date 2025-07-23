pipeline {
    agent any

    environment {
        // Change to your actual servers and SSH credentials
        BLUE_SERVER = "ec2-user@<BLUE_EC2_IP>"
        GREEN_SERVER = "ec2-user@<GREEN_EC2_IP>"
        SSH_KEY = "your-ssh-credentials-id"
        APP_DIR = "/var/www/myapp"
        GIT_REPO = "https://github.com/your-user/your-app.git"
    }

    stages {
        stage('Clone Repo') {
            steps {
                git url: "${GIT_REPO}"
            }
        }

        stage('Deploy to Green') {
            steps {
                sshagent (credentials: ["${SSH_KEY}"]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ${GREEN_SERVER} '
                        sudo rm -rf ${APP_DIR} &&
                        git clone ${GIT_REPO} ${APP_DIR} &&
                        cd ${APP_DIR} &&
                        ./deploy.sh
                    '
                    """
                }
            }
        }

        stage('Health Check on Green') {
            steps {
                script {
                    def result = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" http://${GREEN_SERVER}:8080/health", returnStdout: true).trim()
                    if (result != "200") {
                        error "Green deployment failed health check"
                    }
                }
            }
        }

        stage('Switch Load Balancer to Green') {
            steps {
                echo "Deregistering Blue and Registering Green in Load Balancer..."
                // Call AWS CLI to update target group
                sh """
                aws elbv2 deregister-targets --target-group-arn <TG_ARN> --targets Id=<BLUE_EC2_ID>
                aws elbv2 register-targets --target-group-arn <TG_ARN> --targets Id=<GREEN_EC2_ID>
                """
            }
        }
    }

    post {
        success {
            echo "Green deployment successful!"
        }
        failure {
            echo "Deployment failed. Manual rollback may be needed."
        }
    }
}
