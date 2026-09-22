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

        ENABLE_DEPLOY    = 'true'
    }

    stages {
        stage('Checkout & Detect Changes') {
            steps {
                checkout scm
                
                script {
                    // Define which services go where based on your architecture
                    def ecrList = ['gateway-service', 'auth-service']
                    def nexusList = ['user-service', 'admin-service', 'employee-service', 'customer-service', 'hr-service', 'task-service']
                    def allServices = ecrList + nexusList

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
                // Run one global Maven build so SonarQube has all compiled binaries
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
                // Update Kubeconfig once globally to avoid throttling in parallel steps
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

                // Pre-download Trivy DB to prevent disk crashes during parallel scans
                sh '''
                    export TMPDIR=/var/lib/jenkins/trivy-cache-shared
                    mkdir -p /var/lib/jenkins/trivy-cache-shared
                    trivy image --cache-dir /var/lib/jenkins/trivy-cache-shared --download-db-only
                    trivy image --cache-dir /var/lib/jenkins/trivy-cache-shared --download-java-db-only
                '''
            }
        }

        stage('Parallel Build, Scan & Push') {
            when { expression { env.CHANGED_SERVICES != '' } }
            steps {
                script {
                    def ecrList = ['gateway-service', 'auth-service']
                    def services = env.CHANGED_SERVICES.split(',')
                    def parallelTasks = [:]

                    // Pre-calculate ALL Helm arguments safely BEFORE the loop starts
                    def helmArgs = ""
                    services.each { s ->
                        if (!s) return
                        def sKey = s.replace('-service', '')
                        helmArgs += " --set services.${sKey}.imageTag=${s}-${BUILD_NUMBER}"
                    }
                    env.HELM_SET_ARGS = helmArgs

                    // Define parallel tasks
                    services.each { service ->
                        if (!service) return

                        // CRITICAL FIX: Bind variable locally to avoid Groovy closure scoping bug
                        def localService = service

                        parallelTasks["Process ${localService}"] = {
                            def imageTag = "${localService}-${BUILD_NUMBER}"
                            def isEcr = ecrList.contains(localService)
                            def imageUri = isEcr 
                                ? "${ECR_REGISTRY}/${ECR_REPOSITORY}:${imageTag}" 
                                : "${NEXUS_REGISTRY}/${NEXUS_REPOSITORY}/${localService}:${imageTag}"

                            stage("${localService}: Docker Build") {
                                sh "docker build -f ${localService}/Dockerfile -t ${imageUri} ."
                            }

                            stage("${localService}: Trivy Scan") {
                                retry(3) {
                                    sh """
                                        export TMPDIR=/var/lib/jenkins/trivy-cache-shared
                                        trivy image --cache-dir /var/lib/jenkins/trivy-cache-shared \
                                          --skip-db-update --skip-java-db-update \
                                          --severity HIGH,CRITICAL \
                                          --ignore-unfixed \
                                          --exit-code 0 \
                                          ${imageUri}
                                    """
                                }
                            }

                            stage("${localService}: Push Image") {
                                if (isEcr) {
                                    sh """
                                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                        docker push ${imageUri}
                                    """
                                } else {
                                    withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                                        sh """
                                            echo "\$NEXUS_PASS" | docker login ${NEXUS_REGISTRY} --username "\$NEXUS_USER" --password-stdin
                                            docker push ${imageUri}
                                        """
                                    }
                                }
                            }
                        }
                    }
                    
                    // Execute the parallel builds
                    parallel parallelTasks
                }
            }
        }

        stage('Unified Helm Deploy') {
            when { expression { env.CHANGED_SERVICES != '' && env.ENABLE_DEPLOY == 'true' } }
            steps {
                script {
                    // Deploy ALL changed services atomically in a single execution
                    sh """
                        helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} \
                          --namespace ${K8S_NAMESPACE} \
                          -f ${HELM_VALUES} \
                          -f ${HELM_TEST_VALUES} \
                          ${env.HELM_SET_ARGS} \
                          --atomic --wait --timeout 10m
                    """
                }
            }
        }

        stage('Rollout Status & Smoke Tests') {
            when { expression { env.CHANGED_SERVICES != '' && env.ENABLE_DEPLOY == 'true' } }
            steps {
                script {
                    def portMap = [
                        'auth-service': 8081, 'gateway-service': 8080, 'user-service': 8082,
                        'admin-service': 8082, 'employee-service': 8083, 'customer-service': 8084,
                        'hr-service': 8085, 'task-service': 8089
                    ]
                    
                    def services = env.CHANGED_SERVICES.split(',')
                    for (String svc : services) {
                        if (!svc) continue
                        def port = portMap[svc]
                        
                        sh """
                            kubectl rollout status deployment/${svc} -n ${K8S_NAMESPACE} --timeout=300s
                            kubectl run smoke-test-${svc}-${BUILD_NUMBER} --rm -i --restart=Never \
                              --image=curlimages/curl \
                              --namespace ${K8S_NAMESPACE} \
                              -- curl -sf http://${svc}.${K8S_NAMESPACE}.svc.cluster.local:${port}/actuator/health
                        """
                    }
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
            echo "Pipeline failed. Check the logs to identify the broken stage."
        }
    }
}
