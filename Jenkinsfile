pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "charlires/webapp"
        LATEST_TAG = "latest"
        DOCKER_SERVICE = "golang_service"
        DOCKER_HUB = 'yonig15/webapp'
        USER = 'ubuntu'
        PUBLIC_DNS = 'ec2-13-53-98-71.eu-north-1.compute.amazonaws.com'
        SSH_KEY = 'aws-ssh-key-golang'
        DOCKERHUB_CREDENTIALS = 'ekpopkova-dockerhub'
        DEPLOY_PATH = '~/golang_docker/'
        SLACK_CHANNEL = '#general'
    }

    stages {
        stage('Build base image') {
            steps {
                sh 'make build-base'
            }
        }
        stage('Run the tests') {
            steps {
                //there inner problems with the project
                // sh 'make build-test'
                // sh 'make test-unit'
                // junit 'report/report.xml'

                //List the contents of the current directory for debugging purposes
                sh "ls -l"
                echo "${WORKSPACE}"
            }
        }
        stage('Build image') {
            steps {
                sh 'make build'
            }
        }
        stage('Push to registry'){
            steps{
                script {
                withCredentials([usernamePassword(credentialsId: 'dockerCredntials', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                    sh "echo $dockerHubPassword | docker login -u $dockerHubUser --password-stdin"
                }
                LATEST_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                sh "docker tag ${DOCKER_IMAGE}:${LATEST_TAG} ${DOCKER_HUB}"
                sh "docker push ${DOCKER_IMAGE}:${LATEST_TAG}"
            }
            }
            }
        stage('Install Docker to EC2'){
        steps {
            sshagent([SSH_KEY]) {
                script{
                sh """
                [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                ssh-keyscan -t rsa,dsa ${PUBLIC_DNS} >> ~/.ssh/known_hosts
                """
                    def dockerInstalled = sh(script: """
                    ssh ${USER}@${PUBLIC_DNS} 'command -v docker >/dev/null 2>&1 && echo "true" || echo "false"'
                    """, returnStdout: true).trim()

                    if (dockerInstalled == 'true') {
                        echo 'Docker is already installed'
                    } else {
                        echo 'Docker is not installed, installing docker ....'
                        sh """
                        ssh ${USER}@${PUBLIC_DNS} 'sudo apt-get -y update'
                        ssh ${USER}@${PUBLIC_DNS} 'sudo apt install -y ca-certificates curl gnupg'
                        ssh ${USER}@${PUBLIC_DNS} 'sudo install -m 0755 -d /etc/apt/keyrings'
                        ssh ${USER}@${PUBLIC_DNS} 'curl -fsSL https://download.docker.com/linux/ubuntu/gpg'
                        ssh ${USER}@${PUBLIC_DNS} 'sudo chmod a+r /etc/apt/keyrings/docker.gpg'
                        ssh ${USER}@${PUBLIC_DNS} 'sudo apt-get update'
                        ssh ${USER}@${PUBLIC_DNS} 'sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin'
                        ssh ${USER}@${PUBLIC_DNS} 'sudo systemctl status docker' 
                        ssh ${USER}@${PUBLIC_DNS} 'sudo systemctl enable docker'
                        ssh ${USER}@${PUBLIC_DNS} 'sudo docker ps'
                        ssh ${USER}@${PUBLIC_DNS} 'sudo apt install -y make'
                        """
                    }
                }
            }
        }
        }
        stage('Deploy with rolling updates') {
        steps {
            script {
            sshagent([SSH_KEY]) {  
            //docker swarm is initialized manually
            sh "ssh ${USER}@${PUBLIC_DNS} 'docker pull ${DOCKER_IMAGE}:${LATEST_TAG}'"
            if (sh(script: "ssh ${USER}@${PUBLIC_DNS} 'docker service ls | grep -q ${DOCKER_SERVICE}'", returnStatus: true) == 0) {
                    echo "The Docker service ${DOCKER_SERVICE} already exists"
                    sh "ssh ${USER}@${PUBLIC_DNS} 'docker service update --image ${DOCKER_IMAGE}:${LATEST_TAG} --update-parallelism 1 --update-delay 10s ${DOCKER_SERVICE}'"
            } else {
                    echo "The Docker service ${DOCKER_SERVICE} does not exist"
                    sh "ssh ${USER}@${PUBLIC_DNS} 'docker service create --name ${DOCKER_SERVICE} --replicas 3 ${DOCKER_IMAGE}:${LATEST_TAG}'"
            }
            }
            }
        }
        }
    stage('Push with latest release tag'){
        when {
            branch 'release/*'
        }
        steps {
            script {
                withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD"
                }
                sh "docker tag ${DOCKER_IMAGE} ${DOCKER_HUB}:release"
                sh "docker push $DOCKER_HUB:release" 
        }
    }
    }
    }
    post {
            always {
                    sh 'docker logout'
                }
            success {
                slackSend(
                    channel: SLACK_CHANNEL,
                    color: 'good', 
                    message: "Build succeeded for ${env.JOB_NAME} (${env.BUILD_NUMBER})",
                    )
            }
            failure {
                slackSend(
                    channel: SLACK_CHANNEL,
                    color: 'danger', 
                    message: "Build failed for ${env.JOB_NAME} (${env.BUILD_NUMBER})",
                    )
            }
        }
    }
// yoni golan