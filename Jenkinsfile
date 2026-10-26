pipeline {
    agent any

    environment {
        // Docker Hub configuration - Replace with your Docker Hub username/org
        DOCKERHUB_USERNAME       = 'sumithksam'
        DOCKER_IMAGE_NAME        = "${DOCKERHUB_USERNAME}/stridex-ecommerce"
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-creds' // ID configured in Jenkins Credentials Store

        // Build identifiers
        IMAGE_TAG                = "${BUILD_NUMBER}"
        REPORTS_DIR              = "${WORKSPACE}/reports"
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        timestamps()
        ansiColor('xterm')
    }

    stages {
        stage('Initialize & Clean') {
            steps {
                script {
                    echo "Preparing environment and reports directory..."
                    sh '''
                        rm -rf "${REPORTS_DIR}"
                        mkdir -p "${REPORTS_DIR}"
                        chmod 777 "${REPORTS_DIR}"
                    '''
                }
            }
        }

        stage('Dependency Installation') {
            steps {
                echo "Installing project dependencies..."
                sh '''
                    npm ci --prefer-offline || npm install
                '''
            }
        }

        stage('SCA: npm audit') {
            steps {
                echo "Running npm audit (Software Composition Analysis)..."
                sh '''
                    # Run npm audit without failing the pipeline (|| true)
                    npm audit --json > "${REPORTS_DIR}/npm-audit.json" || true
                    npm audit > "${REPORTS_DIR}/npm-audit.txt" || true
                    
                    # Generate an HTML report for friendly viewing
                    npx --yes npm-audit-html --output "${REPORTS_DIR}/npm-audit.html" || true
                    
                    echo "npm audit completed. Artifacts stored in ${REPORTS_DIR}/"
                '''
            }
        }

        stage('SAST: Semgrep Scan') {
            steps {
                echo "Running Semgrep SAST scan..."
                sh '''
                    # Execute Semgrep using official container; does not fail pipeline if findings exist
                    docker run --rm \
                        -v "${WORKSPACE}:/src" \
                        semgrep/semgrep semgrep scan \
                        --config auto \
                        --json -o /src/reports/semgrep.json \
                        --sarif -o /src/reports/semgrep.sarif \
                        --text -o /src/reports/semgrep.txt || true

                    echo "Semgrep scan completed. Artifacts stored in ${REPORTS_DIR}/"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} and ${DOCKER_IMAGE_NAME}:latest..."
                sh '''
                    docker build -t ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} -t ${DOCKER_IMAGE_NAME}:latest .
                '''
            }
        }

        stage('DAST: OWASP ZAP Scan') {
            steps {
                script {
                    echo "Starting application container and executing OWASP ZAP Baseline Scan..."
                    sh '''
                        # Create dedicated bridge network
                        docker network create zap-net || true

                        # Run the newly built container in the network
                        docker run -d --name zap-app-target \
                            --network zap-net \
                            -e PORT=3000 \
                            ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}

                        # Wait for the application to be ready
                        echo "Waiting for Next.js app to be reachable on port 3000..."
                        for i in $(seq 1 30); do
                            if docker run --rm --network zap-net curlimages/curl:latest -s http://zap-app-target:3000 > /dev/null 2>&1; then
                                echo "Application is healthy and ready for testing!"
                                break
                            fi
                            echo "Waiting for app to start... ($i/30)"
                            sleep 2
                        done

                        # Ensure write permissions for ZAP container (uid 1000)
                        chmod -R 777 "${REPORTS_DIR}"

                        # Run OWASP ZAP Baseline Scan
                        # -I tells ZAP not to return failure code on warnings
                        # || true ensures vulnerability findings never break the pipeline
                        docker run --rm \
                            --network zap-net \
                            -v "${REPORTS_DIR}:/zap/wrk/:rw" \
                            -t ghcr.io/zaproxy/zaproxy:stable \
                            zap-baseline.py \
                            -t http://zap-app-target:3000 \
                            -r zap-report.html \
                            -J zap-report.json \
                            -w zap-report.md \
                            -I || true

                        echo "OWASP ZAP scan completed. Reports stored in ${REPORTS_DIR}/"
                    '''
                }
            }
            post {
                always {
                    sh '''
                        echo "Cleaning up OWASP ZAP target container and network..."
                        docker rm -f zap-app-target || true
                        docker network rm zap-net || true
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo "Logging into Docker Hub and pushing images..."
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKERHUB_CREDENTIALS_ID,
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASS'
                )]) {
                    sh '''
                        echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker push ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Archiving scan reports..."
                archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true

                // If HTML Publisher plugin is installed, publish interactive report
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports',
                        reportFiles: 'zap-report.html,npm-audit.html',
                        reportName: 'Security Scan Reports'
                    ])
                }
            }
        }
        success {
            echo "CI Pipeline finished successfully! Latest image is ready on Docker Hub for CD."
        }
        failure {
            echo "Pipeline failed during build, packaging, or credential operations."
        }
    }
}
