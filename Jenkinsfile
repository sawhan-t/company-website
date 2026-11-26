pipeline {
    agent none

    environment {
        // Docker Hub details
        DOCKERHUB_USERNAME   = 'sawhant'
        DOCKERHUB_REPOSITORY = 'company-website'
        IMAGE_NAME           = "${DOCKERHUB_USERNAME}/${DOCKERHUB_REPOSITORY}"

        // GitHub repository
        GITHUB_REPO          = 'https://github.com/sawhan-t/company-website.git'

        // Deployment server
        DEPLOY_SERVER_IP     = '172.31.14.166'
        DEPLOY_SERVER_USER   = 'ubuntu'

        // Jenkins credentials IDs
        DOCKER_CREDENTIALS   = 'dockerhub'
        SSH_CREDENTIALS      = 'deployment-server-ssh'

        // Container settings
        CONTAINER_NAME       = 'company-website'
    }

    stages {
        // STAGE 1: BUILD SERVER
        stage('Build Docker Image') {
            agent { label 'build-agent' }
            steps {
                echo 'Building image on Build Server'
                git branch: 'main', url: "${GITHUB_REPO}"
                sh '''
                    docker build -t $IMAGE_NAME:$BUILD_NUMBER .
                    docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:latest
                '''
            }
        }

        // STAGE 2: PUSH IMAGE TO DOCKER HUB
        stage('Push Image to Docker Hub') {
            agent { label 'build-agent' }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE_NAME:$BUILD_NUMBER
                        docker push $IMAGE_NAME:latest
                        docker logout
                    '''
                }
            }
        }

        // STAGE 3: DEPLOYMENT SERVER
        stage('Deploy Application') {
            agent { label 'deploy-agent' }
            steps {
                echo 'Deploying application on separate Deployment Server'
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: "${SSH_CREDENTIALS}",
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        ssh -i "$SSH_KEY" -o StrictHostKeyChecking=accept-new "$SSH_USER@$DEPLOY_SERVER_IP" "
                            docker pull $IMAGE_NAME:latest && \
                            docker stop $CONTAINER_NAME || true && \
                            docker rm $CONTAINER_NAME || true && \
                            docker run -d --name $CONTAINER_NAME --restart unless-stopped -p 9090:80 $IMAGE_NAME:latest
                        "
                    '''
                }
            }
        }

        // STAGE 4: VERIFY
        stage('Verify Deployment') {
            agent { label 'deploy-agent' }
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: "${SSH_CREDENTIALS}",
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        ssh -i "$SSH_KEY" -o StrictHostKeyChecking=accept-new "$SSH_USER@$DEPLOY_SERVER_IP" "docker ps"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Docker image built and deployed on separate servers!'
        }
        failure {
            echo 'FAILED: Check Jenkins Console Output.'
        }
    }
}
