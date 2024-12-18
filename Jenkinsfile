def activeColor
def targetColor

pipeline {
    agent any

    environment {
        // Jenkins 로 부터 전달 받은 파라미터, 환경 변수를 사용하기 위한 선언
        APP_TYPE = "${params.APP_TYPE}" // nextjs, spring, express, ...
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
        APP_CREDENTIALS = credentials(params.APP_REPO_CREDENTIALS_ID ? params.APP_REPO_CREDENTIALS_ID : 'github-access') // 애플리케이션 저장소 Token 등록 Credential
        NODE_ARCH = "${params.ENV == 'prod' ? 'amd64' : 'arm64'}" // prod 환경일 때는 amd64, dev 환경일 때는 arm64
        CLUSTER_ISSUER = "${params.ENV == 'prod' ? 'letsencrypt-prod' : 'letsencrypt-staging'}" // prod 환경일 때는 letsencrypt-prod, 그 외 환경일 때는 letsencrypt-staging
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
                                    def activeService = sh(
                                        script: "kubectl get service ${env.APP_NAME} -n ${env.K8S_NAMESPACE} -o jsonpath='{.metadata.labels.color}'",
                                        returnStdout: true
                                    ).trim()

                                    if (activeService == null || activeService.isEmpty() || activeService == "null") {
                                        activeColor = "none"
                                        targetColor = "blue"
                                    } else {
                                        activeColor = activeService
                                        targetColor = (activeColor == "blue") ? "green" : "blue"
                                    }

                                    echo "현재 Active 컬러: ${activeColor}"
                                    echo "배포 Target 컬러: ${targetColor}"
                                } catch (Exception e) {
                                    activeColor = "none"
                                    targetColor = "blue"
                                    echo "서비스 확인 실패. 초기 배포로 진행합니다."
                                }
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
                                userRemoteConfigs: [[
                                    url: "${env.APP_REPO}",
                                    credentialsId: "${env.APP_CREDENTIALS}"
                                ]]
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
                                sh "cp ci-cd-templates/.dockerignore .dockerignore"

                                sh "mkdir -p k8s"

                                // 공통 변수 맵 정의
                                def variables = [
                                    'APP_NAME': env.APP_NAME ?: '',
                                    'COLOR': targetColor ?: 'blue',
                                    'PROJECT_NAME': env.PROJECT_NAME ?: '',
                                    'ENV': env.ENV ?: '',
                                    'COMPONENT': env.COMPONENT ?: '',
                                    'REPLICAS': env.REPLICAS ?: '1',
                                    'CONTAINER_PORT': env.CONTAINER_PORT ?: '',
                                    'SERVICE_PORT': env.SERVICE_PORT ?: '80',
                                    'DOMAIN': env.DOMAIN ?: '',
                                    'DOCKER_IMAGE': env.DOCKER_IMAGE ?: '',
                                    'DOCKER_TAG': env.DOCKER_TAG ?: 'latest',
                                    'NODE_ARCH': env.NODE_ARCH ?: 'amd64',
                                    'CLUSTER_ISSUER': env.CLUSTER_ISSUER ?: ''
                                ]

                                // Target 컬러용 Deployment 템플릿 처리
                                def deploymentContent = readFile "ci-cd-templates/k8s/deployment-template.yaml"

                                // kms 애플리케이션의 경우 환경 변수 파일에서 추가 변수 로드
                                if (env.APP_NAME == 'kms') {
                                    def secretsContent = """
          envFrom:
          - secretRef:
              name: ${env.APP_NAME}-secrets
                                    """

                                    // containers 항목 찾아서 env 섹션 다음에 secrets 설정 추가
                                    def pattern = ~/(?m)(.*- name: TZ\n.*value: "Asia\/Seoul")/
                                    deploymentContent = deploymentContent.replaceAll(pattern, "\$1\n${secretsContent}")

                                    def envFile = ".env.${env.ENV}"
                                    def envContent = readFile(envFile)

                                    envContent.split('\n').each { line ->
                                        if (line && !line.startsWith('#')) {
                                            def parts = line.split('=', 2)
                                            if (parts.size() == 2) {
                                                variables[parts[0].trim()] = parts[1].trim()
                                            }
                                        }
                                    }
                                }

                                variables.each { key, value ->
                                    deploymentContent = deploymentContent.replaceAll(/\$\{${key}\}/, value)
                                }

                                writeFile file: "k8s/deployment-${targetColor}.yaml", text: deploymentContent

                                // Service 템플릿 처리
                                def serviceContent = readFile "ci-cd-templates/k8s/service-template.yaml"
                                variables.each { key, value ->
                                    serviceContent = serviceContent.replaceAll(/\$\{${key}\}/, value)
                                }
                                writeFile file: "k8s/service-processed.yaml", text: serviceContent

                                // Ingress 템플릿 처리
                                def ingressContent = readFile "ci-cd-templates/k8s/ingress-template.yaml"
                                variables.each { key, value ->
                                    ingressContent = ingressContent.replaceAll(/\$\{${key}\}/, value)
                                }
                                writeFile file: "k8s/ingress-processed.yaml", text: ingressContent

                                // Secret 템플릿 처리 (kms 애플리케이션인 경우에만)
                                if (env.APP_NAME == 'kms') {
                                    def secretContent = readFile "ci-cd-templates/k8s/secret-template.yaml"
                                    variables.each { key, value ->
                                        secretContent = secretContent.replaceAll(/\$\{${key}\}/, value)
                                    }
                                    writeFile file: "k8s/secret-processed.yaml", text: secretContent
                                }

                                stash includes: 'k8s/**,Dockerfile', name: 'build-files'
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

                                    if (env.APP_TYPE == 'nextjs' || env.APP_TYPE == 'express') {
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

                stage('K8S 리소스 생성') {
                    steps {
                        container('kubectl') {
                            unstash 'build-files'
                            script {
                                // TARGET_COLOR가 제대로 설정되었는지 확인
                                echo "현재 Target 컬러: ${targetColor}"

                                // 네임스페이스 체크 및 생성
                                def namespaceExists = sh(
                                    script: "kubectl get namespace ${env.K8S_NAMESPACE}",
                                    returnStatus: true
                                ) == 0

                                if (!namespaceExists) {
                                    sh "kubectl create namespace ${env.K8S_NAMESPACE}"

                                    // Docker Hub credentials secret 생성
                                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials',
                                                                    usernameVariable: 'DOCKER_USER',
                                                                    passwordVariable: 'DOCKER_PASSWORD')]) {
                                        sh """
                                            kubectl create secret docker-registry docker-reg-cred \
                                            --namespace=${env.K8S_NAMESPACE} \
                                            --docker-server=https://index.docker.io/v1/ \
                                            --docker-username="\${DOCKER_USER}" \
                                            --docker-password="\${DOCKER_PASSWORD}" \
                                            --docker-email=your-email@example.com
                                        """
                                    }
                                }

                                // 0. kms 일 경우 secret 생성 또는 업데이트
                                if (env.APP_NAME == 'kms') {
                                    sh """
                                        kubectl apply -f k8s/secret-processed.yaml -n ${env.K8S_NAMESPACE}
                                    """
                                }

                                // 1. Target 컬러의 새로운 Deployment 생성/업데이트
                                sh """
                                    kubectl apply -f k8s/deployment-${targetColor}.yaml -n ${env.K8S_NAMESPACE}
                                """

                                // 2. Target 배포가 완전히 준비될 때까지 대기
                                sh """
                                    kubectl rollout status deployment/${env.APP_NAME}-${targetColor} -n ${env.K8S_NAMESPACE} --timeout=180s
                                """

                                // 최초 배포시에만 Service와 Ingress 생성
                                if (activeColor == "none") {
                                    echo "최초 배포 - Service와 Ingress 생성"
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
                                    kubectl patch service ${env.APP_NAME} -n ${env.K8S_NAMESPACE} -p '{"metadata":{"labels":{"color":"${targetColor}"}},"spec":{"selector":{"color":"${targetColor}"}}}'
                                """

                                // 이전 버전 제거 여부 확인
                                if (activeColor != "none") {
                                    def DELETE_OLD = input message: '이전 버전을 제거하시겠습니까?',
                                        parameters: [
                                            choice(name: 'DELETE_OLD_VERSION', choices: ['yes', 'no'], description: '이전 버전 제거 여부')
                                        ]

                                    if (DELETE_OLD == 'yes') {
                                        sh "kubectl delete deployment ${env.APP_NAME}-${activeColor} -n ${env.K8S_NAMESPACE}"
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