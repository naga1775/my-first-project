pipeline {
    agent any
    stages {
        stage('Clone Repo') {
            steps {
                git credentialsId: 'github-creds', url: 'https://github.com/naga1775/aws-project.git'
            }
        }
        stage('Build') {
            steps {
                echo 'Building application...'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying to EC2...'
            }
        }
    }
}
