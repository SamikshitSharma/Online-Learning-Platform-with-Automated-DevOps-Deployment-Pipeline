
pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        APP_IMAGE = 'online-learning-platform'
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
                    docker build -t ${APP_IMAGE}:${BUILD_NUMBER} .
                    timeout 60s docker save ${APP_IMAGE}:${BUILD_NUMBER} -o app-image.tar
                    test -s app-image.tar
                '''
            }
        }

        stage('Deploy to App Servers') {
            steps {
                sh '''
                    set -eu
                    for HOST in "$APP1" "$APP2"; do
                        scp -i "$SSH_KEY" -o StrictHostKeyChecking=no \
                            app-image.tar "$SSH_USER@$HOST:/tmp/app-image.tar"

                        ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no \
                            "$SSH_USER@$HOST" \
                            "sudo docker load -i /tmp/app-image.tar &&
                             sudo docker rm -f online-learning-platform 2>/dev/null || true"
                        
                        ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no \
                            "$SSH_USER@$HOST" \
                            "sudo docker run -d --name online-learning-platform \
                             --restart unless-stopped -p 80:80 \
                             ${APP_IMAGE}:${BUILD_NUMBER}"
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
        always {
            sh 'rm -f app-image.tar'
        }
        success {
            echo 'CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'CI/CD pipeline failed. Review the stage logs.'
        }
    }
}
