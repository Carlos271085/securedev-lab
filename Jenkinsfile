
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
                sh '''
                    docker build -t securedev-ci:${BUILD_NUMBER} .
                '''
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
                sh '''
                    docker run --rm \
                    --entrypoint pip \
                    securedev-ci:${BUILD_NUMBER} \
                    install pip-audit

                    docker run --rm \
                    --entrypoint pip-audit \
                    securedev-ci:${BUILD_NUMBER} \
                    -r requirements.txt
                '''
            }
        }

        stage('OWASP ZAP') {
            steps {
                sh '''
                    curl -fsS "${ZAP_URL}/JSON/spider/action/scan/?url=${TARGET_URL}"

                    sleep 10

                    curl -fsS "${ZAP_URL}/JSON/alert/view/alerts/?baseurl=${TARGET_URL}" \
                    -o zap-alerts.json

                    echo "Reporte ZAP generado"
                '''
            }
        }

        stage('Despliegue') {
            steps {
                echo 'Verificando aplicacion desplegada'
                sh '''
                    docker compose ps
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
