pipeline {
    agent any

    environment {
        FLASK_APP = "app/api.py"
        PYTHONPATH = "."
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
                // Inicia tu contenedor de Docker
                sh 'docker start wiremock || true'
                sleep 5
            }
        }

        stage('Run API and Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    
                    # 1. Liberar el puerto 5000 por si acaso quedó algo colgado
                    fuser -k 5000/tcp || true
                    
                    # 2. Ejecutar Flask en segundo plano
                    flask run --host=0.0.0.0 --port=5000 &
                    
                    # 3. Esperar a que la API responda
                    sleep 10
                    
                    # 4. Ejecutar los tests de integración
                    pytest test/rest
                '''
            }
        }
    }

    post {
        always {
            // Matar el proceso de Flask al terminar para liberar el puerto 5000
            sh 'fuser -k 5000/tcp || true'
            echo 'Limpieza de procesos finalizada.'
        }
    }
}