pipeline {
    agent any

    environment {
        FLASK_APP = "app/api.py"
        // Ruta verificada del motor
        JMETER_BIN = "/home/justin/apache-jmeter-5.6.3/bin/jmeter.sh"
        // Ruta verificada de tu archivo editado (5 hilos, 40 loops)
        JMX_FILE = "test/jmeter/flask.jmx"
    }

    stages {
        stage('Get Code') {
            steps {
                // Jenkins lo hace automático si viene de SCM
                checkout scm
            }
        }

        stage('Static') {
            steps {
                sh './venv/bin/flake8 app/ --format=default > flake8.log || true'
                // Umbrales: 8 amarillo, 10 rojo
                recordIssues tool: flake8(pattern: 'flake8.log'), 
                             unstableTotalAll: 8, failedTotalAll: 10
            }
        }

        stage('Security Test') {
            steps {
                sh './venv/bin/bandit -r app/ -f txt -o bandit.txt || true'
                // Umbrales: 2 amarillo, 4 rojo
                recordIssues tool: bandit(pattern: 'bandit.txt'), 
                             unstableTotalAll: 2, failedTotalAll: 4
            }
        }

        stage('Unit') {
            steps {
                sh '''
                    # Corremos los tests y generamos la data para Coverage de una vez
                    ./venv/bin/coverage run -m pytest test/unit --junitxml=unit-results.xml
                '''
                junit 'unit-results.xml'
            }
        }

        stage('Coverage') {
            steps {
                sh './venv/bin/coverage xml -o coverage.xml'
                // Umbrales página 9: Líneas (85-95) y Ramas (80-90)
                recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']], 
                    lineThresholds: [[threshold: 85, unstableThreshold: 95]],
                    branchThresholds: [[threshold: 80, unstableThreshold: 90]])
            }
        }

        stage('Rest') {
            steps {
                sh '''
                    fuser -k 5000/tcp || true
                    ./venv/bin/flask run --host=0.0.0.0 --port=5000 &
                    sleep 5
                    ./venv/bin/pytest test/rest --junitxml=rest-results.xml
                    kill $(lsof -t -i:5000) || true
                '''
                junit 'rest-results.xml'
            }
        }

        stage('Performance') {
            steps {
                sh '''
                    fuser -k 5000/tcp || true
                    ./venv/bin/flask run --host=0.0.0.0 --port=5000 &
                    sleep 5
                    $JMETER_BIN -n -t $JMX_FILE -l results.jtl
                    kill $(lsof -t -i:5000) || true
                '''
                perfReport 'results.jtl'
            }
        }
    }
}