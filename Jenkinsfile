i
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'starbucks-webapp'
        DOCKERHUB_IMAGE = 'uday1022/starbucks-webapp'
        TEST_CONTAINER = 'starbucks-test'
        REPO_URL = 'https://github.com/udaysontakke022-alt/Starbucks-WebApp.git'
        APP_SERVER = '172.31.18.208'
        APP_USER = 'ubuntu'
    }

    options {
        timestamps()
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 45, unit: 'MINUTES')
    }

    stages {

        stage('Checkout GitHub') {
            steps {
                checkout scm
            }
        }

        stage('Verify Project Files') {
            steps {
                sh '''
                    set -eu

                    test -f Dockerfile
                    test -f package.json
                    test -f nginx.conf
                    test -f .dockerignore

                    echo "Project files verified successfully"
                    git log -1 --oneline
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    echo "Building Starbucks Docker image"

                    docker build \
                        -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    docker images ${IMAGE_NAME}
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    set -eu

                    echo "Scanning Docker image"

                    rm -f trivy-report.txt

                    trivy --config "" image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --parallel 1 \
                        --timeout 15m \
                        --exit-code 1 \
                        --format table \
                        --output trivy-report.txt \
                        ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }

            post {
                always {
                    sh '''
                        if [ -f trivy-report.txt ]; then
                            cat trivy-report.txt
                        fi
                    '''

                    archiveArtifacts(
                        artifacts: 'trivy-report.txt',
                        fingerprint: true,
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Test Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "Starting temporary test container"

                    docker rm -f ${TEST_CONTAINER} \
                        2>/dev/null || true

                    docker run -d \
                        --name ${TEST_CONTAINER} \
                        -p 127.0.0.1:8081:80 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    set -eu

                    for i in $(seq 1 30); do

                        if curl -fsS --max-time 5 \
                            http://127.0.0.1:8081/health \
                            -o /dev/null && \
                           curl -fsS --max-time 5 \
                            http://127.0.0.1:8081/ \
                            -o /dev/null; then

                            echo "Test application is healthy"
                            exit 0
                        fi

                        echo "Attempt $i: Waiting for application"
                        sleep 3

                    done

                    docker logs --tail 50 ${TEST_CONTAINER}
                    exit 1
                '''
            }
        }

        stage('Tag Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker tag \
                        ${IMAGE_NAME}:${BUILD_NUMBER} \
                        ${DOCKERHUB_IMAGE}:${BUILD_NUMBER}

                    docker tag \
                        ${IMAGE_NAME}:${BUILD_NUMBER} \
                        ${DOCKERHUB_IMAGE}:latest

                    echo "Docker images tagged successfully"
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu

                        export DOCKER_CONFIG=$(mktemp -d)

                        trap 'docker logout >/dev/null 2>&1 || true; rm -rf "$DOCKER_CONFIG"' EXIT

                        echo "$DOCKER_TOKEN" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin

                        docker push \
                            ${DOCKERHUB_IMAGE}:${BUILD_NUMBER}

                        docker push \
                            ${DOCKERHUB_IMAGE}:latest

                        echo "Docker Hub push successful"
                    '''
                }
            }
        }

        stage('Verify SSH Connection') {
            steps {
                sshagent(credentials: ['starbucks-deploy-ssh']) {
                    sh '''
                        set -eu

                        ssh -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            ${APP_USER}@${APP_SERVER} \
                            'hostname && whoami && docker ps'

                        echo "SSH connection verified"
                    '''
                }
            }
        }

        stage('Deploy to Application EC2') {
            steps {
                sshagent(credentials: ['starbucks-deploy-ssh']) {
                    sh '''
                        set -eu

                        ssh -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            ${APP_USER}@${APP_SERVER} \
                            bash -s -- "${DOCKERHUB_IMAGE}:${BUILD_NUMBER}" <<'REMOTE'

set -eu

IMAGE="$1"
CONTAINER="starbucks-webapp"
BACKUP="starbucks-backup"

echo "Pulling Docker image: $IMAGE"

docker pull "$IMAGE"

if docker container inspect "$BACKUP" >/dev/null 2>&1; then
    echo "Existing backup found. Review before deployment."
    exit 1
fi

docker container inspect "$CONTAINER" >/dev/null

rollback() {
    STATUS=$?
    trap - EXIT

    if [ "$STATUS" -ne 0 ]; then

        echo "Deployment failed. Starting rollback..."

        docker rm -f "$CONTAINER" 2>/dev/null || true

        docker rename "$BACKUP" "$CONTAINER" || exit 1

        docker start "$CONTAINER" || exit 1

        for i in $(seq 1 15); do

            if curl -fsS --max-time 5 \
                http://localhost/health >/dev/null; then

                echo "Previous application restored successfully"
                exit "$STATUS"

            fi

            sleep 2

        done

        echo "Rollback health check failed."
        echo "Manual intervention required."

        exit 1
    fi
}

echo "Preserving existing application"

docker rename "$CONTAINER" "$BACKUP"

trap rollback EXIT

docker stop "$BACKUP"

echo "Starting new Starbucks application"

docker run -d \
    --name "$CONTAINER" \
    --restart unless-stopped \
    -p 80:80 \
    "$IMAGE"

echo "Checking production application health"

for i in $(seq 1 30); do

    if curl -fsS --max-time 5 \
        http://localhost/health >/dev/null && \
       curl -fsS --max-time 5 \
        http://localhost/ >/dev/null; then

        echo "Production application is healthy"

        trap - EXIT

        docker rm "$BACKUP"

        docker ps

        echo "Production deployment successful"

        exit 0

    fi

    echo "Health check attempt $i"

    sleep 3

done

docker logs --tail 50 "$CONTAINER" || true

exit 1

REMOTE
                    '''
                }
            }
        }

        stage('Production Verification') {
            steps {
                sshagent(credentials: ['starbucks-deploy-ssh']) {
                    sh '''
                        set -eu

                        ssh -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            ${APP_USER}@${APP_SERVER} \
                            'docker ps && curl -fsS http://localhost/health && curl -fsS -o /dev/null http://localhost/'

                        echo "Production verification completed"
                    '''
                }
            }
        }
    }

    post {

        always {
            sh '''
                docker rm -f ${TEST_CONTAINER} \
                    2>/dev/null || true

                echo "Temporary container cleanup completed"
            '''
        }

        success {
            echo '========================================'
            echo 'Starbucks CI/CD Pipeline SUCCESSFUL!'
            echo 'Security Scan Passed'
            echo 'Docker Hub Push Completed'
            echo 'Production Deployment Completed'
            echo 'Application Health Verified'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo 'Starbucks CI/CD Pipeline FAILED!'
            echo 'Check Jenkins console output'
            echo 'Check deployment and rollback status'
            echo '========================================'
        }
    }
}

