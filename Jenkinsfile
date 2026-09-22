pipeline {
    agent any

    environment {
        // AWS & ECR variables
        AWS_REGION       = 'us-east-2'
        AWS_ACCOUNT_ID   = '390034075362'
        ECR_REGISTRY     = '390034075362.dkr.ecr.us-east-2.amazonaws.com'
        
        // Nexus Docker Registry
        NEXUS_REGISTRY   = '10.0.10.22:8082'
        NEXUS_REPOSITORY = 'microservices-docker'
        
        // Kubernetes / Helm
        K8S_NAMESPACE    = 'microservices-staging-ns'
        EKS_CLUSTER_NAME = 'staging-eks-v2'
        HELM_RELEASE     = 'microservice'
        HELM_CHART       = 'helm/microservice'
        HELM_VALUES      = 'helm/microservice/values.yaml'
        HELM_TEST_VALUES = 'helm/microservice/values_test.yaml'
        
        // Pipeline Controls
        ENABLE_DEPLOY    = 'true'
        IMAGE_TAG        = "${BUILD_NUMBER}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Verify Prerequisites') {
            steps {
                sh '''
                    echo "Verifying required tools are installed..."
                    docker --version
                    trivy --version
                    kubectl version --client
                    helm version --short
                    aws --version
                    echo "All required tools verified."
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Detect Changed Services') {
            steps {
                script {
                    def allServices = ['auth-service', 'gateway-service', 'user-service',
                                       'admin-service', 'employee-service', 'customer-service',
                                       'hr-service', 'task-service']

                    def changedFiles = sh(
                        script: "git diff --name-only HEAD~1 HEAD || echo ''",
                        returnStdout: true
                    ).trim().split('\n')

                    env.CHANGED_SERVICES = allServices.findAll { service ->
                        changedFiles.any { it.startsWith(service + '/') }
                    }.join(',')

                    // Fallback to building/deploying all services if no specific change is detected
                    if (env.CHANGED_SERVICES == '') {
                        echo 'No microservice changes detected. Running full deployment for all services.'
                        env.CHANGED_SERVICES = allServices.join(',')
                    } else {
                        echo "Changed services: ${env.CHANGED_SERVICES}"
                    }
                }
            }
        }

        stage('Build Parent POM and Common Library') {
            // when { expression { env.CHANGED_SERVICES != '' } }
            steps {
                // -B runs in batch mode (no progress bars filling the logs)
                // -V prints the Maven version for debugging
                // -e prints full stack traces on failure
                sh 'mvn -B -V -e -N install'
                dir('common-library') {
                    sh 'mvn -B -e clean install -DskipTests'
                }
            }
        }

        stage('Pre-download Trivy Databases') {
            // when { expression { env.CHANGED_SERVICES != '' } }
            steps {
                sh '''
                    export TMPDIR=/var/lib/jenkins/trivy-cache-shared
                    mkdir -p /var/lib/jenkins/trivy-cache-shared
                    trivy image --cache-dir /var/lib/jenkins/trivy-cache-shared --download-db-only
                    trivy image --cache-dir /var/lib/jenkins/trivy-cache-shared --download-java-db-only
                '''
            }
        }

        stage('Build') {
            // when { expression { env.CHANGED_SERVICES != '' } }
            steps {
                script {
                    env.CHANGED_SERVICES.split(',').each { service ->
                        dir(service) {
                            sh 'mvn clean compile'
                        }
                    }
                }
            }
        }

        stage('Test') {
            // when { expression { env.CHANGED_SERVICES != '' } }
            steps {
                script {
                    env.CHANGED_SERVICES.split(',').each { service ->
                        dir(service) {
                            sh 'mvn test'
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            // when { expression { env.CHANGED_SERVICES != '' } }
            steps {
                script {
                    env.CHANGED_SERVICES.split(',').each { service ->
                        dir(service) {
                            withSonarQubeEnv('SonarQube') {
                                sh '''
                                   mvn clean verify
                                   mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.11.0.3922:sonar \
                                     -Dsonar.projectKey=test-microservices \
                                     -Dsonar.projectName=TEST-Microservices
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    def servicesToBuild = ['auth-service', 'gateway-service', 'user-service', 'admin-service', 'employee-service', 'customer-service', 'hr-service', 'task-service']
                    servicesToBuild.each { currentService ->
                        
                        sh "docker build -f ${currentService}/Dockerfile -t ${ECR_REGISTRY}/staging-${currentService}:${IMAGE_TAG} ."
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    def servicesToScan = ['auth-service', 'gateway-service', 'user-service', 'admin-service', 'employee-service', 'customer-service', 'hr-service', 'task-service']
                    servicesToScan.each { currentService ->
                        retry(6) {
                            sh """
                                export TMPDIR=/var/lib/jenkins/trivy-cache-shared
                                sleep \$((RANDOM % 30))
                                trivy image --cache-dir /var/lib/jenkins/trivy-cache-shared --ignorefile ${WORKSPACE}/.trivyignore --skip-db-update --skip-java-db-update --severity HIGH,CRITICAL ${ECR_REGISTRY}/staging-${currentService}:${IMAGE_TAG}
                            """
                        }
                    }
                }
            }
        }

        stage('Push Images') {
            steps {
                script {
                    def servicesToPush = ['auth-service', 'gateway-service', 'user-service', 'admin-service', 'employee-service', 'customer-service', 'hr-service', 'task-service']
                    
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"

                    servicesToPush.each { currentService ->
                        sh """
                            docker push ${ECR_REGISTRY}/staging-${currentService}:${IMAGE_TAG}
                        """
                        retry(3) {
                            withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                                sh """
                                    sleep \$((RANDOM % 25 + 15))
                                    docker login ${NEXUS_REGISTRY} -u \$NEXUS_USER -p \$NEXUS_PASS
                                    docker tag ${ECR_REGISTRY}/staging-${currentService}:${IMAGE_TAG} ${NEXUS_REGISTRY}/${NEXUS_REPOSITORY}/staging-${currentService}:${IMAGE_TAG}
                                    docker push ${NEXUS_REGISTRY}/${NEXUS_REPOSITORY}/staging-${currentService}:${IMAGE_TAG}
                                """
                            }
                        }
                    }
                }
            }
        }
        
        stage('EKS Authentication') {
            // when { expression { env.CHANGED_SERVICES != '' && env.ENABLE_DEPLOY == 'true' } }
            steps {
                sh "aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} --kubeconfig /tmp/kubeconfig-pipeline-${BUILD_NUMBER}"
            }
        }

        stage('Helm Deploy') {
            // when { expression { env.ENABLE_DEPLOY == 'true' } }
            steps {
                script {
                    def changedList = env.CHANGED_SERVICES.split(',')
                    def setArgs = changedList.collect { service ->
                        def helmServiceKey = service.replace('-service', '')
                        return "--set services.${helmServiceKey}.image=staging-${service} --set services.${helmServiceKey}.tag=${IMAGE_TAG}"
                    }.join(' ')

                    retry(5) {
                        sh """
                            export KUBECONFIG=/tmp/kubeconfig-pipeline-${BUILD_NUMBER}
                            
                            # Automatically create the namespace if it does not exist
                            kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                            
                            sleep \$((RANDOM % 15))
                            helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} -n ${K8S_NAMESPACE} --create-namespace --reuse-values -f ${HELM_VALUES} ${setArgs} || \
                            helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} -n ${K8S_NAMESPACE} --create-namespace -f ${HELM_VALUES} ${setArgs}
                        """
                    }
                }
            }
        }

        stage('Rollout Status') {
            steps {
                script {
                    env.CHANGED_SERVICES.split(',').each { service ->
                        // Dynamically determine the namespace (e.g., 'auth-service' -> 'auth-ns')
                        def targetNamespace = service.replace('-service', '') + '-ns'
                        
                        sh """
                            export KUBECONFIG=/tmp/kubeconfig-pipeline-${BUILD_NUMBER}
                            echo "Waiting for ${service} to become ready in namespace ${targetNamespace}..."
                            kubectl rollout status deployment/${service} -n ${targetNamespace} --timeout=300s
                        """
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                script {
                    def ports = ['auth-service': 8081, 'gateway-service': 8080, 'user-service': 8082,
                                 'admin-service': 8083, 'employee-service': 8085, 'customer-service': 8084,
                                 'hr-service': 8086, 'task-service': 8087]
                                 
                    env.CHANGED_SERVICES.split(',').each { service ->
                        def port = ports[service]
                        def targetNamespace = service.replace('-service', '') + '-ns'
                        
                        sh """
                            export KUBECONFIG=/tmp/kubeconfig-pipeline-${BUILD_NUMBER}
                            sleep 10
                            kubectl run smoke-test-${service}-${BUILD_NUMBER} --rm -i --restart=Never -n ${targetNamespace} \\
                                --image=curlimages/curl \\
                                -- curl -sf http://${service}.${targetNamespace}.svc.cluster.local:${port}/actuator/health
                        """
                    }
                }
            }
        }
    }
        
    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline succeeded for changed services: ${env.CHANGED_SERVICES}"
        }
        failure {
            echo "Pipeline failed. Check the stage logs above for the genuine root cause."
        }
    }
}
