pipeline {
    agent none // No usamos un agente global, lo definimos por etapa

    environment {
        FLASK_APP = "app/api.py"
        JMETER_BIN = "/opt/jmeter/bin/jmeter.sh"
        JMX_FILE = "test/jmeter/flask.jmx"
    }

    stages {
        stage('Get Code & Setup') {
            agent { label 'agente-build' }
            steps {
                // Comandos de diagnóstico obligatorios
                sh 'whoami; hostname; echo "Workspace: ${WORKSPACE}"'
                
                checkout scm
                sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install --upgrade pip
                    ./venv/bin/pip install flask pytest coverage flake8 bandit
                '''
                // Guardamos el código y el venv para pasárselo al otro agente
                stash name: 'proyecto-completo', includes: '**/*', useDefaultExcludes: false
            }
        }

        stage('Pruebas en Paralelo Distribuidas') {
            parallel {
                stage('Unit Tests') {
                    agent { label 'agente-test' }
                    steps {
                        unstash 'proyecto-completo'
                        sh 'whoami; hostname; echo "Workspace: ${WORKSPACE}"'
                        sh '''
                            PYTHONPATH=.
                            ./venv/bin/coverage run --branch --source=app \
                                --omit=app/__init__.py,app/api.py \
                                -m pytest test/unit --junitxml=result_unit.xml
                            ./venv/bin/coverage xml -o coverage.xml
                        '''
                        junit 'result_unit.xml'
                        // Guardamos el reporte de cobertura para la etapa final
                        stash name: 'reporte-cobertura', includes: 'coverage.xml'
                    }
                }

                stage('Rest & Performance') {
                    agent { label 'agente-test' }
                    steps {
                        unstash 'proyecto-completo'
                        sh 'whoami; hostname; echo "Workspace: ${WORKSPACE}"'
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
                        unstash 'proyecto-completo'
                        sh 'whoami; hostname; echo "Workspace: ${WORKSPACE}"'
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh '''
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

        stage('Resultados Finales') {
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
            // Limpieza obligatoria del espacio de trabajo en ambos nodos
            node('agente-build') { cleanWs() }
            node('agente-test') { 
                sh 'docker stop wiremock || true'
                cleanWs() 
            }
        }
    }
}