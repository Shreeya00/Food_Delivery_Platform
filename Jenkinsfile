pipeline {
    agent any

    environment {
        IMAGE_NAME     = "foodie-express"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_NAME = "foodie-express-app"
        HOST_PORT      = "8080"
        CONTAINER_PORT = "80"
        REGISTRY       = "your-dockerhub-username" // Replace with your Docker Hub username or registry URL
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Cloning repository..."
                checkout scm
            }
        }

        stage('Lint HTML') {
            steps {
                echo "Validating HTML files..."
                sh '''
                    # Check all HTML files are present
                    for file in index.html menu.html about.html offers.html contact.html; do
                        if [ ! -f "$file" ]; then
                            echo "MISSING: $file"
                            exit 1
                        fi
                        echo "Found: $file"
                    done
                    echo "All HTML files present."
                '''
            }
        }

        stage('Docker Build') {
            steps {
                echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                sh '''
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Docker Test') {
            steps {
                echo "Running smoke test on container..."
                sh '''
                    # Start a temporary test container
                    docker run -d --name test_container -p 9090:80 ${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 3

                    # Validate HTTP 200 from Nginx
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:9090)
                    echo "HTTP Status: $STATUS"

                    # Cleanup test container
                    docker stop test_container
                    docker rm test_container

                    if [ "$STATUS" != "200" ]; then
                        echo "Smoke test FAILED. Expected 200, got $STATUS"
                        exit 1
                    fi
                    echo "Smoke test PASSED."
                '''
            }
        }

        stage('Push to Registry') {
            when {
                branch 'main'
            }
            steps {
                echo "Pushing image to Docker Hub..."
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker tag ${IMAGE_NAME}:latest ${REGISTRY}/${IMAGE_NAME}:latest
                        docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${REGISTRY}/${IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                echo "Deploying container to host..."
                sh '''
                    # Stop and remove existing container if running
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true

                    # Run updated container
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${HOST_PORT}:${CONTAINER_PORT} \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "Deployed at http://localhost:${HOST_PORT}"
                '''
            }
        }

        stage('Cleanup') {
            steps {
                echo "Removing dangling Docker images..."
                sh 'docker image prune -f'
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully. Build #${BUILD_NUMBER} is live."
        }
        failure {
            echo "Pipeline FAILED at stage. Check logs above."
        }
        always {
            echo "Pipeline finished. Workspace retained for inspection."
        }
    }
}