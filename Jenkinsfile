pipeline {
    agent any

    environment {
        FLASK_APP = "app/api.py"
    }

    stages {
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
            }
        }

        stage('Run API and Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    fuser -k 5000/tcp || true
                    flask run --host=0.0.0.0 --port=5000 &
                    pytest test/rest --junitxml=results.xml
                '''
            }
        }

        stage('Results'){
            steps {
                junit 'result*.xml'
            }
         }
    }
}