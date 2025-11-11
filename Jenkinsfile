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

        // === ĐÃ SỬA HOÀN CHỈNH - BẮT ĐẦU TỪ ĐÂY ===
        stage('Deploy to Production Server') {
            steps {
                script {
                    withCredentials([
                        file(credentialsId: 'cinema-env-prod', variable: 'ENV_FILE'),
                        sshUserPrivateKey(credentialsId: SSH_CRED_ID, keyFileVariable: 'SSH_KEY')
                    ]) {
                        sh """
                            # Copy .env từ Jenkins credential
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no "\$ENV_FILE" ubuntu@${SERVER_HOST}:/home/ubuntu/.env

                            # Copy file từ repo
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no ${WORKSPACE}/${COMPOSE_FILE} ubuntu@${SERVER_HOST}:/home/ubuntu/
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no ${WORKSPACE}/nginx.conf ubuntu@${SERVER_HOST}:/home/ubuntu/ || true
                            scp -i "\$SSH_KEY" -o StrictHostKeyChecking=no ${WORKSPACE}/init-ssl.sh ubuntu@${SERVER_HOST}:/home/ubuntu/ || true

                            # === TOÀN BỘ CHẠY TRÊN VPS ===
                            ssh -i "\$SSH_KEY" -o StrictHostKeyChecking=no ubuntu@${SERVER_HOST} bash -s << 'EOF'
                                cd /home/ubuntu

                                echo '=== KIỂM TRA FILE ==='
                                ls -la .env ${COMPOSE_FILE} || exit 1

                                echo '=== PULL IMAGE ==='
                                docker pull ${LATEST_IMAGE} || exit 1

                                echo '=== DỌN DẸP CŨ ==='
                                docker compose -f ${COMPOSE_FILE} --env-file .env down -v || true

                                echo '=== KHỞI ĐỘNG DỊCH VỤ ==='
                                docker compose -f ${COMPOSE_FILE} --env-file .env up -d || exit 1

                                echo '=== CHỜ 20 GIÂY ==='
                                sleep 20

                                echo '=== KIỂM TRA CONTAINER ==='
                                if ! docker ps | grep -q backend; then
                                    echo 'BACKEND KHÔNG CHẠY!'
                                    docker compose -f ${COMPOSE_FILE} logs backend
                                    exit 1
                                fi

                                echo '=== TEST API TRÊN VPS ==='
                                if curl -f http://localhost:8000/docs > /dev/null 2>&1; then
                                    echo 'API ĐÃ CHẠY THÀNH CÔNG!'
                                else
                                    echo 'API LỖI! XEM LOG:'
                                    docker compose -f ${COMPOSE_FILE} logs backend | tail -50
                                    exit 1
                                fi
EOF
                        """
                    }
                }
            }
        }
        // === KẾT THÚC PHẦN SỬA ===
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