pipeline {
    agent any

    environment {
        FLASK_APP = "app/api.py"
    }

    stages {
        stage('Preparación en Paralelo') {
            parallel {
                stage('Instalar Dependencias') {
                    steps {
                        sh '''
                            python3 -m venv venv
                            ./venv/bin/pip install --upgrade pip
                            ./venv/bin/pip install -r requirements.txt
                        '''
                    }
                }
                stage('Levantar Wiremock') {
                    steps {
                        sh 'docker start wiremock || true'
                        sleep 3
                    }
                }
            }
        }

        stage('Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    fuser -k 5000/tcp || true
                    flask run --host=0.0.0.0 --port=5000 &
                    pytest test/rest --junitxml=results.xml
                '''
            }
        }

        stage('Resultados'){
            steps {
                junit 'result*.xml'
            }
         }
    }
}