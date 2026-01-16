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

    stage('Preparación en Paralelo') {
            parallel {
                stage('Instalar Dependencias') {
                    steps {
                        sh '''
                            python3 -m venv venv
                            ./venv/bin/pip install --upgrade pip
                            if [ -f requirements.txt ]; then
                                ./venv/bin/pip install -r requirements.txt
                            else
                                ./venv/bin/pip install flask pytest coverage flake8 bandit
                            fi
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
                        kill $(lsof -t -i:5000) || true
                    '''
                    junit 'result_rest.xml'
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
                            [threshold: 8, type: 'TOTAL', unstable: true],
                            [threshold: 10, type: 'TOTAL', unstable: false]
                        ]
                    )
                }
            }
        }

        stage('Security Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        ./venv/bin/bandit -r app --exit-zero -f json -o bandit.json
                    '''
                    recordIssues(
                        tools: [pyLint(pattern: 'bandit.json')],
                        qualityGates: [
                            [threshold: 2, type: 'TOTAL', unstable: true],
                            [threshold: 4, type: 'TOTAL', unstable: false]
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
                            [threshold: 85, metric: 'LINE', unstable: true],
                            [threshold: 80, metric: 'BRANCH', unstable: true],
                            [threshold: 95, metric: 'LINE', unstable: false],
                            [threshold: 90, metric: 'BRANCH', unstable: false]
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
