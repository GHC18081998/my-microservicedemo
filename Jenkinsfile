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
        NEXUS_REGISTRY   = '10.0.10.22:8082' 
        NEXUS_REPOSITORY = 'microservices-docker'

        // Kubernetes / Helm
        K8S_NAMESPACE    = 'microservices-staging-ns'
        EKS_CLUSTER_NAME = 'staging-eks-v2'
        HELM_RELEASE     = 'microservice'
        HELM_CHART       = 'helm/microservice'
        HELM_VALUES      = 'helm/microservice/values.yaml'
        HELM_TEST_VALUES = 'helm/microservice/values_test.yaml'

        ENABLE_DEPLOY = 'true'
    }

    stages {
        stage('Checkout & Detect Changes') {
            steps {
                checkout scm
                
                script {
                    // Unified list containing all 8 services
                    def allServices = [
                        'gateway-service', 'auth-service', 'user-service', 
                        'admin-service', 'employee-service', 'customer-service', 
                        'hr-service', 'task-service'
                    ]

                    // Detect which services changed in this commit
                    def changedFiles = sh(script: "git diff --name-only HEAD~1 HEAD || echo ''", returnStdout: true).trim().split('\n')
                    
                    def changed = allServices.findAll { service ->
                        changedFiles.any { it.startsWith(service + '/') }
                    }

                    env.CHANGED_SERVICES = changed.join(',')

                    if (env.CHANGED_SERVICES == '') {
                        echo 'No microservice changes detected. Skipping the rest of the pipeline.'
                        currentBuild.result = 'SUCCESS'
                    } else {
                        echo "Changes detected in: ${env.CHANGED_SERVICES}"
                    }
                }
            }
        }

        stage('Global Build & Quality Gate') {
            when { expression { env.CHANGED_SERVICES != '' } }
            steps {
                sh 'mvn clean verify'

                withSonarQubeEnv('sonarqube') {
                    sh '''
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                        -Dsonar.projectKey=test-microservices \
                        -Dsonar.projectName="TEST-Microservices"
                    '''
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Prepare Cluster & Trivy Cache') {
            when { expression { env.CHANGED_SERVICES != '' } }
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}"
                
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -  
                        kubectl create secret docker-registry nexus-registry-secret \
                          --docker-server=${NEXUS_REGISTRY} \
                          --docker-username="${NEXUS_USER}" \
                          --docker-password="${NEXUS_PASS}" \
                          --namespace=${K8S_NAMESPACE} \
                          --dry-run=client -o yaml | kubectl apply -f -
                    '''
                }

                sh '''
                    export TMPDIR=/var/lib/jenkins/trivy-cache-shared
                    mkdir -p /var/lib/jenkins/trivy-cache-shared
                    trivy image --cache-dir /var/lib/jenkins/trivy-cache-shared --download-db-only
                    trivy image --cache-dir /var/lib/jenkins/trivy-cache-shared --download-java-db-only
                '''
            }
        }

        stage('Parallel Build, Scan & Deploy') {
            when { expression { env.CHANGED_SERVICES != '' } }
            steps {
                script {
                    def services = env.CHANGED_SERVICES.split(',')
                    def parallelTasks = [:]

                    services.each { service ->
                        if (!service) return

                        parallelTasks["Process ${service}"] = {
                            def imageTag = "${service}-${BUILD_NUMBER}"
                            
                            // Define both URI paths
                            def ecrUri = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${imageTag}"
                            def nexusUri = "${NEXUS_REGISTRY}/${NEXUS_REPOSITORY}/${service}:${imageTag}"

                            stage("${service}: Docker Build") {
                                // Build once, but apply BOTH tags using two -t flags
                                sh "docker build -f ${service}/Dockerfile -t ${ecrUri} -t ${nexusUri} ."
                            }

                            stage("${service}: Trivy Scan") {
                                retry(3) {
                                    // We only need to scan the ECR tag since the underlying image is identical
                                    sh """
                                        export TMPDIR=/var/lib/jenkins/trivy-cache-shared
                                        trivy image --cache-dir /var/lib/jenkins/trivy-cache-shared \
                                          --skip-db-update --skip-java-db-update \
                                          --severity HIGH,CRITICAL \
                                          --ignore-unfixed \
                                          --exit-code 0 \
                                          ${ecrUri}
                                    """
                                }
                            }

                            stage("${service}: Push Image to ECR & Nexus") {
                                // 1. Push to ECR
                                sh """
                                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                    docker push ${ecrUri}
                                """
                                
                                // 2. Push to Nexus
                                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                                    sh """
                                        echo "\$NEXUS_PASS" | docker login ${NEXUS_REGISTRY} --username "\$NEXUS_USER" --password-stdin
                                        docker push ${nexusUri}
                                    """
                                }
                            }

                            stage("${service}: Helm Deploy") {
                                if (env.ENABLE_DEPLOY == 'true') {
                                    lock('helm-deploy-lock') {
                                        def serviceKey = service.replace('-service', '')
                                        sh """
                                            helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} \
                                              --namespace ${K8S_NAMESPACE} \
                                              -f ${HELM_VALUES} \
                                              -f ${HELM_TEST_VALUES} \
                                              --reuse-values \
                                              --set services.${serviceKey}.imageTag=${imageTag} \
                                              --atomic --wait --timeout 10m
                                        """
                                    }
                                }
                            }

                            stage("${service}: Rollout & Smoke Test") {
                                if (env.ENABLE_DEPLOY == 'true') {
                                    def portMap = [
                                        'auth-service': 8081, 'gateway-service': 8080, 'user-service': 8082,
                                        'admin-service': 8082, 'employee-service': 8083, 'customer-service': 8084,
                                        'hr-service': 8085, 'task-service': 8089
                                    ]
                                    def port = portMap[service]

                                    sh """
                                        kubectl rollout status deployment/${service} -n ${K8S_NAMESPACE} --timeout=300s
                                        kubectl run smoke-test-${service}-${BUILD_NUMBER} --rm -i --restart=Never \
                                          --image=curlimages/curl \
                                          --namespace ${K8S_NAMESPACE} \
                                          -- curl -sf http://${service}.${K8S_NAMESPACE}.svc.cluster.local:${port}/actuator/health
                                    """
                                }
                            }
                        }
                    }
                    parallel parallelTasks
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker logout ${ECR_REGISTRY} >/dev/null 2>&1 || true
                docker logout ${NEXUS_REGISTRY} >/dev/null 2>&1 || true
            '''
            cleanWs()
        }
        success {
            echo "Pipeline succeeded! Changed services dynamically processed: ${env.CHANGED_SERVICES}"
        }
        failure {
            echo "Pipeline failed. Check the parallel stage logs to identify the broken microservice."
        }
    }
}
