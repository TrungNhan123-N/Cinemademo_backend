pipeline {
    agent any

    environment {
        REGISTRY       = "docker.io"
        DOCKER_USER    = "nhanbackend2004"
        IMAGE_NAME     = "cinemademo_backend"
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        FULL_IMAGE     = "${REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        LATEST_IMAGE   = "${REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:latest"
        
        // Server config
        SERVER_HOST    = "3.25.179.145"
        SSH_CRED_ID    = "vps-ssh-key"
        COMPOSE_FILE   = "docker-compose.prod.yml"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Cloning repository..."
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
                        echo "Building & pushing image..."
                        sh """
                            docker build -t ${FULL_IMAGE} -t ${LATEST_IMAGE} .
                            echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin ${REGISTRY}
                            docker push ${FULL_IMAGE}
                            docker push ${LATEST_IMAGE}
                        """
                    }
                }
            }
        }

        // ĐÃ ĐƯỢC CẬP NHẬT - BẮT ĐẦU TỪ ĐÂY
        stage('Deploy to Production Server') {
            steps {
                script {
                    withCredentials([
                        file(credentialsId: 'cinema-env-prod', variable: 'ENV_FILE'),
                        sshUserPrivateKey(credentialsId: SSH_CRED_ID, keyFileVariable: 'SSH_KEY')
                    ]) {
                        sh """
                            # Copy file .env từ Jenkins credential
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no "\$ENV_FILE" ubuntu@${SERVER_HOST}:/home/ubuntu/.env

                            # Copy các file từ repo (workspace)
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no ${WORKSPACE}/${COMPOSE_FILE} ubuntu@${SERVER_HOST}:/home/ubuntu/
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no ${WORKSPACE}/nginx.conf ubuntu@${SERVER_HOST}:/home/ubuntu/ || echo "nginx.conf không tồn tại, bỏ qua"
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no ${WORKSPACE}/init-ssl.sh ubuntu@${SERVER_HOST}:/home/ubuntu/ || echo "init-ssl.sh không tồn tại, bỏ qua"

                            # SSH vào VPS và deploy
                            ssh -i "\$SSH_KEY" -o StrictHostKeyChecking=no ubuntu@${SERVER_HOST} \"
                                cd /home/ubuntu && \
                                echo '=== Kiểm tra file trên VPS ===' && \
                                ls -la .env ${COMPOSE_FILE} nginx.conf init-ssl.sh && \
                                \
                                echo '=== Pull image mới nhất ===' && \
                                docker pull ${LATEST_IMAGE} || exit 1 && \
                                \
                                echo '=== Dừng và xóa container cũ ===' && \
                                docker compose -f ${COMPOSE_FILE} --env-file .env down -v || true && \
                                \
                                echo '=== Khởi động lại dịch vụ ===' && \
                                docker compose -f ${COMPOSE_FILE} --env-file .env up -d && \
                                \
                                echo '=== Chờ 15 giây để khởi động ===' && sleep 15 && \
                                \
                                echo '=== Kiểm tra API ===' && \
                                curl -f http://localhost:8000/docs && echo 'API ĐÃ CHẠY!' || \
                                (echo 'API LỖI! Xem log:' && docker compose -f ${COMPOSE_FILE} logs backend | tail -50 && exit 1)
                            \"
                        """
                    }
                }
            }
        }
        // KẾT THÚC PHẦN CẬP NHẬT
    }

    post {
        success {
            echo "BACKEND ĐÃ DEPLOY THÀNH CÔNG!"
            echo "Truy cập: https://api.cinema-demo.vn/docs"
        }
        failure {
            echo "CÓ LỖI TRONG PIPELINE! Kiểm tra log ngay!"
        }
        always {
            echo "Dọn dẹp image cũ..."
            sh 'docker image prune -f || true'
        }
    }
}