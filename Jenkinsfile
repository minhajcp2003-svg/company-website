
pipeline {
    agent none

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        // Replace each placeholder with the real value.
        DOCKERHUB_USERNAME = 'DOCKERHUB_USERNAME'
        DOCKERHUB_REPOSITORY = 'DOCKERHUB_REPOSITORY'
        DOCKERHUB_CREDENTIAL_ID = 'DOCKERHUB_CREDENTIAL_ID'
        DEPLOYMENT_SERVER_IP = 'DEPLOYMENT_SERVER_IP'
        DEPLOYMENT_SSH_CREDENTIAL_ID = 'DEPLOYMENT_SSH_CREDENTIAL_ID'
        GITHUB_REPOSITORY_URL = 'GITHUB_REPOSITORY_URL'

        APP_NAME = 'company-website'
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/${DOCKERHUB_REPOSITORY}"
    }

    stages {

        stage('Checkout') {
            agent {
                label 'build-server'
            }

            steps {
                git branch: 'main',
                    url: "${GITHUB_REPOSITORY_URL}"
            }
        }

        stage('Build and Push Image') {
            agent {
                label 'build-server'
            }

            steps {
                script {
                    env.DEPLOY_IMAGE =
                        "${env.IMAGE_NAME}:${env.BUILD_NUMBER}"

                    sh '''
                        docker build --pull \
                          -t "$IMAGE_NAME:$BUILD_NUMBER" .

                        docker tag \
                          "$IMAGE_NAME:$BUILD_NUMBER" \
                          "$IMAGE_NAME:latest"
                    '''

                    withCredentials([
                        usernamePassword(
                            credentialsId: "${DOCKERHUB_CREDENTIAL_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )
                    ]) {
                        sh '''
                            set +x

                            printf '%s' "$DOCKER_PASSWORD" |
                              docker login \
                                --username "$DOCKER_USER" \
                                --password-stdin

                            docker push "$DEPLOY_IMAGE"
                            docker push "$IMAGE_NAME:latest"

                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Deploy to Deployment Server') {
            agent {
                label 'build-server'
            }

            steps {
                sshagent(
                    credentials: [
                        "${DEPLOYMENT_SSH_CREDENTIAL_ID}"
                    ]
                ) {
                    sh '''
                        set -eu

                        ssh \
                          -o BatchMode=yes \
                          -o StrictHostKeyChecking=accept-new \
                          "$DEPLOYMENT_SERVER_IP" '
                            set -eu

                            docker pull "'"$DEPLOY_IMAGE"'"

                            docker rm -f "'"$APP_NAME"'" \
                              2>/dev/null || true

                            docker run -d \
                              --name "'"$APP_NAME"'" \
                              --restart unless-stopped \
                              -p 9090:80 \
                              "'"$DEPLOY_IMAGE"'"

                            for attempt in $(seq 1 15); do

                              if curl -fsS \
                                http://localhost:9090/ \
                                >/dev/null; then

                                echo "Deployment verification successful"
                                exit 0
                              fi

                              sleep 2
                            done

                            echo "Deployment verification failed" >&2

                            docker logs "'"$APP_NAME"'" || true

                            exit 1
                        '
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Company Website deployed successfully.'
        }

        failure {
            echo 'Pipeline failed. Review the stage logs.'
        }
    }
}
