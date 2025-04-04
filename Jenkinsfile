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
        SERVICES = "spring-petclinic-admin-server spring-petclinic-api-gateway spring-petclinic-discovery-server spring-petclinic-customers-service spring-petclinic-genai-service spring-petclinic-vets-service spring-petclinic-visits-service"
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    COMMIT_IDS = [:]
                }
            }
        }
        stage('Checkout Code') {
            steps {
                script {
                    def branchMap = [
                        'spring-petclinic-customers-service': params.CUSTOMERS_SERVICE_BRANCH,
                        'spring-petclinic-visits-service': params.VISITS_SERVICE_BRANCH,
                        'spring-petclinic-vets-service': params.VETS_SERVICE_BRANCH,
                        'spring-petclinic-genai-service': params.GENAI_SERVICE_BRANCH,
                        'spring-petclinic-discovery-server': 'main',
                        'spring-petclinic-admin-server': 'main',
                        'spring-petclinic-api-gateway': 'main'
                    ]
                    SERVICES.split().each { service ->  
                        COMMIT_IDS[service] = checkoutService(service, branchMap[service])
                    }
                }
            }
        }
        stage('Build and Push Images') {
            steps {
                script {
                    SERVICES.split().each { service ->
                        dir(service) {
                            // Build with Maven
                            sh "chmod +x ./mvnw"
                            sh "./mvnw clean package -DskipTests"
                            
                            def tag = (COMMIT_IDS[service] && COMMIT_IDS[service] != 'main') ? COMMIT_IDS[service] : 'latest'
                            echo "Building image for ${service} with tag ${tag}"
                            
                            // Get the service name without the prefix for Docker image
                            def shortName = service.replace('spring-petclinic-', '')
                            
                            sh """
                            docker build -f docker/Dockerfile \
                                --build-arg ARTIFACT_NAME=target/${service}-0.0.1-SNAPSHOT \
                                --build-arg EXPOSED_PORT=8080 \
                                -t ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-${shortName}:${tag} .
                            """
                            
                            sh """
                            docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
                            docker push ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-${shortName}:${tag}
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
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-customers-service:${COMMIT_IDS['spring-petclinic-customers-service'] ?: 'latest'}
                      genai-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-genai-service:${COMMIT_IDS['spring-petclinic-genai-service'] ?: 'latest'}
                      vets-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-vets-service:${COMMIT_IDS['spring-petclinic-vets-service'] ?: 'latest'}
                      visits-service:
                        image: ${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-visits-service:${COMMIT_IDS['spring-petclinic-visits-service'] ?: 'latest'}
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
                url: "https://github.com/matoupine/spring-petclinic-microservices-cd.git",
                credentialsId: 'jenkins-petclinic-cd'
            ]]
        ])
        def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        return (branch == 'main') ? 'main' : commitId
    }
}