pipeline {
    agent any

    environment {
        PROMETHEUS_IMAGE = 'prom/prometheus:v2.53.4'
    }

    stages {
        stage('Validate Config') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                            set -eu
                            if command -v promtool >/dev/null 2>&1; then
                                promtool check config prometheus/prometheus.yml
                            elif command -v docker >/dev/null 2>&1; then
                                docker info >/dev/null
                                docker run --rm \
                                    --volume "$WORKSPACE:/workspace:ro" \
                                    --workdir /workspace \
                                    --entrypoint promtool \
                                    "$PROMETHEUS_IMAGE" \
                                    check config prometheus/prometheus.yml
                            else
                                echo "Validation requires promtool or Docker with daemon access." >&2
                                exit 1
                            fi
                        '''
                    } else {
                        bat '''
                            @echo off
                            where promtool >nul 2>nul
                            if not errorlevel 1 (
                                promtool check config prometheus\\prometheus.yml
                                if errorlevel 1 exit /b 1
                            ) else (
                                where docker >nul 2>nul
                                if errorlevel 1 (
                                    echo Validation requires promtool or Docker with daemon access. 1>&2
                                    exit /b 1
                                )
                                docker info >nul
                                if errorlevel 1 exit /b 1
                                docker run --rm -v "%WORKSPACE%:/workspace:ro" -w /workspace --entrypoint promtool %PROMETHEUS_IMAGE% check config prometheus/prometheus.yml
                                if errorlevel 1 exit /b 1
                            )
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'docker compose up -d --no-deps prometheus'
                    } else {
                        bat 'docker compose up -d --no-deps prometheus'
                    }
                }
            }
        }
    }
}
