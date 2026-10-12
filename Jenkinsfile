pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        EC2_HOST = '47.131.148.193'
        EC2_USER = 'ubuntu'
        APP_DIR = '/home/ubuntu/Retail-Transaction-Analytics-Platform'
        SSH_CREDENTIALS = 'ec2-deploy-ssh'
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
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -eu

                    docker build \
                      -f backend/Dockerfile \
                      -t retail-backend:${IMAGE_TAG} .

                    docker build \
                      -f frontend/Dockerfile \
                      -t retail-frontend:${IMAGE_TAG} ./frontend

                    docker save \
                      retail-backend:${IMAGE_TAG} \
                      retail-frontend:${IMAGE_TAG} \
                      -o images.tar
                '''
            }
        }

        stage('Transfer Images to EC2') {
            steps {
                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                    sh '''
                        set -eu

                        # EC2 host key must already be trusted by Jenkins.
                        ssh -o StrictHostKeyChecking=yes \
                          "$EC2_USER@$EC2_HOST" \
                          "mkdir -p '$APP_DIR/deploy'"

                        scp -o StrictHostKeyChecking=yes \
                          images.tar \
                          "$EC2_USER@$EC2_HOST:$APP_DIR/deploy/images.tar"
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                    sh '''
                        set -eu

                        ssh -o StrictHostKeyChecking=yes \
                          "$EC2_USER@$EC2_HOST" \
                          "APP_DIR='$APP_DIR' IMAGE_TAG='$IMAGE_TAG' bash -s" <<'REMOTE'
                        set -eu
                        cd "$APP_DIR"

                        docker load -i deploy/images.tar

                        export IMAGE_TAG

                        docker compose up -d --no-build backend frontend

                        # Check that the API starts successfully.
                        for i in $(seq 1 30); do
                            if curl -fsS \
                              http://127.0.0.1:8000/openapi.json >/dev/null; then
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
                        rm -f deploy/images.tar
REMOTE
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully.'
        }

        failure {
            echo 'Deployment failed. Review the console output.'
        }

        always {
            archiveArtifacts artifacts: 'images.tar',
                             allowEmptyArchive: true
        }
    }
}
