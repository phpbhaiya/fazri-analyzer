pipeline {
    agent any
    
    environment {
        IMAGE_NAME = 'fazri-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        CONTAINER_NAME = "${IMAGE_NAME}-prod"
        DOCKERFILE_PATH = 'Dockerfile.prod'
        
        NEXT_PUBLIC_FASTAPI_BASE_URL = credentials('NEXT_PUBLIC_FASTAPI_BASE_URL')
        NEXT_PUBLIC_CDN_URL = credentials('NEXT_PUBLIC_CDN_URL')
        
        CONTAINER_PORT = '3000'
        HOST_PORT = '3000'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Cleanup Old Container') {
            steps {
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    docker images ${IMAGE_NAME} --format "{{.ID}} {{.CreatedAt}}" | \
                    tail -n +4 | awk '{print \$1}' | xargs -r docker rmi -f || true
                """
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                        -f ${DOCKERFILE_PATH} \
                        --build-arg NEXT_PUBLIC_FASTAPI_BASE_URL=${NEXT_PUBLIC_FASTAPI_BASE_URL} \
                        --build-arg NEXT_PUBLIC_CDN_URL=${NEXT_PUBLIC_CDN_URL} \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \
                        -t ${IMAGE_NAME}:latest \
                        .
                """
            }
        }
        
        stage('Run Container') {
            steps {
                sh """
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${HOST_PORT}:${CONTAINER_PORT} \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
        
        stage('Health Check') {
            steps {
                sh """
                    sleep 10
                    docker ps | grep ${CONTAINER_NAME} || exit 1
                    
                    max_attempts=12
                    attempt=0
                    while [ \$attempt -lt \$max_attempts ]; do
                        if curl -f http://localhost:${HOST_PORT} > /dev/null 2>&1; then
                            echo "Application is healthy!"
                            exit 0
                        fi
                        attempt=\$((attempt + 1))
                        sleep 5
                    done
                    exit 1
                """
            }
        }
    }
    
    post {
        success {
            echo "✅ Deployment successful at http://localhost:${HOST_PORT}"
        }
        failure {
            sh "docker logs ${CONTAINER_NAME} || true"
        }
    }
}