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

        stage('Tests (Unit & Rest)') {
            parallel {
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
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh '''
                                fuser -k 5000/tcp || true
                                ./venv/bin/flask run --host=0.0.0.0 --port=5000 &
                                sleep 5
                                PYTHONPATH=.
                                ./venv/bin/pytest test/rest --junitxml=result_rest.xml
                                fuser -k 5000/tcp || true
                            '''
                            junit 'result_rest.xml'
                        }
                    }
                }
            }
        }

        stage('Static') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                       ./venv/bin/flake8 app --exit-zero --format=pylint > flake8.out
                    '''
                    recordIssues(
                        tools: [flake8(pattern: 'flake8.out')],
                        qualityGates: [
                            [criticality: 'NOTE',  integerThreshold: 8,  threshold: 8.0,  type: 'TOTAL'],
                            [criticality: 'ERROR', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']
                        ]
                    )
                }
            }
        }

        stage('Security Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        ./venv/bin/bandit -r app --exit-zero -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                        cat bandit.out
                    '''
                    recordIssues(
                        tools: [pyLint(pattern: 'bandit.out')],
                        qualityGates: [
                            [criticality: 'NOTE',    integerThreshold: 2, threshold: 2.0, type: 'TOTAL'],
                            [criticality: 'FAILURE', integerThreshold: 4, threshold: 4.0, type: 'TOTAL']
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
                    fuser -k 5000/tcp || true
                '''
                perfReport sourceDataFiles: 'flask.jtl'
            }
        }
        
        stage('Coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    recordCoverage(
                        tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
                        qualityGates: [
                            [criticality: 'ERROR', integerThreshold: 85, metric: 'LINE',   threshold: 85.0],
                            [criticality: 'NOTE',  integerThreshold: 95, metric: 'LINE',   threshold: 95.0],
                            [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0],
                            [criticality: 'NOTE',  integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]
                        ]
                    )
                }
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