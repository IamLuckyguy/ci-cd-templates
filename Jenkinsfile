pipeline {
    agent any

    environment {
        // Jenkins 로 부터 전달 받은 파라미터, 환경 변수를 사용하기 위한 선언
        APP_TYPE = "${params.APP_TYPE}" // nodejs, spring, ...
        COMPONENT = "${params.COMPONENT}" // frontend, backend
        ENV = "${params.ENV}" // dev, prod, ..
        REPLICAS = "${params.REPLICAS}" // 리플리카 수
        CONTAINER_PORT = "${params.CONTAINER_PORT}" // 컨테이너 포트 NODE 기본 포트 3000, SPRING 기본 포트 8080
        SERVICE_PORT = "80" // kubernetes service port는 80으로 고정
        DOMAIN = "${params.DOMAIN}" // ingress 생성이 필요할 때 사용됨. 도메인 이름 front.kwt.co.kr, user-api-dev.kwt.co.kr, ...

        PROJECT_NAME = "${params.PROJECT_NAME}" // 프로젝트 이름 kwt
        APP_NAME = "${params.APP_NAME}" // 애플리케이션 이름 front, api-gateway, ...
        DOCKER_CONFIG = credentials('docker-hub-credentials') // docker hub 에 접속할 수 있는 credential
        BRANCH = "${params.BRANCH}" // 브랜치 배포시 사용, 브랜치 이름.
        IMAGE_TAG = "${params.IMAGE_TAG}" // Jenkins 에서 입력받은 이미지 태그, 없으면 latest/빌드 번호 사용. 이전 이미지 태그(빌드번호) 입력시 롤백 배포에 사용 가능
        DOCKER_TAG = "${params.IMAGE_TAG ?: env.BUILD_NUMBER}" // docker image tag
        DOCKER_USERNAME = "wondookong"
        K8S_NAMESPACE = "${params.PROJECT_NAME}-${params.ENV}" // 네임스페이스가 없을 경우 생성하도록
        DOCKER_IMAGE = "${DOCKER_USERNAME}/${K8S_NAMESPACE}-${params.APP_NAME}" // docker hub image 경로
        TEMPLATE_REPO = "${scm.userRemoteConfigs[0].url}" // CI/CD 템플릿 저장소 URL
        TEMPLATE_BRANCH = "${params.TEMPLATE_BRANCH}" // CI/CD 템플릿 저장소 브랜치
        APP_REPO = "${params.APP_REPO}" // 애플리케이션 저장소 URL
        NODE_ARCH = "${params.ENV == 'prod' ? 'amd64' : 'arm64'}" // prod 환경일 때는 amd64, dev 환경일 때는 arm64
        CLUSTER_ISSUER = "${params.ENV == 'prod' ? 'letsencrypt-prod' : 'letsencrypt-staging'}" // prod 환경일 때는 letsencrypt-prod, 그 외 환경일 때는 letsencrypt-staging
        INTERNAL_IP_RANGE = "${params.ENV == 'prod' ? '0.0.0.0/0' : '192.168.100.0/24'}" // prod 환경이 아닌 경우 지정된 IP 대역만 접근 가능

        // 블루-그린 배포를 위한 추가 환경 변수
        ACTIVE_COLOR = ""
        TARGET_COLOR = ""
    }

    stages {
        stage('배포 파이프라인 체크아웃, 템플릿 처리') {
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
                                    // 현재 active 서비스의 color 확인
                                    def activeService = sh(
                                        script: "kubectl get service ${env.APP_NAME} -n ${env.K8S_NAMESPACE} -o jsonpath='{.metadata.labels.color}'",
                                        returnStdout: true
                                    ).trim()

                                    // null 체크를 더 엄격하게 수정
                                    if (activeService == null || activeService.isEmpty() || activeService == "null") {
                                        echo "서비스가 없거나 color 라벨이 없습니다. 초기 배포로 진행합니다."
                                        env.ACTIVE_COLOR = "none"
                                        env.TARGET_COLOR = "blue"
                                    } else {
                                        env.ACTIVE_COLOR = activeService
                                        env.TARGET_COLOR = env.ACTIVE_COLOR == "blue" ? "green" : "blue"
                                    }

                                    echo "현재 Active 컬러: ${env.ACTIVE_COLOR}"
                                    echo "배포 Target 컬러: ${env.TARGET_COLOR}"
                                } catch (Exception e) {
                                    echo "서비스 확인 중 오류가 발생했습니다. 초기 배포로 진행합니다."
                                    env.ACTIVE_COLOR = "none"
                                    env.TARGET_COLOR = "blue"
                                }

                                // 환경변수가 null이 아님을 보장
                                env.ACTIVE_COLOR = env.ACTIVE_COLOR ?: "none"
                                env.TARGET_COLOR = env.TARGET_COLOR ?: "blue"
                            }
                        }
                    }
                }

                stage('어플리케이션 체크아웃') {
                    steps {
                        script {
                            checkout([
                                $class: 'GitSCM',
                                branches: [[name: "${env.BRANCH}"]],
                                userRemoteConfigs: [[url: "${env.APP_REPO}"]]
                            ])
                        }
                    }
                }

                stage('K8S 템플릿 파일 처리') {
                    steps {
                        container('kubectl') {
                            script {
                                // 템플릿 저장소 클론
                                dir('ci-cd-templates') {
                                    git url: env.TEMPLATE_REPO,
                                    branch: env.TEMPLATE_BRANCH
                                }

                                sh "cp ci-cd-templates/Dockerfile-${env.APP_TYPE} Dockerfile"
                                sh "mkdir -p k8s"

                                // Target 컬러용 Deployment 템플릿 처리
                                def deploymentContent = readFile "ci-cd-templates/k8s/deployment-template.yaml"
                                deploymentContent = deploymentContent
                                    .replaceAll('\\$\\{APP_NAME\\}', "${env.APP_NAME}-${env.TARGET_COLOR}")
                                    .replaceAll('\\$\\{COLOR\\}', env.TARGET_COLOR)
                                    .replaceAll('\\$\\{PROJECT_NAME\\}', env.PROJECT_NAME)
                                    .replaceAll('\\$\\{ENV\\}', env.ENV)
                                    .replaceAll('\\$\\{COMPONENT\\}', env.COMPONENT)
                                    .replaceAll('\\$\\{REPLICAS\\}', env.REPLICAS)
                                    .replaceAll('\\$\\{CONTAINER_PORT\\}', env.CONTAINER_PORT)
                                    .replaceAll('\\$\\{SERVICE_PORT\\}', env.SERVICE_PORT)
                                    .replaceAll('\\$\\{DOMAIN\\}', env.DOMAIN)
                                    .replaceAll('\\$\\{DOCKER_IMAGE\\}', env.DOCKER_IMAGE)
                                    .replaceAll('\\$\\{DOCKER_TAG\\}', env.DOCKER_TAG)
                                    .replaceAll('\\$\\{NODE_ARCH\\}', env.NODE_ARCH)
                                    .replaceAll('\\$\\{CLUSTER_ISSUER\\}', env.CLUSTER_ISSUER)
                                    .replaceAll('\\$\\{INTERNAL_IP_RANGE\\}', env.INTERNAL_IP_RANGE)
                                writeFile file: "k8s/deployment-${env.TARGET_COLOR}.yaml", text: deploymentContent

                                // Service 템플릿 처리 (블루/그린 공용)
                                def serviceContent = readFile "ci-cd-templates/k8s/service-template.yaml"
                                writeFile file: "k8s/service.yaml", text: serviceContent

                                // Ingress 템플릿 처리
                                def ingressContent = readFile "ci-cd-templates/k8s/ingress-template.yaml"
                                writeFile file: "k8s/ingress.yaml", text: ingressContent

                                stash includes: 'k8s/**,Dockerfile', name: 'build-files'
                            }
                        }
                    }
                }

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
                                    echo "Namespace does not exist. Creating new Namespace..."
                                    sh "kubectl create namespace ${env.K8S_NAMESPACE}"
                                } else {
                                    echo "Namespace already exists. Skipping creation."
                                }

                                // Target 컬러의 deployment 존재 여부 확인
                                def targetDeploymentExists = sh(
                                    script: "kubectl get deployment ${env.APP_NAME}-${env.TARGET_COLOR} -n ${env.K8S_NAMESPACE}",
                                    returnStatus: true
                                ) == 0

                                if (!targetDeploymentExists) {
                                    echo "Target Deployment does not exist. Creating new Deployment..."
                                    sh "kubectl apply -f k8s/deployment-${env.TARGET_COLOR}.yaml -n ${env.K8S_NAMESPACE}"
                                } else {
                                    echo "Target Deployment exists. Updating..."
                                    sh "kubectl apply -f k8s/deployment-${env.TARGET_COLOR}.yaml -n ${env.K8S_NAMESPACE}"
                                }

                                // Service는 처음 한번만 생성되고, 이후에는 selector만 변경됨
                                def serviceExists = sh(
                                    script: "kubectl get service ${env.APP_NAME} -n ${env.K8S_NAMESPACE}",
                                    returnStatus: true
                                ) == 0

                                if (!serviceExists) {
                                    echo "Service does not exist. Creating new Service..."
                                    // 초기 서비스 생성 시 Target 컬러를 selector로 지정
                                    def serviceContent = readFile "k8s/service.yaml"
                                    serviceContent = serviceContent
                                        .replaceAll('\\$\\{APP_NAME\\}', env.APP_NAME)
                                        .replaceAll('\\$\\{PROJECT_NAME\\}', env.PROJECT_NAME)
                                        .replaceAll('\\$\\{ENV\\}', env.ENV)
                                        .replaceAll('\\$\\{COMPONENT\\}', env.COMPONENT)
                                        .replaceAll('\\$\\{COLOR\\}', env.TARGET_COLOR)
                                        .replaceAll('\\$\\{CONTAINER_PORT\\}', env.CONTAINER_PORT)
                                        .replaceAll('\\$\\{SERVICE_PORT\\}', env.SERVICE_PORT)
                                    writeFile file: "k8s/service-processed.yaml", text: serviceContent
                                    sh "kubectl apply -f k8s/service-processed.yaml -n ${env.K8S_NAMESPACE}"
                                }

                                // Ingress 처리
                                def ingressExists = sh(
                                    script: "kubectl get ingress ${env.APP_NAME} -n ${env.K8S_NAMESPACE}",
                                    returnStatus: true
                                ) == 0

                                if (!ingressExists) {
                                    echo "Ingress does not exist. Creating new Ingress..."
                                    sh "kubectl apply -f k8s/ingress.yaml -n ${env.K8S_NAMESPACE}"
                                } else {
                                    echo "Ingress exists. Updating if needed..."
                                    sh "kubectl apply -f k8s/ingress.yaml -n ${env.K8S_NAMESPACE}"
                                }
                            }
                        }
                    }
                }

                stage('어플리케이션 빌드, 이미지 생성 후 푸시') {
                    when {
                        expression { // IMAGE_TAG 가 빈 값일 때만 빌드를 수행
                            return env.IMAGE_TAG == null || env.IMAGE_TAG.isEmpty() || env.IMAGE_TAG == ''
                        }
                    }
                    steps {
                        container('kaniko') {
                            withEnv(['DOCKER_CONFIG=/kaniko/.docker']) {
                                script {
                                    unstash 'build-files'

                                    def platform = "--custom-platform=linux/arm64"
                                    def buildArgs1 = ""
                                    def buildArgs2 = ""

                                    if (env.APP_TYPE == 'nodejs') {
                                        buildArgs1 = "--build-arg NODE_ENV=${env.ENV}"
                                    } else if (env.APP_TYPE == 'spring') {
                                        buildArgs1 = "--build-arg SPRING_PROFILES_ACTIVE=${env.ENV}"
                                    }

                                    if (env.ENV == 'prod') {
                                        platform = "--custom-platform=linux/amd64"
                                        buildArgs2 = "--build-arg PLATFORM=linux/amd64"
                                    } else {
                                        buildArgs2 = "--build-arg PLATFORM=linux/arm64"
                                    }

                                    sh """
                                        /kaniko/executor \\
                                        --context `pwd` \\
                                        ${platform} \\
                                        --destination ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} \\
                                        --destination ${env.DOCKER_IMAGE}:latest \\
                                        --dockerfile `pwd`/Dockerfile \\
                                        ${buildArgs1} \\
                                        ${buildArgs2}
                                    """
                                }
                            }
                        }
                    }
                }

                stage('Target 환경 배포') {
                    steps {
                        container('kubectl') {
                            script {
                                // Target 컬러 배포
                                sh """
                                    kubectl apply -f k8s/deployment-${env.TARGET_COLOR}.yaml -n ${env.K8S_NAMESPACE}
                                    kubectl rollout status deployment/${env.APP_NAME}-${env.TARGET_COLOR} -n ${env.K8S_NAMESPACE} --timeout=180s
                                """
                            }
                        }
                    }
                }

                stage('배포 전환 승인') {
                    steps {
                        script {
                            env.SWITCH_APPROVED = input message: 'Target 환경으로 전환하시겠습니까?',
                                parameters: [
                                    choice(name: 'SWITCH_TRAFFIC', choices: ['yes', 'no'], description: '트래픽을 Target 환경으로 전환')
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
                                // 서비스 레이블 업데이트로 트래픽 전환
                                sh """
                                    kubectl patch service ${env.APP_NAME} -n ${env.K8S_NAMESPACE} -p '{"metadata":{"labels":{"color":"${env.TARGET_COLOR}"}},"spec":{"selector":{"color":"${env.TARGET_COLOR}"}}}'
                                """

                                // 이전 버전 제거 전 확인
                                if (env.ACTIVE_COLOR != "none") {
                                    def DELETE_OLD = input message: '이전 버전을 제거하시겠습니까?',
                                        parameters: [
                                            choice(name: 'DELETE_OLD_VERSION', choices: ['yes', 'no'], description: '이전 버전 제거')
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
            // 임시 파일 정리
            sh "rm -rf ci-cd-templates"
        }
        success {
            echo 'The Pipeline succeeded :)'
        }
        failure {
            echo "Pipeline failed :("
        }
    }
}