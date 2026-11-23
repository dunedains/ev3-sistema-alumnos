// Jenkinsfile - Pipeline de "Sistema Alumnos" (SecureDev · Evaluación Parcial 3 · OCY1102)
//
// Build -> Test -> SAST (Bandit) -> SCA (pip-audit) -> Deploy Staging -> DAST (OWASP ZAP) -> [gate] -> Deploy Producción.
// Cualquier hallazgo Medium/High deja el build UNSTABLE y el gate impide desplegar en producción.
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 40, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    // Revisa GitHub cada ~3 minutos: cada push (tuyo o un PR de Dependabot fusionado) dispara un build.
    triggers {
        pollSCM('H/3 * * * *')
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    // Versión = 1.0.<nº de build>+<commit corto>: une cada despliegue con su commit (trazabilidad)
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}+${sha}"
                }
                echo "Versión de este build: ${env.APP_VERSION}"
                sh 'git log -1 --format="Commit %h de %an: %s"'
                // Cada build parte limpio: sin reportes viejos y con un entorno virtual nuevo
                sh 'rm -rf .venv reports && mkdir -p reports'
                sh 'python3 -m venv .venv'
                sh '.venv/bin/pip install --quiet -r requirements.txt -r requirements-dev.txt'
                sh '.venv/bin/python -m compileall -q app.py'
            }
        }

        stage('Test') {
            steps {
                sh '.venv/bin/python -m pytest --junitxml=reports/junit.xml'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                }
            }
        }

        // [SEGURIDAD] Revisión continua del código (SAST) en cada build
        stage('Security Review (SAST)') {
            steps {
                sh 'bandit -r app.py -f json -o reports/bandit.json || true'
                sh 'bandit -r app.py -f txt -o reports/bandit.txt || true'
                script {
                    // Hallazgos Medium o High marcan el build como UNSTABLE y bloquean producción
                    def rc = sh(returnStatus: true, script: 'bandit -q -r app.py -ll')
                    if (rc != 0) { unstable('Bandit: hay hallazgos de severidad Medium/High en app.py') }
                }
            }
        }

        // [SEGURIDAD] Dependencias con vulnerabilidades conocidas (SCA)
        stage('Dependency Management') {
            steps {
                sh 'pip-audit -r requirements.txt -f json -o reports/pip-audit.json || true'
                script {
                    def rc = sh(returnStatus: true, script: 'pip-audit -r requirements.txt --desc on > reports/pip-audit.txt 2>&1')
                    sh 'cat reports/pip-audit.txt'
                    if (rc != 0) { unstable('pip-audit: hay dependencias con vulnerabilidades conocidas (CVE/GHSA)') }
                }
            }
        }

        // [ZAP] Despliegue en staging para el análisis dinámico
        stage('Deploy Staging') {
            steps {
                sh 'bash scripts/deploy.sh staging 5001'
            }
        }

        // [ZAP] Pruebas automatizadas de seguridad (DAST) con OWASP ZAP sobre staging
        stage('DAST - OWASP ZAP') {
            steps {
                script {
                    def rc = sh(returnStatus: true, script: 'python3 scripts/zap_scan.py http://jenkins:5001')
                    if (rc == 2) { unstable('OWASP ZAP: hay alertas de riesgo High/Medium') }
                    else if (rc != 0) { error('El escaneo de OWASP ZAP falló') }
                }
            }
        }

        stage('Deploy Producción') {
            // [GATE] Solo se despliega a producción si TODAS las revisiones de seguridad pasaron (build SUCCESS)
            when {
                expression { currentBuild.currentResult == 'SUCCESS' }
            }
            steps {
                sh 'bash scripts/deploy.sh produccion 5000'
                script { env.PRODUCCION = env.APP_VERSION }
            }
        }
    }

    post {
        always {
            // Registro de trazabilidad: build <-> versión <-> commit <-> pruebas <-> hallazgos <-> despliegue
            withEnv(["RESULTADO=${currentBuild.currentResult}"]) {
                sh 'python3 scripts/trazabilidad.py'
            }
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}
