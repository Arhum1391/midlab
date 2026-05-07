// EC2/Linux agent: requires Git, Python 3, Docker, and curl.
// Add the Jenkins user to the docker group (or use sudo docker in sh steps).
// Open EC2 security group inbound TCP 8000 for the evaluator.
//
// Builds are triggered by GitHub push webhooks (not pollSCM). In Jenkins, enable
// "GitHub hook trigger for GITScm polling" on this job, and in GitHub add a webhook
// to http://YOUR_JENKINS_HOST:8080/github-webhook/ (Payload: application/json,
// events: Just the push event). See README.md for full steps.

pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        IMAGE_NAME = 'midlab-api'
        CONTAINER_NAME = 'mlops-api'
    }

    stages {
        stage('Branch gate') {
            steps {
                script {
                    def b = env.BRANCH_NAME
                    if (b != null && b != 'main') {
                        error("Pipeline runs on main only (current branch: ${b}).")
                    }
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Ensure config') {
            steps {
                sh '''
                    if [ ! -f config.json ]; then
                        echo "config.json missing. Copy configs/FA23-BAI-006_config.json to config.json at repo root."
                        exit 1
                    fi
                '''
            }
        }

        stage('Train') {
            steps {
                sh '''
                    set -e
                    python3 -m venv .jenkins-venv
                    . .jenkins-venv/bin/activate
                    pip install -U pip
                    pip install -r requirements.txt
                    test -f dataset/train.csv
                    python train.py
                '''
            }
        }

        stage('Docker build') {
            steps {
                sh """
                    set -e
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest .
                """
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    set -e
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    docker run -d -p 8000:8000 --name ${CONTAINER_NAME} --restart unless-stopped ${IMAGE_NAME}:${BUILD_NUMBER}
                """
            }
        }

        stage('Smoke test') {
            steps {
                sh '''
                    set -e
                    sleep 5
                    curl -sf http://127.0.0.1:8000/metrics | head -c 500
                    echo
                '''
            }
        }
    }
}
