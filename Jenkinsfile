pipeline {
    agent any

    environment {
        // Docker Hub configuration
        DOCKERHUB_USERNAME       = 'sumithksam'
        DOCKER_IMAGE_NAME        = "${DOCKERHUB_USERNAME}/stridex-ecommerce"
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-creds' 
        
        // Git configuration for pushing manifest updates
        GIT_CREDENTIALS_ID       = 'github-pat' 

        // Build identifiers (Using BUILD_NUMBER ensures immutable, traceable tags like 1, 2, 3...)
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
                sh 'npm ci --prefer-offline || npm install'
            }
        }

        stage('SCA: npm audit') {
            steps {
                echo "Running npm audit (Software Composition Analysis)..."
                sh '''
                    npm audit --json > "${REPORTS_DIR}/npm-audit.json" || true
                    npm audit > "${REPORTS_DIR}/npm-audit.txt" || true
                    npx --yes npm-audit-html --output "${REPORTS_DIR}/npm-audit.html" || true
                    echo "npm audit completed."
                '''
            }
        }

        stage('SAST: Semgrep Scan') {
            steps {
                echo "Running Semgrep SAST scan..."
                sh '''
                    docker run --rm -v "${WORKSPACE}:/src" semgrep/semgrep semgrep scan \
                        --config auto \
                        --json -o /src/reports/semgrep.json \
                        --sarif -o /src/reports/semgrep.sarif \
                        --text -o /src/reports/semgrep.txt || true
                    echo "Semgrep scan completed."
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
                        docker network create zap-net || true
                        docker run -d --name zap-app-target --network zap-net -e PORT=3000 ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}

                        echo "Waiting for Next.js app to be reachable on port 3000..."
                        for i in $(seq 1 30); do
                            if docker run --rm --network zap-net curlimages/curl:latest -s http://zap-app-target:3000 > /dev/null 2>&1; then
                                echo "Application is healthy and ready for testing!"
                                break
                            fi
                            echo "Waiting for app to start... ($i/30)"
                            sleep 2
                        done

                        chmod -R 777 "${REPORTS_DIR}"
                        docker run --rm --network zap-net -v "${REPORTS_DIR}:/zap/wrk/:rw" -t ghcr.io/zaproxy/zaproxy:stable \
                            zap-baseline.py -t http://zap-app-target:3000 -r zap-report.html -J zap-report.json -w zap-report.md -I || true
                        echo "OWASP ZAP scan completed."
                    '''
                }
            }
            post {
                always {
                    sh '''
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

        // ==========================================
        // CD / GitOps Stage for ArgoCD
        // ==========================================
        stage('Update K8s Manifests & Trigger ArgoCD') {
            steps {
                echo "Updating k8s/deployment.yaml with new image tag: ${IMAGE_TAG}..."
                script {
                    // 1. Update the image tag in the deployment file using sed
                    sh '''
                        # Replace the entire image line with the new specific build number tag
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}|g" k8s/deployment.yaml
                        
                        # Verify the change was made
                        echo "Updated deployment.yaml image to:"
                        grep "image:" k8s/deployment.yaml
                    '''

                    // 2. Commit and push back to Git to trigger ArgoCD
                    withCredentials([usernamePassword(
                        credentialsId: env.GIT_CREDENTIALS_ID,
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_PASS'
                    )]) {
                        sh '''
                            # Configure Git user for the commit
                            git config user.email "jenkins-ci@stridex.local"
                            git config user.name "Jenkins CI Bot"

                            # Stage the changed file
                            git add k8s/deployment.yaml

                            # Commit with [skip ci] to prevent Jenkins from triggering an infinite loop on this commit!
                            git commit -m "chore(k8s): update image tag to ${IMAGE_TAG} [skip ci]" || echo "No changes to commit"

                            # Push back to the CORRECT repository (sumith-k-sam/StrideX-E-commerce)
                            git remote set-url origin https://${GIT_USER}:${GIT_PASS}@github.com/sumith-k-sam/StrideX-E-commerce.git
                            git push origin HEAD:main
                            
                            # Reset remote URL to remove credentials from local config for security
                            git remote set-url origin https://github.com/sumith-k-sam/StrideX-E-commerce.git
                        '''
                    }
                    echo "✅ ArgoCD will now detect the commit in sumith-k-sam/StrideX-E-commerce and deploy version ${IMAGE_TAG} automatically."
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Archiving scan reports..."
                archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true

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
            echo "🎉 CI/CD Pipeline finished successfully! Image ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} is deployed via ArgoCD."
        }
        failure {
            echo "❌ Pipeline failed. Check console output for details."
        }
    }
}
