pipeline {
    agent any

    environment {
        REGISTRY       = "docker.io"
        DOCKER_USER    = "nhanbackend2004"
        IMAGE_NAME     = "cinema_backend"  // ← ĐÚNG TÊN
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        FULL_IMAGE     = "${REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        LATEST_IMAGE   = "${REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:latest"
        
        SERVER_HOST    = "3.25.179.145"
        SSH_CRED_ID    = "vps-ssh-key"
        COMPOSE_FILE   = "docker-compose.prod.yml"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/TrungNhan123-N/Cinemademo_backend.git',
                    credentialsId: 'github-pat'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo "=== BUILD IMAGE ==="
                            docker build -t ${FULL_IMAGE} -t ${LATEST_IMAGE} .

                            echo "=== LOGIN DOCKER HUB ==="
                            echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin ${REGISTRY}

                            echo "=== PUSH IMAGE ==="
                            docker push ${FULL_IMAGE}
                            docker push ${LATEST_IMAGE}
                        """
                    }
                }
            }
        }

        stage('Deploy to VPS') {
            steps {
                script {
                    withCredentials([
                        file(credentialsId: 'cinema-env-prod', variable: 'ENV_FILE'),
                        sshUserPrivateKey(credentialsId: SSH_CRED_ID, keyFileVariable: 'SSH_KEY')
                    ]) {
                        sh """
                            # Copy file
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no "\$ENV_FILE" ubuntu@${SERVER_HOST}:/home/ubuntu/.env
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no ${WORKSPACE}/${COMPOSE_FILE} ubuntu@${SERVER_HOST}:/home/ubuntu/

                            # Chạy trên VPS
                            ssh -i "\$SSH_KEY" -o StrictHostKeyChecking=no ubuntu@${SERVER_HOST} \"
                                cd /home/ubuntu && \
                                docker login -u nhanbackend2004 -p '\$(cat /tmp/docker-pass || echo nopass)' || true && \
                                docker pull ${LATEST_IMAGE} && \
                                docker compose -f ${COMPOSE_FILE} --env-file .env down -v || true && \
                                docker compose -f ${COMPOSE_FILE} --env-file .env up -d && \
                                sleep 20 && \
                                curl -f http://localhost:8000/docs && echo 'API ĐÃ CHẠY!' || \
                                (echo 'LỖI!' && docker compose -f ${COMPOSE_FILE} logs backend | tail -50 && exit 1)
                            \"
                        """
                    }
                }
            }
        }
    }

    post {
        success { echo "HOÀN TẤT: https://api.cinema-demo.vn/docs" }
        failure { echo "LỖI! XEM LOG JENKINS" }
    }
}