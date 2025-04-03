pipeline {
    agent any

    parameters {
        string(name: 'SERVICE_NAME', defaultValue: 'vets-service', description: 'Service to deploy (e.g., vets-service)')
        string(name: 'SERVICE_BRANCH', defaultValue: 'main', description: 'Branch for the specified service (e.g., dev_vets_service)')
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        SERVICES = "eureka-service admin-server zipkin api-gateway customers-service genai-service vets-service visits-service"
        COMMIT_ID = ''
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    // Checkout the specified branch for the selected service
                    COMMIT_ID = checkoutService(params.SERVICE_NAME, params.SERVICE_BRANCH)
                    // Checkout main branch for other services
                    for (service in SERVICES.split()) {
                        if (service != params.SERVICE_NAME) {
                            checkoutService(service, 'main')
                        }
                    }
                }
            }
        }

        stage('Build and Push Images') {
            steps {
                script {
                    for (service in SERVICES.split()) {
                        dir(service) {
                            def tag = (service == params.SERVICE_NAME) ? COMMIT_ID : 'latest'
                            sh "docker build -t ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-${service}:${tag} ."
                            sh """
                            docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
                            docker push ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-${service}:${tag}
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes with Helm') {
            steps {
                script {
                    // Create a Helm values file dynamically
                    writeFile file: 'values.yaml', text: """
                    services:
                      admin-server:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-admin-server:latest
                      api-gateway:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-api-gateway:latest
                        service:
                          type: NodePort
                          port: 80
                          nodePort: 30080
                      customers-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-customers-service:latest                            
                      genai-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-genai-service:latest
                      vets-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-vets-service:${params.SERVICE_NAME == 'vets-service' ? COMMIT_ID : 'latest'}
                      visits-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-visits-service:latest
                    """
                    // Deploy using Helm
                    sh "helm upgrade --install petclinic ./helm-chart -f values.yaml --namespace developer --create-namespace"
                }
            }
        }

        stage('Provide Access URL') {
            steps {
                script {
                    def workerNodeIp = sh(script: "kubectl get nodes -o wide | grep worker | awk '{print \$6}'", returnStdout: true).trim()
                    echo "Access the application at: petclinic.local:${workerNodeIp}:30080"
                    echo "Add to your /etc/hosts: ${workerNodeIp} petclinic.local"
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

def checkoutService(String service, String branch) {
    dir(service) {
        def checkoutResult = checkout([
            $class: 'GitSCM',
            branches: [[name: "*/${branch}"]],
            doGenerateSubmoduleConfigurations: false,
            extensions: [],
            submoduleCfg: [],
            userRemoteConfigs: [[
                url: "https://github.com/spring-petclinic/spring-petclinic-microservices.git",
                credentialsId: 'jenkins-petclinic'
            ]]
        ])
        return sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }
}