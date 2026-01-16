pipeline {
    agent any

    environment {
        FLASK_APP = "app/api.py"
        JMETER_BIN = "/home/justin/apache-jmeter-5.6.3/bin/jmeter.sh"
        JMX_FILE = "test/jmeter/flask.jmx"
    }

    stages {
        stage('Get Code') {
            steps {
                checkout scm
            }
        }

        stage('Static') {
            steps {
                sh './venv/bin/flake8 app/ --format=default > flake8.log || true'
                // Sintaxis corregida para umbrales de Flake8
                recordIssues tool: flake8(pattern: 'flake8.log'), 
                             unstableTotalAll: 8, 
                             failedTotalAll: 10
            }
        }

        stage('Security Test') {
            steps {
                sh './venv/bin/bandit -r app/ -f txt -o bandit.txt || true'
                // Sintaxis corregida para umbrales de Bandit
                recordIssues tool: bandit(pattern: 'bandit.txt'), 
                             unstableTotalAll: 2, 
                             failedTotalAll: 4
            }
        }

        stage('Unit') {
            steps {
                sh './venv/bin/coverage run -m pytest test/unit --junitxml=unit-results.xml'
                junit 'unit-results.xml'
            }
        }

        stage('Coverage') {
            steps {
                sh './venv/bin/coverage xml -o coverage.xml'
                // Sintaxis simplificada para Coverage
                recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
                    thresholds: [
                        [thresholdTarget: 'Line', unstableThreshold: 95.0, failureThreshold: 85.0],
                        [thresholdTarget: 'Branch', unstableThreshold: 90.0, failureThreshold: 80.0]
                    ]
                )
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