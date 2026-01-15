pipeline {
    agent any

    environment {
        FLASK_APP = "app/api.py"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Environment') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Wiremock') {
            steps {
                sh 'docker start wiremock || true'
                sleep 5
            }
        }

        stage('Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest test/rest
                '''
            }
        }
    }
}