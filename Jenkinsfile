pipeline{
    agent any
    environment {
        PYTHONPATH = "${WORKSPACE}"
        FLASK_APP = "./app/api.py"
    }
    stages{
        stage('Get Code'){
            steps{
                deleteDir()
                echo "Fecha de ejecución..." 
                sh 'date'
                echo "Me voy a traer el codigo"
                checkout scm
                sh 'ls -la'
                echo WORKSPACE
            }
        }
        stage('Test'){  
            parallel {
                stage('JUnit'){
                    steps{
                          echo 'WORKSPACE->' + WORKSPACE
                          echo 'PYTHONPATH->' + PYTHONPATH
                          sh '''
                                coverage run \
                                    --branch \
                                    --data-file=.coverage \
                                    --source=app \
                                    --omit="app/__init__.py,app/api.py" \
                                    -m pytest ./test/unit
                            '''
                    }
                }
                stage('Rest'){
                    steps{
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE'){
                             sh '''
                                    echo 'Levantamos Flask...'
                                    flask run &
                                    echo 'Levantamos el Wiremock...'
                                    java -jar /Users/david/Wiremock/wiremock-standalone-3.13.0.jar --port 9090 --root-dir ./test/wiremock &
                                    echo 'Miramos que procesos estan levantados...'
                                    ps|grep java
                                    timeout=30
                                    while ! nc -z 127.0.0.1 9090; do
                                        sleep 1
                                        timeout=$((timeout - 1))
                                        if [ $timeout -le 0 ]; then
                                            echo "Timeout esperando a Wiremock"
                                            WIREMOCK_PID = $(ps | grep 'wiremock' | grep -v grep | awk '{print $1}')
                                            echo $WIREMOCK_PID
                                            kill $WIREMOCK_PID
                                            exit 1
                                        fi
                                    done
                                    echo 'Lanzamos las pruebas...'
                                    pytest ./test/rest
                            '''
                        }
                    }
                }
            }
        }
        stage('Static'){
             steps{
                 sh '''
                    flake8 --exit-zero --format=pylint app > flake8.out
                 '''
                 recordIssues qualityGates: [
                        [integerThreshold: 8, threshold: 8.0, type: 'TOTAL'], 
                        [criticality: 'FAILURE', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']
                     ], 
                     tools: [
                         flake8(linesLookAhead: 0, pattern: 'flake8.out')
                    ]
             }
         }
         stage('Security'){
             steps{
                 sh '''
                    bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                 '''
                 recordIssues(
                     tools: [pyLint(name: "Bandit", pattern: 'bandit.out')],
                     qualityGates: [
                            [threshold: 2, type: 'TOTAL', unstable: true],
                            [threshold: 4, type: 'TOTAL', unstable: false]
                     ]
                )
             }
         }     
         stage('Performance'){
             steps{
                 sh '''
                    echo "PATH ->" $PATH
                    if ! nc -z 127.0.0.1 5000 ; then
                        echo 'Levantamos Flask...'
                        flask run &
                    fi
                    echo 'Flask levantado...'
                    echo 'Lanzamos las pruebas de Rendimiento...'
                    jmeter -n -t ./test/jmeter/cp1b.jmx -f -l flask.jtl 
                 '''
                perfReport sourceDataFiles: 'flask.jtl'
             }
         }

        stage('Cobertura'){
            steps{
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE'){
                        sh 'coverage xml'
                        recordCoverage qualityGates: 
                            [
                                [integerThreshold: 95, metric: 'LINE', threshold: 95.0], 
                                [criticality: 'ERROR', integerThreshold: 85, metric: 'LINE', threshold: 85.0], 
                                [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0], 
                                [integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]
                            ], 
                            tools: [
                                    [parser: 'COBERTURA', pattern: 'coverage.xml']
                                ]
                    
                }
            }
        }
    }
}