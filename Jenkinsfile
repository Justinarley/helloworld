pipeline {
    agent any

    environment {
        FLASK_APP = "app/api.py"
        JMETER_BIN = "/opt/jmeter/bin/jmeter.sh"
        JMX_FILE = "test/jmeter/flask.jmx"
    }

    stages {
        stage('Get Code') {
            steps {
                checkout scm
            }
        }

        stage('Preparación en Paralelo') {
            parallel {
                stage('Instalar Dependencias') {
                    steps {
                        sh '''
                            python3 -m venv venv
                            ./venv/bin/pip install --upgrade pip
                            ./venv/bin/pip install flask pytest coverage flake8 bandit
                            if [ -f requirements.txt ]; then ./venv/bin/pip install -r requirements.txt; fi
                        '''
                    }
                }
                stage('Levantar Wiremock') {
                    steps {
                        sh 'docker start wiremock || true'
                    }
                }
            }
        }

        stage('Unit') {
            steps {
                sh '''
                    PYTHONPATH=.
                    ./venv/bin/coverage run --branch --source=app \
                        --omit=app/__init__.py,app/api.py \
                        -m pytest test/unit --junitxml=result_unit.xml
                    ./venv/bin/coverage xml -o coverage.xml
                '''
                junit 'result_unit.xml'
            }
        }

        stage('Rest') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh '''
                        fuser -k 5000/tcp || true
                        ./venv/bin/flask run --host=0.0.0.0 --port=5000 &
                        sleep 5
                        PYTHONPATH=.
                        ./venv/bin/pytest test/rest --junitxml=result_rest.xml
                        kill $(lsof -t -i:5000) || true
                    '''
                    junit 'result_rest.xml'
                }
            }
        }

        stage('Static') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh './venv/bin/flake8 app --exit-zero --format=pylint > flake8.out'
                    recordIssues(
                        tools: [flake8(pattern: 'flake8.out')],
                        qualityGates: [
                            [threshold: 8, type: 'TOTAL', criticality: 'UNSTABLE'],
                            [threshold: 10, type: 'TOTAL', criticality: 'FAILURE']
                        ]
                    )
                }
            }
        }

        stage('Security Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh './venv/bin/bandit -r app --exit-zero -f json -o bandit.json'
                    recordIssues(
                        tools: [pyLint(pattern: 'bandit.json')],
                        qualityGates: [
                            [threshold: 2, type: 'TOTAL', criticality: 'UNSTABLE'],
                            [threshold: 4, type: 'TOTAL', criticality: 'FAILURE']
                        ]
                    )
                }
            }
        }

        stage('Coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    recordCoverage(
                        tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
                        qualityGates: [
                            [threshold: 95.0, metric: 'LINE', baseline: 'PROJECT', criticality: 'UNSTABLE'],
                            [threshold: 85.0, metric: 'LINE', baseline: 'PROJECT', criticality: 'FAILURE'],
                            [threshold: 90.0, metric: 'BRANCH', baseline: 'PROJECT', criticality: 'UNSTABLE'],
                            [threshold: 80.0, metric: 'BRANCH', baseline: 'PROJECT', criticality: 'FAILURE']
                        ]
                    )
                }
            }
        }

        stage('Performance') {
            steps {
                sh '''
                    fuser -k 5000/tcp || true
                    ./venv/bin/flask run --host=0.0.0.0 --port=5000 &
                    sleep 5
                    $JMETER_BIN -n -t $JMX_FILE -l flask.jtl -f
                    kill $(lsof -t -i:5000) || true
                '''
                perfReport sourceDataFiles: 'flask.jtl'
            }
        }
    }

    post {
        always {
            sh 'docker stop wiremock || true'
            cleanWs()
        }
    }
}