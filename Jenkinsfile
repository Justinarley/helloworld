pipeline {
    agent none 

    environment {
        FLASK_APP = "app/api.py"
        JMETER_BIN = "/opt/jmeter/bin/jmeter.sh"
        JMX_FILE = "test/jmeter/flask.jmx"
    }

    stages {
        stage('Get Code & Stash') {
            agent { label 'agente-build' }
            steps {
                sh 'whoami; hostname; echo "Workspace: ${WORKSPACE}"'
                checkout scm
                // Stash solo del código fuente, excluyendo carpetas pesadas o problemáticas
                stash name: 'codigo-fuente', includes: '**/*', excludes: 'venv/**, .pytest_cache/**'
            }
        }

        stage('Pruebas en Paralelo') {
            parallel {
                stage('Unit Tests') {
                    agent { label 'agente-test' }
                    steps {
                        unstash 'codigo-fuente'
                        sh 'whoami; hostname; echo "Workspace: ${WORKSPACE}"'
                        sh '''
                            python3 -m venv venv
                            ./venv/bin/pip install flask pytest coverage
                            PYTHONPATH=. ./venv/bin/coverage run -m pytest test/unit --junitxml=result_unit.xml
                            ./venv/bin/coverage xml -o coverage.xml
                        '''
                        junit 'result_unit.xml'
                        stash name: 'reporte-cobertura', includes: 'coverage.xml'
                    }
                }

                stage('Rest & Performance') {
                    agent { label 'agente-test' }
                    steps {
                        unstash 'codigo-fuente'
                        sh 'whoami; hostname; echo "Workspace: ${WORKSPACE}"'
                        sh 'docker start wiremock || true'
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh '''
                                python3 -m venv venv
                                ./venv/bin/pip install flask pytest
                                fuser -k 5000/tcp || true
                                ./venv/bin/flask run --host=0.0.0.0 --port=5000 &
                                sleep 5
                                PYTHONPATH=. ./venv/bin/pytest test/rest --junitxml=result_rest.xml
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
                        unstash 'codigo-fuente'
                        sh 'whoami; hostname; echo "Workspace: ${WORKSPACE}"'
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh '''
                                python3 -m venv venv
                                ./venv/bin/pip install flake8 bandit
                                ./venv/bin/flake8 app --exit-zero --format=pylint > flake8.out
                                ./venv/bin/bandit --exit-zero -r app -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                            '''
                            recordIssues(tools: [flake8(pattern: 'flake8.out')])
                            recordIssues(tools: [pyLint(pattern: 'bandit.out')])
                        }
                    }
                }
            }
        }

        stage('Final Analysis') {
            agent { label 'agente-build' }
            steps {
                unstash 'reporte-cobertura'
                sh 'whoami; hostname; echo "Workspace: ${WORKSPACE}"'
                recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']])
            }
        }
    }

    post {
        always {
            node('agente-build') { cleanWs() }
            node('agente-test') { 
                sh 'docker stop wiremock || true'
                cleanWs() 
            }
        }
    }
}