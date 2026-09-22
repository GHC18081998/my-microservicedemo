pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {

        // AWS / ECR
        AWS_REGION       = 'us-east-2'
        AWS_ACCOUNT_ID   = '390034075362'
        ECR_REPOSITORY   = 'myproject-staging'
        ECR_REGISTRY     = '390034075362.dkr.ecr.us-east-2.amazonaws.com'

        // Nexus Docker Registry
        NEXUS_REGISTRY   = '10.0.10.22:8081'
        NEXUS_REPOSITORY = 'microservices-docker'

        // Kubernetes / Helm
        K8S_NAMESPACE    = 'microservices-staging-ns'
        EKS_CLUSTER_NAME = 'staging-eks-v2'
        HELM_RELEASE     = 'crm'
        HELM_CHART       = 'helm/microservice'
        HELM_VALUES      = 'helm/microservice/values.yaml'
        HELM_TEST_VALUES  = 'helm/microservice/values_test.yaml'

        // Keep CD off until Nexus image pulling from EKS is configured
        ENABLE_DEPLOY = 'true'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Environment Validation') {
            steps {
                sh '''
                    set -e

                    echo "===== Java ====="
                    java -version

                    echo "===== Maven ====="
                    mvn -version

                    echo "===== Docker ====="
                    docker --version

                    echo "===== AWS CLI ====="
                    aws --version

                    echo "===== AWS Identity ====="
                    aws sts get-caller-identity

                    echo "===== SonarQube ====="
                    curl -fsS http://10.0.10.49:9000/api/system/status

                    echo
                    echo "===== Nexus ====="
                    curl -fsS http://${NEXUS_REGISTRY}/service/rest/v1/status
                '''
            }
        }

        stage('Maven Build & Unit Tests') {
            steps {
                sh '''
                    set -e
                    mvn clean verify
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        set -e
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                        -Dsonar.projectKey=test-microservices \
                        -Dsonar.projectName="TEST-Microservices"
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Helm Validation') {
            steps {
                sh '''
                    set -e

                    helm lint ${HELM_CHART} \
                      -f ${HELM_VALUES} \
                      -f ${HELM_TEST_VALUES}

                    helm template ${HELM_RELEASE} ${HELM_CHART} \
                      -f ${HELM_VALUES} \
                      -f ${HELM_TEST_VALUES} \
                      > /tmp/crm-rendered.yaml
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                script {

                    def ecrServices = [
                        'gateway-service',
                        'auth-service'
                    ]

                    def nexusServices = [
                        'user-service',
                        'admin-service',
                        'employee-service',
                        'customer-service',
                        'hr-service',
                        'task-service'
                    ]

                    ecrServices.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Building ECR image: ${service}:${imageTag}"

                        sh """
                            docker build \
                              -f ${service}/Dockerfile \
                              -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${imageTag} \
                              .
                        """
                    }

                    nexusServices.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Building Nexus image: ${service}:${imageTag}"

                        sh """
                            docker build \
                              -f ${service}/Dockerfile \
                              -t ${NEXUS_REGISTRY}/${NEXUS_REPOSITORY}/${service}:${imageTag} \
                              .
                        """
                    }
                }
            }
        }
        stage('Trivy Security Scan') {
            environment {
                TMPDIR = '/var/lib/jenkins/trivy-tmp'
            }

            steps {
                script {

                    def ecrServices = [
                        'gateway-service',
                        'auth-service'
                    ]

                    def nexusServices = [
                        'user-service',
                        'admin-service',
                        'employee-service',
                        'customer-service',
                        'hr-service',
                        'task-service'
                    ]

                    ecrServices.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"
                        def image = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${imageTag}"

                        echo "Trivy scanning ECR image: ${image}"

                        sh """
                            trivy image \
                              --severity HIGH,CRITICAL \
                              --ignore-unfixed \
                              --exit-code 1 \
                              ${image}
                        """
                    }

                    nexusServices.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"
                        def image = "${NEXUS_REGISTRY}/${NEXUS_REPOSITORY}/${service}:${imageTag}"

                        echo "Trivy scanning Nexus image: ${image}"

                        sh """
                            trivy image \
                              --severity HIGH,CRITICAL \
                              --ignore-unfixed \
                              --exit-code 1 \
                              ${image}
                        """
                    }
                }
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    set -e

                    aws ecr get-login-password \
                      --region ${AWS_REGION} \
                    | docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push ECR Images') {
            steps {
                script {

                    def services = [
                        'gateway-service',
                        'auth-service'
                    ]

                    services.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Pushing ${service} to ECR"

                        sh """
                            docker push \
                              ${ECR_REGISTRY}/${ECR_REPOSITORY}:${imageTag}
                        """
                    }
                }
            }
        }

        stage('Nexus Login') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {

                    sh '''
                        set +x

                        echo "$NEXUS_PASS" | docker login \
                          ${NEXUS_REGISTRY} \
                          --username "$NEXUS_USER" \
                          --password-stdin
                    '''
                }
            }
        }

        stage('Push Nexus Images') {
            steps {
                script {

                    def services = [
                        'user-service',
                        'admin-service',
                        'employee-service',
                        'customer-service',
                        'hr-service',
                        'task-service'
                    ]

                    services.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Pushing ${service} to Nexus"

                        sh """
                            docker push \
                              ${NEXUS_REGISTRY}/${NEXUS_REPOSITORY}/${service}:${imageTag}
                        """
                    }
                }
            }
        }

        stage('Configure EKS') {

            when {
                environment name: 'ENABLE_DEPLOY', value: 'true'
            }

            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        set -e

                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER_NAME}

                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -  

                        kubectl get pods -n ${K8S_NAMESPACE}

                        kubectl create secret docker-registry nexus-registry-secret \
                          --docker-server=${NEXUS_REGISTRY} \
                          --docker-username="${NEXUS_USER}" \
                          --docker-password="${NEXUS_PASS}" \
                          --namespace=${K8S_NAMESPACE} \
                          --dry-run=client -o yaml | kubectl apply -f -

                        echo "Nexus image pull secret configured."
                    '''
                }
            }
        }

        stage('Deploy with Helm') {

            when {
                environment name: 'ENABLE_DEPLOY', value: 'true'
            }

            steps {
                sh '''
                    set -e

                    helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} \
                      --namespace ${K8S_NAMESPACE} \
                      --create-namespace \
                      -f ${HELM_VALUES} \
                      -f ${HELM_TEST_VALUES} \
                      --set services.gateway.imageTag=gateway-service-${BUILD_NUMBER} \
                      --set services.auth.imageTag=auth-service-${BUILD_NUMBER} \
                      --set services.user.imageTag=user-service-${BUILD_NUMBER} \
                      --set services.admin.imageTag=admin-service-${BUILD_NUMBER} \
                      --set services.employee.imageTag=employee-service-${BUILD_NUMBER} \
                      --set services.customer.imageTag=customer-service-${BUILD_NUMBER} \
                      --set services.hr.imageTag=hr-service-${BUILD_NUMBER} \
                      --set services.task.imageTag=task-service-${BUILD_NUMBER} \
                      --atomic \
                      --wait \
                      --timeout 10m
                '''
            }
        }

        stage('Verify Deployment') {

            when {
                environment name: 'ENABLE_DEPLOY', value: 'true'
            }

            steps {
                sh '''
                    set -e

                    echo "===== Deployments ====="
                    kubectl get deployments -n ${K8S_NAMESPACE}

                    echo "===== Pods ====="
                    kubectl get pods -n ${K8S_NAMESPACE}

                    echo "===== Services ====="
                    kubectl get svc -n ${K8S_NAMESPACE}

                    echo "===== Ingress ====="
                    kubectl get ingress -n ${K8S_NAMESPACE}
                '''
            }
        }
    }

    post {

        success {
            echo 'Microservices CI pipeline completed successfully.'
        }

        failure {
            echo 'Microservice pipeline failed. Check the failed stage logs.'
        }

        always {
            sh '''
                docker logout ${ECR_REGISTRY} >/dev/null 2>&1 || true
                docker logout ${NEXUS_REGISTRY} >/dev/null 2>&1 || true
            '''
            cleanWs()
        }
    }
}
