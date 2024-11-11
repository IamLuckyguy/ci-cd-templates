pipeline {
    agent any

    environment {
        // 기존 환경변수 설정은 동일...
        APP_TYPE = "${params.APP_TYPE}"
        COMPONENT = "${params.COMPONENT}"
        ENV = "${params.ENV}"
        REPLICAS = "${params.REPLICAS}"
        CONTAINER_PORT = "${params.CONTAINER_PORT}"
        SERVICE_PORT = "80"
        DOMAIN = "${params.DOMAIN}"
        PROJECT_NAME = "${params.PROJECT_NAME}"
        APP_NAME = "${params.APP_NAME}"
        DOCKER_CONFIG = credentials('docker-hub-credentials')
        BRANCH = "${params.BRANCH}"
        IMAGE_TAG = "${params.IMAGE_TAG}"
        DOCKER_TAG = "${params.IMAGE_TAG ?: env.BUILD_NUMBER}"
        DOCKER_USERNAME = "wondookong"
        K8S_NAMESPACE = "${params.PROJECT_NAME}-${params.ENV}"
        DOCKER_IMAGE = "${DOCKER_USERNAME}/${K8S_NAMESPACE}-${params.APP_NAME}"
        TEMPLATE_REPO = "${scm.userRemoteConfigs[0].url}"
        TEMPLATE_BRANCH = "${params.TEMPLATE_BRANCH}"
        APP_REPO = "${params.APP_REPO}"
        NODE_ARCH = "${params.ENV == 'prod' ? 'amd64' : 'arm64'}"
        CLUSTER_ISSUER = "${params.ENV == 'prod' ? 'letsencrypt-prod' : 'letsencrypt-staging'}"
        INTERNAL_IP_RANGE = "${params.ENV == 'prod' ? '0.0.0.0/0' : '192.168.100.0/24'}"

        // Blue/Green 배포를 위한 환경변수
        ACTIVE_COLOR = ""
        TARGET_COLOR = ""
    }

    stages {
        stage('배포 파이프라인 체크아웃') {
            steps {
                stash name: 'source', includes: '**'
                script {
                    def podTemplateContent = readFile "k8s/jenkins-pod-template.yaml"
                    podTemplateContent = podTemplateContent.replaceAll('\\$\\{NODE_ARCH\\}', env.NODE_ARCH)
                    env.POD_TEMPLATE_CONTENT = podTemplateContent
                }
            }
        }

        stage('주요 파이프라인 진입') {
            agent {
                kubernetes {
                    yaml "${env.POD_TEMPLATE_CONTENT}"
                }
            }

            stages {
                stage('현재 Active 컬러 확인') {
                    steps {
                        container('kubectl') {
                            script {
                                try {
                                    def activeService = sh(
                                        script: "kubectl get service ${env.APP_NAME} -n ${env.K8S_NAMESPACE} -o jsonpath='{.metadata.labels.color}'",
                                        returnStdout: true
                                    ).trim()

                                    if (activeService == null || activeService.isEmpty() || activeService == "null") {
                                        env.ACTIVE_COLOR = "none"
                                        env.TARGET_COLOR = "blue"
                                    } else {
                                        env.ACTIVE_COLOR = activeService
                                        env.TARGET_COLOR = (env.ACTIVE_COLOR == "blue") ? "green" : "blue"
                                    }

                                    echo "현재 Active 컬러: ${env.ACTIVE_COLOR}"
                                    echo "배포 Target 컬러: ${env.TARGET_COLOR}"
                                } catch (Exception e) {
                                    env.ACTIVE_COLOR = "none"
                                    env.TARGET_COLOR = "blue"
                                    echo "서비스 확인 실패. 초기 배포로 진행합니다."
                                }
                            }
                        }
                    }
                }

                // ... 기존 stage들 유지 ...

                stage('K8S 리소스 생성') {
                    steps {
                        container('kubectl') {
                            unstash 'build-files'
                            script {
                                // 네임스페이스 체크 및 생성
                                def namespaceExists = sh(
                                    script: "kubectl get namespace ${env.K8S_NAMESPACE}",
                                    returnStatus: true
                                ) == 0

                                if (!namespaceExists) {
                                    sh "kubectl create namespace ${env.K8S_NAMESPACE}"
                                }

                                // 1. Target 컬러의 새로운 Deployment 생성/업데이트
                                sh """
                                    kubectl apply -f k8s/deployment-${env.TARGET_COLOR}.yaml -n ${env.K8S_NAMESPACE}
                                """

                                // 2. Target 배포가 완전히 준비될 때까지 대기
                                sh """
                                    kubectl rollout status deployment/${env.APP_NAME}-${env.TARGET_COLOR} -n ${env.K8S_NAMESPACE} --timeout=180s
                                """

                                // 최초 배포시에만 Service와 Ingress 생성
                                if (env.ACTIVE_COLOR == "none") {
                                    sh """
                                        kubectl apply -f k8s/service-processed.yaml -n ${env.K8S_NAMESPACE}
                                        kubectl apply -f k8s/ingress-processed.yaml -n ${env.K8S_NAMESPACE}
                                    """
                                }
                            }
                        }
                    }
                }

                stage('배포 승인') {
                    steps {
                        script {
                            env.SWITCH_APPROVED = input message: '새로운 버전으로 전환하시겠습니까?',
                                parameters: [
                                    choice(name: 'SWITCH_TRAFFIC', choices: ['yes', 'no'], description: '트래픽 전환 여부')
                                ]
                        }
                    }
                }

                stage('트래픽 전환') {
                    when {
                        environment name: 'SWITCH_APPROVED', value: 'yes'
                    }
                    steps {
                        container('kubectl') {
                            script {
                                // 3. 서비스 selector 업데이트로 트래픽 전환
                                sh """
                                    kubectl patch service ${env.APP_NAME} -n ${env.K8S_NAMESPACE} -p '{"spec":{"selector":{"color":"${env.TARGET_COLOR}"}}}'
                                """

                                // 이전 버전 제거 여부 확인
                                if (env.ACTIVE_COLOR != "none") {
                                    def DELETE_OLD = input message: '이전 버전을 제거하시겠습니까?',
                                        parameters: [
                                            choice(name: 'DELETE_OLD_VERSION', choices: ['yes', 'no'], description: '이전 버전 제거 여부')
                                        ]

                                    if (DELETE_OLD == 'yes') {
                                        sh "kubectl delete deployment ${env.APP_NAME}-${env.ACTIVE_COLOR} -n ${env.K8S_NAMESPACE}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline execution completed"
            sh "rm -rf ci-cd-templates"
        }
        success {
            echo 'Pipeline succeeded :)'
        }
        failure {
            echo 'Pipeline failed :('
        }
    }
}