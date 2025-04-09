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
        SERVICES = "customers-service visits-service vets-service genai-service admin-server config-server api-gateway discovery-server"
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    COMMIT_IDS = [:]
                    def branchMap = [
                        'customers-service': params.CUSTOMERS_SERVICE_BRANCH,
                        'visits-service'   : params.VISITS_SERVICE_BRANCH,
                        'vets-service'     : params.VETS_SERVICE_BRANCH,
                        'genai-service'    : params.GENAI_SERVICE_BRANCH,
                        'admin-server'     : 'main',
                        'config-server'    : 'main',
                        'api-gateway'      : 'main',
                        'discovery-server' : 'main'
                    ]
                    SERVICES.split().each { service ->
                        COMMIT_IDS[service] = checkoutService(service, branchMap[service])
                        echo "Commit ID of ${service}: ${COMMIT_IDS[service]}"
                    }
                }
            }
        }

        stage('Build & Push Docker Images via Maven') {
            steps {
                script {
                    SERVICES.split().each { service ->
                        def tag = (COMMIT_IDS[service] && COMMIT_IDS[service] != 'main') ? COMMIT_IDS[service] : 'latest'
                        def moduleName = "spring-petclinic-${service}"

                        echo "🐳 Building and pushing Docker image for ${service} using Maven buildDocker profile"

                        sh """
                            ./mvnw clean install -PbuildDocker -pl ${moduleName} -Ddocker.image.tag=${tag} -Ddocker.username=${DOCKERHUB_CREDENTIALS_USR} -Ddocker.password=${DOCKERHUB_CREDENTIALS_PSW}
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes with Helm') {
            steps {
                script {
                    def yaml = SERVICES.split().collect { service ->
                        def imageTag = COMMIT_IDS[service] ?: 'latest'
                        def imagePath = "${DOCKERHUB_CREDENTIALS_USR}/spring-petclinic-${service}:${imageTag}"
                        def serviceBlock = (service == 'api-gateway') ? """
                          service:
                            type: NodePort
                            port: 80
                            nodePort: 30080
                        """ : ""
                        """  ${service}:\n    image: ${imagePath}${serviceBlock}"""
                    }.join("\n")

                    writeFile file: 'values.yaml', text: "services:\n${yaml}"
                    sh "helm upgrade --install petclinic ./helm-chart -f values.yaml --namespace developer --create-namespace"
                }
            }
        }

        stage('Provide Access URL') {
            steps {
                script {
                    def ip = sh(script: "minikube ip || kubectl get nodes -o wide | awk 'NR==2{print \$6}'", returnStdout: true).trim()
                    echo "Access the app at: http://petclinic.local:30080"
                    echo "Add to /etc/hosts: ${ip} petclinic.local"
                }
            }
        }
    }

    post {
        success {
            echo '✅ Deployment completed successfully!'
        }
        failure {
            echo '❌ Deployment failed!'
        }
    }
}

def checkoutService(String service, String branch) {
    dir(service) {
        checkout([
            $class: 'GitSCM',
            branches: [[name: "*/${branch}"]],
            userRemoteConfigs: [[
                url: "https://github.com/matoupine/spring-petclinic-microservices-cd.git",
                credentialsId: 'jenkins-petclinic-cd'
            ]]
        ])
        return (branch == 'main') ? 'main' : sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }
}
