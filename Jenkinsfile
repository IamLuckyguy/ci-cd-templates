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
        K8S_CONFIG = readFile 'k8s/jenkins-pod-template.yaml' // Jenkins 에서 사용할 k8s pod 설정 파일
        TEMPLATE_REPO = "https://github.com/IamLuckyguy/ci-cd-templates.git"
        TEMPLATE_BRANCH = "${params.TEMPLATE_BRANCH}" // CI/CD 템플릿 저장소 브랜치
    }

    stages {
        stage('Checkout Application') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "${env.BRANCH}"]],
                        userRemoteConfigs: [[url: "${scm.userRemoteConfigs[0].url}"]]
                    ])
                }
            }
        }

        stage('Prepare Templates') {
            steps {
                script {
                    // 템플릿 저장소 클론
                    dir('ci-cd-templates') {
                        git url: env.TEMPLATE_REPO, branch: env.TEMPLATE_BRANCH
                    }

                    // Dockerfile 복사
                    sh "cp ci-cd-templates/Dockerfile-${env.APP_TYPE} Dockerfile"

                    // K8s 템플릿 처리
                    def templates = ['deployment', 'service', 'ingress']
                    templates.each { template ->
                        def content = readFile "ci-cd-templates/k8s/${template}-template.yaml"
                        content = content.replaceAll('\\$\\{APP_NAME\\}', env.APP_NAME)
                            .replaceAll('\\$\\{PROJECT_NAME\\}', env.PROJECT_NAME)
                            .replaceAll('\\$\\{ENV\\}', env.ENV)
                            .replaceAll('\\$\\{COMPONENT\\}', env.COMPONENT)
                            .replaceAll('\\$\\{REPLICAS\\}', env.REPLICAS)
                            .replaceAll('\\$\\{CONTAINER_PORT\\}', env.CONTAINER_PORT)
                            .replaceAll('\\$\\{SERVICE_PORT\\}', env.SERVICE_PORT)
                            .replaceAll('\\$\\{DOMAIN\\}', env.DOMAIN)
                            .replaceAll('\\$\\{DOCKER_IMAGE\\}', env.DOCKER_IMAGE)
                            .replaceAll('\\$\\{DOCKER_TAG\\}', env.DOCKER_TAG)
                        writeFile file: "k8s/${template}.yaml", text: content
                    }
                }
            }
        }

        stage('Setup Kubernetes Resources') {
            agent {
                kubernetes {
                    yaml "${env.K8S_CONFIG}"
                    podRetention never()
                }
            }
            steps {
                container('kubectl') {
                    script {
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

                        def deploymentExists = sh(
                            script: "kubectl get deployment ${env.APP_NAME} -n ${env.K8S_NAMESPACE}",
                            returnStatus: true
                        ) == 0

                        if (!deploymentExists) {
                            echo "Deployment does not exist. Creating new Deployment..."
                            sh "kubectl apply -f k8s/deployment.yaml -n ${env.K8S_NAMESPACE}"
                        } else {
                            echo "Deployment already exists. Skipping creation."
                        }

                        def serviceExists = sh(
                            script: "kubectl get service ${env.APP_NAME} -n ${env.K8S_NAMESPACE}",
                            returnStatus: true
                        ) == 0

                        if (!serviceExists) {
                            echo "Service does not exist. Creating new Service..."
                            sh "kubectl apply -f k8s/service.yaml -n ${env.K8S_NAMESPACE}"
                        } else {
                            echo "Service already exists. Skipping creation."
                        }

                        def ingressExists = sh(
                            script: "kubectl get ingress ${env.APP_NAME} -n ${env.K8S_NAMESPACE}",
                            returnStatus: true
                        ) == 0

                        if (!ingressExists) {
                            echo "Ingress does not exist. Creating new Ingress..."
                            sh "kubectl apply -f k8s/ingress.yaml -n ${env.K8S_NAMESPACE}"
                        } else {
                            echo "Ingress already exists. Skipping creation."
                        }
                    }
                }
            }
        }

        stage('Build and Push with Kaniko') {
            when {
                expression { // IMAGE_TAG 가 빈 값일 때만 빌드를 수행
                    return env.IMAGE_TAG == null || env.IMAGE_TAG.isEmpty()
                }
            }
            steps {
                container('kaniko') {
                    withEnv(['DOCKER_CONFIG=/kaniko/.docker']) {
                        def buildArgs = ""

                        if (env.APP_TYPE == 'nodejs') {
                            buildArgs = "--build-arg NODE_ENV=${env.ENV}"
                        } else if (env.APP_TYPE == 'spring') {
                            buildArgs = "--build-arg SPRING_PROFILES_ACTIVE=${env.ENV}"
                        }

                        sh """
                            /kaniko/executor \\
                            --context `pwd` \\
                            --destination ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} \\
                            --destination ${env.DOCKER_IMAGE}:latest \\
                            --dockerfile `pwd`/Dockerfile-${env.APP_TYPE} \\
                            ${buildArgs}
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    script {
                        echo "K8S_NAMESPACE: ${env.K8S_NAMESPACE}"
                        echo "DEPLOYMENT_NAME: ${env.APP_NAME}"
                        def previousVersion
                        try {
                            previousVersion = sh(
                                    script: "kubectl get deployment ${env.APP_NAME} -n ${env.K8S_NAMESPACE} -o=jsonpath='{.spec.template.spec.containers[0].image}'",
                                    returnStdout: true
                            ).trim()
                            echo "Previous version: ${previousVersion}"
                        } catch (Exception e) {
                            echo "Error details: ${e.getMessage()}"
                            echo "Failed to get previous version. This might be the first deployment."
                            previousVersion = "${env.DOCKER_IMAGE}:latest"
                        }

                        sh """sed -i 's|image: .*|image: ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}|' k8s/deployment.yaml"""
                        sh "cat k8s/deployment.yaml"

                        try {
                            sh "kubectl apply -f k8s/deployment.yaml -n ${env.K8S_NAMESPACE}"
                            sh "kubectl apply -f k8s/service.yaml -n ${env.K8S_NAMESPACE}"
                            sh "kubectl apply -f k8s/ingress.yaml -n ${env.K8S_NAMESPACE}"
                            sh "kubectl rollout status deployment/${env.APP_NAME} -n ${env.K8S_NAMESPACE} --timeout=180s"
                        } catch (Exception e) {
                            echo "Deployment failed: ${e.message}"
                            if (previousVersion) {
                                echo "Rolling back to ${previousVersion}"
                                sh "kubectl set image deployment/${env.APP_NAME} ${env.APP_NAME}=${previousVersion} -n ${env.K8S_NAMESPACE}"
                                sh "kubectl rollout status deployment/${env.APP_NAME} -n ${env.K8S_NAMESPACE} --timeout=180s"
                            } else {
                                echo "No previous version available for rollback"
                            }
                            error "Deployment failed, check logs for details"
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