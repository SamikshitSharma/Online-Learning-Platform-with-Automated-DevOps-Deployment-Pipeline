
pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        APP_IMAGE = 'online-learning-platform'
        DOCKERHUB_REPO = 'samsharma05/online-learning-platform'
        APP1 = '23.100.94.108'
        APP2 = '23.100.94.79'
        SSH_USER = 'samadmin'
        SSH_KEY = '/var/lib/jenkins/.ssh/id_ed25519'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/SamikshitSharma/Online-Learning-Platform-with-Automated-DevOps-Deployment-Pipeline.git'
            }
        }

        stage('Validate') {
            steps {
                sh '''
                    set -eu
                    test -s index.html
                    echo "index.html validation passed"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    printf 'FROM nginx:alpine\\nCOPY index.html /usr/share/nginx/html/index.html\\n' > Dockerfile
                    docker build -t "${APP_IMAGE}:${BUILD_NUMBER}" .
                    docker tag "${APP_IMAGE}:${BUILD_NUMBER}" "${DOCKERHUB_REPO}:${BUILD_NUMBER}"
                    docker tag "${APP_IMAGE}:${BUILD_NUMBER}" "${DOCKERHUB_REPO}:latest"
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x
                        trap 'docker logout >/dev/null 2>&1 || true' EXIT

                        printf '%s' "$DOCKERHUB_TOKEN" | docker login \
                            --username "$DOCKERHUB_USER" \
                            --password-stdin

                        docker push "${DOCKERHUB_REPO}:${BUILD_NUMBER}"
                        docker push "${DOCKERHUB_REPO}:latest"
                    '''
                }
            }
        }

        stage('Deploy to App Servers') {
            steps {
                sh '''
                    set -eu
                    for HOST in "$APP1" "$APP2"; do
                        ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no \
                            "$SSH_USER@$HOST" \
                            "sudo docker pull ${DOCKERHUB_REPO}:${BUILD_NUMBER} &&
                             (sudo docker rm -f online-learning-platform >/dev/null 2>&1 || true) &&
                             sudo docker run -d --name online-learning-platform \
                             --restart unless-stopped -p 80:80 \
                             ${DOCKERHUB_REPO}:${BUILD_NUMBER}"
                    done
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -eu
                    for URL in "http://${APP1}" "http://${APP2}"; do
                        CODE=$(curl -sS -o /dev/null -w '%{http_code}' --max-time 15 "$URL")
                        echo "$URL -> HTTP $CODE"
                        test "$CODE" = "200"
                    done
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'CI/CD pipeline failed. Review the stage logs.'
        }
    }
}
