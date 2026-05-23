pipeline {
    agent any

    environment {
        DOCKER_IMAGE    = 'techstore-app'
        DOCKER_HUB_USER = 'eslemd'
        SONAR_HOST      = 'http://localhost:9000'
        SONAR_TOKEN     = credentials('sonar-token')
        SLACK_CHANNEL   = '#devops-techstore'
    }

    stages {

        // ── 1. KAYNAK KOD ───────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Kod GitHub'dan alındı: ${env.GIT_COMMIT?.take(7)}"
            }
        }

        // ── 2. ORTAM KURULUMU ───────────────────────────────────
        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
                echo "✅ Python sanal ortamı hazır"
            }
        }

        // ── 3. BİRİM TESTLERİ ──────────────────────────────────
        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate

                    mkdir -p test-results

                    pytest tests/test_app.py \
                        -v \
                        --tb=short \
                        --junit-xml=test-results/unit-tests.xml \
                        --cov=app \
                        --cov-report=xml:coverage.xml \
                        --cov-report=term-missing
                '''
            }

            post {
                always {
                    junit 'test-results/unit-tests.xml'

                    publishCoverage adapters: [
                        coberturaAdapter('coverage.xml')
                    ]
                }
            }
        }

        // ── 4. SONARQUBE ANALİZİ ───────────────────────────────
        stage('SonarQube Analysis') {
            steps {
                script {

                    // Jenkins Global Tool Configuration'daki isim
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {

                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=techstore \
                            -Dsonar.projectName="TechStore E-Commerce" \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=venv/**,tests/**,**/__pycache__/** \
                            -Dsonar.python.coverage.reportPaths=coverage.xml
                        """
                    }
                }
            }
        }

        // ── 5. QUALITY GATE ────────────────────────────────────
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }

                echo "✅ SonarQube kalite kapısı geçildi"
            }
        }

        // ── 6. DOCKER BUILD ────────────────────────────────────
        stage('Build Docker Image') {
            steps {

                sh """
                    docker build \
                        -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} \
                        -t ${DOCKER_IMAGE}:latest \
                        --build-arg BUILD_DATE=\$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                        --build-arg GIT_COMMIT=${env.GIT_COMMIT?.take(7)} \
                        .
                """

                echo "✅ Docker imajı oluşturuldu"
            }
        }

        // ── 7. DOCKER HUB PUSH ─────────────────────────────────
        stage('Push to Docker Hub') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh """
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin

                        docker tag ${DOCKER_IMAGE}:latest \$DOCKER_USER/${DOCKER_IMAGE}:${env.BUILD_NUMBER}

                        docker tag ${DOCKER_IMAGE}:latest \$DOCKER_USER/${DOCKER_IMAGE}:latest

                        docker push \$DOCKER_USER/${DOCKER_IMAGE}:${env.BUILD_NUMBER}

                        docker push \$DOCKER_USER/${DOCKER_IMAGE}:latest
                    """
                }

                echo "✅ Docker Hub push tamamlandı"
            }
        }

        // ── 8. DEPLOY ──────────────────────────────────────────
        stage('Deploy') {

            steps {

                sh """
                    docker stop techstore-app 2>/dev/null || true
                    docker rm techstore-app 2>/dev/null || true

                    docker run -d \
                        --name techstore-app \
                        --restart unless-stopped \
                        -p 5000:5000 \
                        ${DOCKER_HUB_USER}/${DOCKER_IMAGE}:latest

                    sleep 10
                """

                echo "✅ Deploy tamamlandı"
            }
        }

        // ── 9. SMOKE TEST ──────────────────────────────────────
        stage('Smoke Test') {

            steps {

                sh '''
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/health)

                    if [ "$STATUS" != "200" ]; then
                        echo "❌ Health endpoint başarısız"
                        exit 1
                    fi

                    STATUS2=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/)

                    if [ "$STATUS2" != "200" ]; then
                        echo "❌ Ana sayfa başarısız"
                        exit 1
                    fi

                    echo "✅ Smoke testleri başarılı"
                '''
            }
        }

        // ── 10. UI TESTLERİ ────────────────────────────────────
        stage('UI Tests') {

            steps {

                sh '''
                    . venv/bin/activate

                    pytest tests/test_ui.py -v --tb=short || true
                '''
            }
        }
    }

    // ── POST ACTIONS ───────────────────────────────────────────
    post {

        success {

            echo "🎉 Pipeline başarıyla tamamlandı!"

            script {
                try {
                    slackSend(
                        channel: env.SLACK_CHANNEL,
                        color: 'good',
                        message: """
✅ TechStore Deploy Başarılı
• Branch: ${env.BRANCH_NAME}
• Build: #${env.BUILD_NUMBER}
• Commit: ${env.GIT_COMMIT?.take(7)}
• URL: ${env.BUILD_URL}
                        """
                    )
                } catch (Exception e) {
                    echo "Slack bildirimi gönderilemedi"
                }
            }
        }

        failure {

            echo "❌ Pipeline başarısız!"

            script {
                try {
                    slackSend(
                        channel: env.SLACK_CHANNEL,
                        color: 'danger',
                        message: """
❌ TechStore Deploy Başarısız
• Branch: ${env.BRANCH_NAME}
• Build: #${env.BUILD_NUMBER}
• URL: ${env.BUILD_URL}
                        """
                    )
                } catch (Exception e) {
                    echo "Slack bildirimi gönderilemedi"
                }
            }
        }

        always {

            sh "docker image prune -f --filter 'until=72h' || true"

            cleanWs()
        }
    }
}