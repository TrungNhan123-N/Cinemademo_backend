pipeline {
    agent any

    environment {
        REGISTRY       = "docker.io"
        DOCKER_USER    = "nhanbackend2004"                                      // thay bằng username Docker Hub của bạn
        IMAGE_NAME     = "cinemademo_backend"
        IMAGE_TAG      = "${env.BUILD_NUMBER}"                                  // tag theo build number
        FULL_IMAGE     = "${REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        LATEST_IMAGE   = "${REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:latest"
        
        // Server config
        SERVER_HOST    = "16.176.143.220"                                // ví dụ: 136.110.0.26 hoặc api.cinema-demo.vn
        SSH_CRED_ID    = "vps-ssh-key"                                          // Jenkins → Credentials → Add SSH key
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
                        bat """
                            docker build -t ${FULL_IMAGE} -t ${LATEST_IMAGE} .
                            echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin ${REGISTRY}
                            docker push ${FULL_IMAGE}
                            docker push ${LATEST_IMAGE}
                        """
                    }
                }
            }
        }

        stage('Deploy to Production Server') {
            steps {
                script {
                    echo "Deploying to production via docker-compose..."
                    withCredentials([
                        file(credentialsId: 'cinema-env-prod', variable: 'ENV_FILE'),
                        sshUserPrivateKey(credentialsId: SSH_CRED_ID, keyFileVariable: 'SSH_KEY')
                    ]) {
                        bat """
                            :: Copy files qua VPS
                            scp -i %SSH_KEY% -o StrictHostKeyChecking=no %ENV_FILE% ubuntu@${SERVER_HOST}:/home/ubuntu/.env
                            scp -i %SSH_KEY% -o StrictHostKeyChecking=no ${COMPOSE_FILE} ubuntu@${SERVER_HOST}:/home/ubuntu/
                            scp -i %SSH_KEY% -o StrictHostKeyChecking=no nginx.conf ubuntu@${SERVER_HOST}:/home/ubuntu/
                            scp -i %SSH_KEY% -o StrictHostKeyChecking=no init-ssl.sh ubuntu@${SERVER_HOST}:/home/ubuntu/

                            :: SSH vào VPS và deploy
                            ssh -i %SSH_KEY% -o StrictHostKeyChecking=no ubuntu@${SERVER_HOST} \"
                                cd /home/ubuntu &&
                                echo 'Pulling latest image...' &&
                                docker pull ${LATEST_IMAGE} &&
                                docker compose -f ${COMPOSE_FILE} --env-file .env up -d --remove-orphans &&
                                echo 'Renew SSL if needed...' &&
                                chmod +x init-ssl.sh &&
                                ./init-ssl.sh ||
                                echo 'SSL already exists or renewal skipped'
                            \"
                        """
                    }
                }
            }
        }
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
            bat "docker image prune -f || true"
        }
    }
}