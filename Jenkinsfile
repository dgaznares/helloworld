pipeline{
    agent any
    environment {
        PYTHONPATH = "${WORKSPACE}"
        FLASK_APP = "./app/api.py"
    }
    stages{
        stage('Get Code'){
            steps{
                cleanWs()
                echo "Fecha de ejecución..." 
                sh 'date'
                echo "Me voy a traer el codigo"
                git 'https://github.com/dgaznares/helloworld.git'
                sh 'ls -la'
                echo WORKSPACE
            }
        }
        stage('Build'){
            steps{
                echo("No vamos a hacer nada.")
            }
        }
        stage('Test'){  
                parallel {
                   stage('JUnit'){
                        steps{
                          echo 'WORKSPACE->' + WORKSPACE
                          echo 'PYTHONPATH->' + PYTHONPATH
                          sh 'pytest --junitxml=result-unit.xml ./test/unit'
                        }
                    }
                    stage('Prueba de servicios'){
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
                                    pytest --junitxml=result-rest.xml ./test/rest
                                '''
                            }
                        }
                    }
                }
         }
         stage('Results'){
             steps{
                 junit 'result*.xml'
             }
         }
    }
}