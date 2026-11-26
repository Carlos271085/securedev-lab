
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'securedev-ci'
        ZAP_URL = 'http://zap-securedev:8080'
        TARGET_URL = 'http://securedev-estudiante-app-1:8000'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Codigo descargado desde GitHub'
            }
        }

        stage('Validar estructura') {
            steps {
                sh '''
                    test -f app.py
                    test -f requirements.txt
                    test -f Dockerfile
                    test -f compose.yaml
                    echo "Estructura validada"
                '''
            }
        }

        stage('Construccion Docker') {
            steps {
                echo 'Construyendo imagen Docker de SecureDev'
                sh 'docker version'
                sh 'docker build -t securedev-ci:${BUILD_NUMBER} .'
                sh 'docker image inspect securedev-ci:${BUILD_NUMBER}'
            }
        }


        stage('Pruebas de sintaxis') {
            steps {
                sh '''
                    docker run --rm \
                    --entrypoint python \
                    securedev-ci:${BUILD_NUMBER} \
                    -m compileall -q .
                '''
            }
        }

        stage('Auditoria de dependencias') {
            steps {
               echo 'Ejecutando auditoria de dependencias con pip-audit'

                sh """
                    docker run --rm \
                    --entrypoint sh \
                    securedev-ci:${BUILD_NUMBER} \
                    -c "pip install --no-cache-dir pip-audit && pip-audit -r requirements.txt"
                """
            }
        }

        stage('OWASP ZAP') {
            steps {
                echo 'Ejecutando analisis de seguridad con OWASP ZAP'

                sh '''
                    set -e

                    echo "Iniciando Spider..."
                    RESPONSE=$(curl -fsS "${ZAP_URL}/JSON/spider/action/scan/?url=${TARGET_URL}")
                    SCAN_ID=$(echo "$RESPONSE" | sed -n 's/.*"scan":"\\([0-9]*\\)".*/\\1/p')

                    if [ -z "$SCAN_ID" ]; then
                        echo "No se pudo iniciar el Spider"
                        exit 1
                    fi

                    STATUS=0

                    for i in $(seq 1 30); do
                        RESPONSE=$(curl -fsS "${ZAP_URL}/JSON/spider/view/status/?scanId=${SCAN_ID}")
                        STATUS=$(echo "$RESPONSE" | sed -n 's/.*"status":"\\([0-9]*\\)".*/\\1/p')

                        echo "Progreso Spider: ${STATUS}%"

                        [ "$STATUS" = "100" ] && break
                        sleep 2
                    done

                     if [ "$STATUS" != "100" ]; then
                        echo "El Spider no finalizo a tiempo"
                    exit 1
                    fi

                    echo "Esperando analisis pasivo..."

                    PENDING=1

                        for i in $(seq 1 30); do
                        RESPONSE=$(curl -fsS "${ZAP_URL}/JSON/pscan/view/recordsToScan/")
                        PENDING=$(echo "$RESPONSE" | sed -n 's/.*"recordsToScan":"\\([0-9]*\\)".*/\\1/p')

                echo "Registros pendientes: ${PENDING}"

                [ "$PENDING" = "0" ] && break
                sleep 2
            done

            if [ "$PENDING" != "0" ]; then
                echo "El analisis pasivo no finalizo a tiempo"
                exit 1
            fi

            echo "Guardando resultados..."
            curl -fsS "${ZAP_URL}/JSON/alert/view/alerts/?baseurl=${TARGET_URL}" -o zap-alerts.json

            echo "Reporte ZAP generado correctamente"
        '''
            }
        }

        stage('Despliegue') {
            steps {
             echo 'Verificando aplicacion desplegada'

        sh '''
            set -e

            echo "Verificando contenedor SecureDev..."

            docker inspect -f '{{.State.Running}}' securedev-estudiante-app-1 | grep -q true

            echo "Verificando respuesta HTTP..."

            docker run --rm \
                --network securedev-net \
                --entrypoint python \
                securedev-ci:${BUILD_NUMBER} \
                -c "import urllib.request; r=urllib.request.urlopen('http://securedev-estudiante-app-1:8000/', timeout=10); print('HTTP:', r.status); assert r.status == 200"

            echo "Aplicacion desplegada y respondiendo correctamente"
        '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'zap-alerts.json',
                             allowEmptyArchive: true

            echo 'Pipeline SecureDev finalizado'
        }
    }
}
