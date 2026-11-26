
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Construyendo imagen Docker'
                bat 'docker compose build'
            }
        }

        stage('Tests') {
            steps {
                echo 'Validando sintaxis Python'
                bat 'py -m compileall -q app.py'
            }
        }

        stage('Dependency Audit') {
            steps {
                echo 'Revisando dependencias'
                bat 'py -m pip install pip-audit'
                bat 'pip-audit -r requirements.txt'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Desplegando SecureDev'
                bat 'docker compose up -d'
            }
        }

        stage('OWASP ZAP') {
            steps {
                echo 'Ejecutando análisis ZAP Baseline'
                bat 'docker run --rm --add-host=host.docker.internal:host-gateway ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t http://host.docker.internal:8000 -I'
            }
        }
    }

    post {
        always {
            echo 'Pipeline SecureDev finalizado'
        }
    }
}
