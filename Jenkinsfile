pipeline {
    agent none

    environment {
        FLASK_APP = "app/api.py"
        JMETER_BIN = "/opt/jmeter/bin/jmeter.sh"
        JMX_FILE  = "test/jmeter/flask.jmx"
    }

    stages {

        stage('Get Code & Stash') {
            agent { label 'agente-build' }
            steps {
                sh 'whoami; hostname; echo "WORKSPACE=$WORKSPACE"'
                checkout scm
                stash name: 'codigo-fuente', includes: '**/*', excludes: 'venv/**, .pytest_cache/**'
            }
        }

        stage('Dependencies') {
            agent { label 'agente-build' }
            steps {
                sh 'whoami; hostname; echo "WORKSPACE=$WORKSPACE"'
                unstash 'codigo-fuente'
                sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install --upgrade pip
                    ./venv/bin/pip install -r requirements.txt
                '''
                stash name: 'venv', includes: 'venv/**'
            }
        }

        stage('Pruebas en Paralelo') {
            parallel {

                stage('Unit Tests') {
                    agent { label 'agente-test' }
                    steps {
                        sh 'whoami; hostname; echo "WORKSPACE=$WORKSPACE"'
                        unstash 'codigo-fuente'
                        unstash 'venv'
                        sh '''
                            PYTHONPATH=.
                            ./venv/bin/coverage run --branch --source=app \
                              --omit=app/__init__.py,app/api.py \
                              -m pytest test/unit --junitxml=result_unit.xml
                            ./venv/bin/coverage xml -o coverage.xml
                        '''
                        junit 'result_unit.xml'
                        stash name: 'reporte-cobertura', includes: 'coverage.xml'
                    }
                }

                stage('Rest & Performance') {
                    agent { label 'agente-test' }
                    steps {
                        sh 'whoami; hostname; echo "WORKSPACE=$WORKSPACE"'
                        unstash 'codigo-fuente'
                        unstash 'venv'
                        sh 'docker start wiremock || true'
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh '''
                                fuser -k 5000/tcp || true
                                ./venv/bin/flask run --host=0.0.0.0 --port=5000 &
                                sleep 5
                                PYTHONPATH=.
                                ./venv/bin/pytest test/rest --junitxml=result_rest.xml
                                $JMETER_BIN -n -t $JMX_FILE -l flask.jtl -f
                                fuser -k 5000/tcp || true
                            '''
                            junit 'result_rest.xml'
                            perfReport sourceDataFiles: 'flask.jtl'
                        }
                    }
                }

                stage('Static & Security') {
                    agent { label 'agente-test' }
                    steps {
                        sh 'whoami; hostname; echo "WORKSPACE=$WORKSPACE"'
                        unstash 'codigo-fuente'
                        unstash 'venv'
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh '''
                                ./venv/bin/flake8 app --exit-zero --format=pylint > flake8.out
                                ./venv/bin/bandit --exit-zero -r app -f custom -o bandit.out
                            '''
                            recordIssues(
                                tools: [flake8(pattern: 'flake8.out')],
                                qualityGates: [
                                    [criticality: 'NOTE',  integerThreshold: 8,  threshold: 8.0,  type: 'TOTAL'],
                                    [criticality: 'ERROR', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']
                                ]
                            )
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
            }
        }

        stage('Final Analysis') {
            agent { label 'agente-build' }
            steps {
                sh 'whoami; hostname; echo "WORKSPACE=$WORKSPACE"'
                unstash 'reporte-cobertura'
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

    post {
        always {
            node('agente-test') {
                sh 'docker stop wiremock || true'
                cleanWs()
            }
            node('agente-build') {
                cleanWs()
            }
        }
    }
}
