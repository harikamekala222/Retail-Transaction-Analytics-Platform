```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_DIR = '/home/ubuntu/Retail-Transaction-Analytics-Platform'
        IMAGE_TAG = "build-${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate') {
            steps {
                sh '''
                    set -eu
                    test -f docker-compose.yml
                    test -f backend/Dockerfile
                    test -f frontend/Dockerfile
                    test -f "$APP_DIR/.env"
                '''
            }
        }

        stage('Build Images') {
            steps {
                sh '''
                    set -eu
                    docker build \
                      -f backend/Dockerfile \
                      -t retail-backend:${IMAGE_TAG} .

                    docker build \
                      -f frontend/Dockerfile \
                      -t retail-frontend:${IMAGE_TAG} ./frontend
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -eu
                    cd "$APP_DIR"

                    export IMAGE_TAG

                    docker compose up -d --no-build backend frontend

                    for i in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1:8000/openapi.json >/dev/null; then
                            echo "Backend health check passed"
                            break
                        fi

                        if [ "$i" -eq 30 ]; then
                            echo "Backend health check failed"
                            docker compose logs --tail=100 backend
                            exit 1
                        fi

                        sleep 5
                    done

                    docker compose ps
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully.'
        }
        failure {
            echo 'Deployment failed. Check the console output.'
        }
    }
}
```
