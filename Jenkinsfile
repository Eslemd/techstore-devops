pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'eslemd/techstore-app'
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

        stage('SonarQube Analysis') {
            steps {
                script {

                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {

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
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                    -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                    -t ${DOCKER_IMAGE}:latest .
                """
            }
        }

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

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy') {
            steps {

                sh """
                    docker stop techstore-app || true
                    docker rm techstore-app || true

                    docker pull ${DOCKER_IMAGE}:latest

                    docker run -d \
                        --name techstore-app \
                        -p 5000:5000 \
                        ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Smoke Test') {
            steps {

                sh '''
                    sleep 10

                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/health)

                    if [ "$STATUS" != "200" ]; then
                        echo "Smoke test failed!"
                        exit 1
                    fi

                    echo "Smoke test passed!"
                '''
            }
        }
    }

    post {

        always {

            sh '''
                docker image prune -f || true
            '''

            cleanWs()
        }
    }
}