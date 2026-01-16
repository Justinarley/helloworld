pipeline {
    agent any

    environment {
        FLASK_APP = "app/api.py"
        // Tu ruta de JMeter
        JMETER_BIN = "/home/justin/apache-jmeter-5.6.3/bin/jmeter.sh"
        // Tu archivo JMX
        JMX_FILE = "test/jmeter/flask.jmx"
    }

    stages {
        stage('Get Code') {
            steps {
                checkout scm
            }
        }

        stage('Install deps') {
            steps {
                sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install --upgrade pip
                    ./venv/bin/pip install flask coverage pytest bandit flake8
                '''
            }
        }

        stage('Static') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh './venv/bin/flake8 --exit-zero --format=pylint app > flake8.out'
                    recordIssues(
                        tools: [flake8(pattern: 'flake8.out')],
                        qualityGates: [
                            [threshold: 8, type: 'TOTAL', unstable: true],
                            [threshold: 10, type: 'TOTAL', unstable: false]
                        ],
                        sourceCodeRetention: 'LAST_BUILD'
                    )
                }
            }
        }

        stage('Security Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh './venv/bin/bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'
                    recordIssues(
                        tools: [pyLint(pattern: 'bandit.out')],
                        qualityGates: [
                            [threshold: 2, type: 'TOTAL', unstable: true],
                            [threshold: 4, type: 'TOTAL', unstable: false]
                        ],
                        sourceCodeRetention: 'LAST_BUILD'
                    )
                }
            }
        }

        stage('Unit') {
            steps {
                sh '''
                    PYTHONPATH=.
                    ./venv/bin/coverage run --branch --source=app --omit=app/__init__.py,app/api.py \
                        -m pytest test/unit --junitxml=result_unit.xml
                    ./venv/bin/coverage xml -o coverage.xml
                '''
                junit 'result_unit.xml'
            }
        }

        stage('Rest') {
            steps {
                sh '''
                    fuser -k 5000/tcp || true
                    ./venv/bin/flask run --host=0.0.0.0 --port=5000 &
                    sleep 5
                    PYTHONPATH=$WORKSPACE
                    ./venv/bin/pytest --junitxml=result_rest.xml test/rest
                    kill $(lsof -t -i:5000) || true
                '''
                junit 'result_rest.xml'
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

        stage('Cobertura') {
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
    }

    post {
        always {
            cleanWs()
        }
    }
}