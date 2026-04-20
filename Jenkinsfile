pipeline {
    agent any

    environment {
        IMAGE_NAME     = "foodie-express"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_NAME = "foodie-express-app"
        HOST_PORT      = "8080"
        CONTAINER_PORT = "80"
        REGISTRY       = "your-dockerhub-username"
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
                bat '''
                    @echo off
                    set MISSING=0
                    for %%f in (index.html menu.html about.html offers.html contact.html) do (
                        if not exist %%f (
                            echo MISSING: %%f
                            set MISSING=1
                        ) else (
                            echo Found: %%f
                        )
                    )
                    if %MISSING%==1 exit /b 1
                    echo All HTML files present.
                '''
            }
        }

        stage('Docker Build') {
            steps {
                echo "Building Docker image..."
                bat """
                    docker build -t %IMAGE_NAME%:%IMAGE_TAG% .
                    docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                """
            }
        }

        stage('Docker Test') {
            steps {
                echo "Running smoke test on container..."
                bat """
                    docker run -d --name test_container -p 9090:80 %IMAGE_NAME%:%IMAGE_TAG%
                    timeout /t 3 /nobreak
                    curl -s -o NUL -w "%%{http_code}" http://localhost:9090 > status.txt
                    set /p STATUS=<status.txt
                    docker stop test_container
                    docker rm test_container
                    del status.txt
                    if not "%STATUS%"=="200" (
                        echo Smoke test FAILED. Got: %STATUS%
                        exit /b 1
                    )
                    echo Smoke test PASSED.
                """
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
                    bat """
                        echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
                        docker tag %IMAGE_NAME%:%IMAGE_TAG% %REGISTRY%/%IMAGE_NAME%:%IMAGE_TAG%
                        docker tag %IMAGE_NAME%:latest %REGISTRY%/%IMAGE_NAME%:latest
                        docker push %REGISTRY%/%IMAGE_NAME%:%IMAGE_TAG%
                        docker push %REGISTRY%/%IMAGE_NAME%:latest
                        docker logout
                    """
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                echo "Deploying container..."
                bat """
                    docker stop %CONTAINER_NAME% || true
                    docker rm %CONTAINER_NAME% || true
                    docker run -d --name %CONTAINER_NAME% --restart unless-stopped -p %HOST_PORT%:%CONTAINER_PORT% %IMAGE_NAME%:%IMAGE_TAG%
                    echo Deployed at http://localhost:%HOST_PORT%
                """
            }
        }

        stage('Cleanup') {
            steps {
                echo "Pruning dangling images..."
                bat 'docker image prune -f'
            }
        }
    }

    post {
        success {
            echo "Pipeline completed. Build #${BUILD_NUMBER} is live."
        }
        failure {
            echo "Pipeline FAILED. Check logs above."
        }
        always {
            echo "Pipeline finished."
        }
    }
}