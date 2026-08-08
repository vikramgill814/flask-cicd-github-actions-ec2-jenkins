pipeline {
    agent any

    environment {
        VENV = "${WORKSPACE}/venv"
        STAGING_DIR = "/var/lib/jenkins/flask-staging"
        STAGING_PORT = "5001"
    }

    stages {

        stage('Build') {
            steps {
                echo 'Installing Python dependencies...'

                sh '''
                    rm -rf ${VENV}
                    python3 -m venv ${VENV}

                    ${VENV}/bin/python -m pip install --upgrade pip
                    ${VENV}/bin/pip install -r requirements.txt

                    echo "Dependencies installed successfully."
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running unit tests...'

                sh '''
                    ${VENV}/bin/pytest -v
                '''
            }
        }

        stage('Build Artifact') {
            steps {
                echo 'Creating application build artifact...'

                sh '''
                    rm -f app-build.zip

                    zip -r app-build.zip . \
                      -x ".git/*" \
                      -x ".github/*" \
                      -x "venv/*" \
                      -x "__pycache__/*" \
                      -x "*.pyc" \
                      -x "app-build.zip"

                    ls -lh app-build.zip
                '''

                archiveArtifacts artifacts: 'app-build.zip', fingerprint: true
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo 'Deploying Flask application to Jenkins staging environment...'

                sh '''
                    mkdir -p ${STAGING_DIR}

                    cp app.py ${STAGING_DIR}/
                    cp requirements.txt ${STAGING_DIR}/

                    cd ${STAGING_DIR}

                    if [ ! -d "venv" ]; then
                        python3 -m venv venv
                    fi

                    ./venv/bin/python -m pip install --upgrade pip
                    ./venv/bin/pip install -r requirements.txt

                    if [ -f flask.pid ] && kill -0 "$(cat flask.pid)" 2>/dev/null; then
                        echo "Stopping previous staging application..."
                        kill -15 "$(cat flask.pid)" || true
                        sleep 3
                    fi

                    echo "Starting Flask application on port ${STAGING_PORT}..."

                    PORT=${STAGING_PORT} \
                    setsid nohup ./venv/bin/python app.py \
                        < /dev/null > app.log 2>&1 &

                    echo $! > flask.pid

                    sleep 8

                    echo "Application log:"
                    cat app.log || true

                    echo "Running staging health check..."

                    curl --retry 5 \
                         --retry-delay 2 \
                         --retry-connrefused \
                         -f http://127.0.0.1:${STAGING_PORT}/health

                    echo "Staging deployment successful."
                '''
            }
        }
    }

    post {
        success {
            echo 'Jenkins CI/CD Pipeline completed successfully.'
        }

        failure {
            echo 'Jenkins CI/CD Pipeline failed.'
        }

        always {
            echo "Build finished: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}