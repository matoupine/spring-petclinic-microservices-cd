pipeline {
    agent any

    parameters {
        string(name: 'CUSTOMERS_SERVICE_BRANCH', defaultValue: 'main', description: 'Branch for customers-service')
        string(name: 'VISITS_SERVICE_BRANCH', defaultValue: 'main', description: 'Branch for visits-service')
        string(name: 'VETS_SERVICE_BRANCH', defaultValue: 'main', description: 'Branch for vets-service')
        string(name: 'GENAI_SERVICE_BRANCH', defaultValue: 'main', description: 'Branch for genai-service')
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        SERVICES = "eureka-service admin-server zipkin api-gateway customers-service genai-service vets-service visits-service"
        COMMIT_IDS = [:]
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    def branchMap = [
                        'customers-service': params.CUSTOMERS_SERVICE_BRANCH,
                        'visits-service': params.VISITS_SERVICE_BRANCH,
                        'vets-service': params.VETS_SERVICE_BRANCH,
                        'genai-service': params.GENAI_SERVICE_BRANCH,
                        'eureka-service': 'main',
                        'admin-server': 'main',
                        'zipkin': 'main',
                        'api-gateway': 'main'
                    ]
                    for (service in SERVICES.split()) {
                        COMMIT_IDS[service] = checkoutService(service, branchMap[service])
                    }
                }
            }
        }

        stage('Build and Push Images') {
            steps {
                script {
                    for (service in SERVICES.split()) {
                        dir(service) {
                            def tag = (COMMIT_IDS[service] && COMMIT_IDS[service] != 'main') ? COMMIT_IDS[service] : 'latest'
                            // Chỉ định đường dẫn tới Dockerfile trong thư mục docker/
                            sh "docker build -f docker/Dockerfile -t ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-${service}:${tag} ."
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
                    writeFile file: 'values.yaml', text: """
                    services:
                      eureka-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-eureka-service:latest
                      admin-server:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-admin-server:latest
                      zipkin:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-zipkin:latest
                      api-gateway:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-api-gateway:latest
                        service:
                          type: NodePort
                          port: 80
                          nodePort: 30080
                      customers-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-customers-service:${COMMIT_IDS['customers-service'] ?: 'latest'}
                      genai-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-genai-service:${COMMIT_IDS['genai-service'] ?: 'latest'}
                      vets-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-vets-service:${COMMIT_IDS['vets-service'] ?: 'latest'}
                      visits-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-visits-service:${COMMIT_IDS['visits-service'] ?: 'latest'}
                    """
                    sh "helm upgrade --install petclinic ./helm-chart -f values.yaml --namespace developer --create-namespace"
                }
            }
        }

        stage('Provide Access URL') {
            steps {
                script {
                    def workerNodeIp = sh(script: "minikube ip || kubectl get nodes -o wide | awk 'NR==2{print \$6}'", returnStdout: true).trim()
                    echo "Access the application at: http://petclinic.local:30080"
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
            userRemoteConfigs: [[
                url: "https://github.com/spring-petclinic/spring-petclinic-microservices.git",
                credentialsId: 'jenkins-petclinic'
            ]]
        ])
        def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        return (branch == 'main') ? 'main' : commitId
    }
}