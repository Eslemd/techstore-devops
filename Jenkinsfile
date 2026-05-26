pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "eslemd/techstore-app"
        SONARQUBE_ENV = "SonarQube"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate

                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate

                    mkdir -p test-results

                    pytest tests/test_app.py -v --tb=short \
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

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv("${SONARQUBE_ENV}") {

                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=techstore \
                        -Dsonar.projectName='TechStore E-Commerce' \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=venv/**,tests/**,**/__pycache__/**,templates/**,static/** \
                        -Dsonar.python.version=3.13 \
                        -Dsonar.python.coverage.reportPaths=coverage.xml
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''

                    DOCKER_BUILDKIT=1 docker build \
                    -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                    -t ${DOCKER_IMAGE}:latest .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker stop techstore-app || true
                    docker rm techstore-app || true

                    docker pull ${DOCKER_IMAGE}:latest

                    docker run -d \
                    --name techstore-app \
                    -p 5000:5000 \
                    ${DOCKER_IMAGE}:latest
                '''
            }
        }

       stage('Smoke Test') {
            steps {
                sh '''
                    echo "Container durumunu kontrol ediliyor..."
                    docker ps -a --filter "name=techstore-app"

                    echo "Uygulama logları kontrol ediliyor (Olası çökme nedenleri için):"
                    docker logs techstore-app

                    sleep 10

                    # Container'ın kendi içinden localine istek atmasını sağlıyoruz, network karmaşasını çözer.
                    STATUS=$(docker exec techstore-app curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/health)

                    echo "Gelen HTTP Durum Kodu: $STATUS"

                    if [ "$STATUS" != "200" ]; then
                        echo "Smoke test failed! Status: $STATUS"
                        exit 1
                    fi

                    echo "Smoke test passed!"
                '''
            }
        }
    }

post {
        always {
            sh 'docker image prune -f'
            cleanWs()
        }

        success {
            sh "curl -X POST -H 'Content-type: application/json' --data '{\"text\":\"Build Başarılı! ✅ \\nProje: ${env.JOB_NAME} \\nBuild No: #${env.BUILD_NUMBER} \\nUygulama başarıyla deploy edildi ve smoke testten geçti!\"}' https://hooks.slack.com/services/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"
        }

        failure {
            sh "curl -X POST -H 'Content-type: application/json' --data '{\"text\":\"Build Başarısız! ❌ \\nProje: ${env.JOB_NAME} \\nBuild No: #${env.BUILD_NUMBER} \\nPipeline bir aşamada hata verdi. Konsol loglarını kontrol edin.\"}' https://hooks.slack.com/services/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"
        }
    }
}