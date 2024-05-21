pipeline {
    agent any

    triggers {
        pollSCM('* * * * *')
    }

    environment {
        DOCKER_HUB_REPO = 'dockerhub-username/node-app'
        DOCKER_HUB_CREDENTIALS = 'hub-credentials-id'
        STAGING_SERVER = 'user@your-staging-server-ip'
        SSH_CREDENTIALS = 'ssh-credentials-id'
        CURRENT_VERSION = "${env.BUILD_ID}"
        PREVIOUS_VERSION = '' 
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/seshi7/express.git'

            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${DOCKER_HUB_REPO}:${CURRENT_VERSION}")
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    dockerImage.inside {
                        sh 'npm install'
                        sh 'npm test'
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_HUB_CREDENTIALS) {
                        dockerImage.push('latest')
                        dockerImage.push(CURRENT_VERSION)
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                script {
                    sshagent([SSH_CREDENTIALS]) {
                            PREVIOUS_VERSION = sh (
                            script: "ssh -o StrictHostKeyChecking=no ${STAGING_SERVER} 'docker ps --filter \"name=node-app\" --format \"{{.Image}}\" | awk -F: \"{print \\$2}\" || true'",
                            returnStdout: true
                        ).trim()
                        
                        // Deploying the new version
                        try {
                            sh """
                            ssh -o StrictHostKeyChecking=no ${STAGING_SERVER} '
                            docker pull ${DOCKER_HUB_REPO}:${CURRENT_VERSION} &&
                            docker stop node-app || true &&
                            docker rm node-app || true &&
                            docker run -d -p 8080:8080 --name node-app ${DOCKER_HUB_REPO}:${CURRENT_VERSION}
                            '
                            """
                        } catch (Exception e) {
                            // Rollback to the previous version if deployment fails
                            if (PREVIOUS_VERSION) {
                                sh """
                                ssh -o StrictHostKeyChecking=no ${STAGING_SERVER} '
                                docker pull ${DOCKER_HUB_REPO}:${PREVIOUS_VERSION} &&
                                docker stop node-app || true &&
                                docker rm node-app || true &&
                                docker run -d -p 8080:8080 --name node-app ${DOCKER_HUB_REPO}:${PREVIOUS_VERSION}
                                '
                                """
                            } else {
                                error "Deployment failed and no previous version to rollback to."
                            }
                            error "Deployment failed, rolled back to previous version."
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
