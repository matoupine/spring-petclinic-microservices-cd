pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKER_IMAGE = "matoupine/spring-petclinic"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/matoupine/spring-petclinic-microservices.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.build("${DOCKER_IMAGE}:${commitId}")
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.withRegistry('https://registry.hub.docker.com', 'DOCKERHUB_CREDENTIALS') {
                        docker.image("${DOCKER_IMAGE}:${commitId}").push()
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    sh """
                    kubectl set image deployment/spring-petclinic vets-service=${DOCKER_IMAGE}:${commitId}
                    kubectl rollout status deployment/spring-petclinic
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Deployment thành công!'
        }
        failure {
            echo '❌ Có lỗi xảy ra!'
        }
    }
}
