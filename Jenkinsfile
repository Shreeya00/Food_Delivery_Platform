pipeline {
    agent any

    environment {
        HTDOCS_PATH = "C:\\xampp\\htdocs\\Food_Delivery_Platform"
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

        stage('Deploy to XAMPP') {
            steps {
                echo "Deploying to XAMPP htdocs..."
                bat """
                    @echo off

                    REM Create target folder if it doesn't exist
                    if not exist "%HTDOCS_PATH%" mkdir "%HTDOCS_PATH%"

                    REM Copy all project files to htdocs
                    xcopy /E /Y /I . "%HTDOCS_PATH%"

                    echo Deploy complete. Site available at http://localhost/foodie-express/
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                echo "Checking files exist in htdocs..."
                bat """
                    @echo off
                    set MISSING=0
                    for %%f in (index.html menu.html about.html offers.html contact.html) do (
                        if not exist "%HTDOCS_PATH%\\%%f" (
                            echo MISSING in htdocs: %%f
                            set MISSING=1
                        ) else (
                            echo Verified: %%f
                        )
                    )
                    if %MISSING%==1 exit /b 1
                    echo All files deployed successfully.
                """
            }
        }
    }

    post {
        success {
            echo "Pipeline completed. Visit http://localhost/foodie-express/"
        }
        failure {
            echo "Pipeline FAILED. Check logs above."
        }
        always {
            echo "Pipeline finished."
        }
    }
}