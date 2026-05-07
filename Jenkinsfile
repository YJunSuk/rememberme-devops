pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              name: jenkins-agent
            spec:
              containers:
              - name: docker
                image: docker:29.4.1-cli-alpine3.23
                command:
                - cat
                tty: true
                volumeMounts:
                - mountPath: "/var/run/docker.sock"
                  name: docker-socket
              - name: maven
                image: maven:3.9.6-eclipse-temurin-21
                command:
                - cat
                tty: true
              volumes:
              - name: docker-socket
                hostPath:
                  path: "/var/run/docker.sock"
            '''
        }
    }

    // ============================================
    // GitHub Webhook 자동 트리거 설정
    // ============================================
    triggers {
        githubPush()  // GitHub Push 이벤트 수신 시 자동 빌드
    }

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-access'
        GITHUB_TOKEN = credentials('github-token')
        DOCKER_REGISTRY = 'yjs0530'
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/rememberme-backend"
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/rememberme-frontend"
        GITOPS_REPO = 'github.com/YJunSuk/devops-k8s-manifests.git'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        // ============================================
        // 1. 소스 코드 체크아웃
        // ============================================
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // ============================================
        // 1.5 변경 사항 감지 (Detect Changes)
        // ============================================
        stage('Detect Changes') {
            steps {
                script {
                    // 현재 커밋과 이전 커밋(HEAD~1) 간의 변경 파일을 가져온다.
                    def changedFiles = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim().split("\n")

                    echo "Changed files:\n${changedFiles.join('\n')}"
                    
                    // 프론트엔드와 백엔드 디렉토리 변경 감지
                    env.SHOULD_BUILD_FRONTEND = changedFiles.any { it.startsWith("devops-frontend/") } ? "true" : "false"
                    env.SHOULD_BUILD_BACKEND = changedFiles.any { it.startsWith("devops-backend/") } ? "true" : "false"

                    echo "SHOULD_BUILD_FRONTEND : ${env.SHOULD_BUILD_FRONTEND}"
                    echo "SHOULD_BUILD_BACKEND : ${env.SHOULD_BUILD_BACKEND}"
                }
            }
        }

        // ============================================
        // 2. 백엔드 빌드 (Maven)
        // ============================================
        stage('Backend Build') {
            when {
                expression { return env.SHOULD_BUILD_BACKEND == "true" }
            }
            steps {
                container('maven') {
                    dir('devops-backend') {
                        sh 'mvn clean package -DskipTests -B'
                    }
                }
            }
        }
        // ============================================
        // 4. Docker 로그인
        // ============================================
        stage('Docker Login') {
            when {
                expression { return env.SHOULD_BUILD_FRONTEND == "true" || env.SHOULD_BUILD_BACKEND == "true" }
            }
            steps {
                container('docker') {
                    sh 'docker logout'

                    withCredentials([usernamePassword(
                        credentialsId: DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                    }
                }
            }
        }

        // ============================================
        // 5. Docker 이미지 빌드 & 푸시 (Backend)
        // ============================================
        stage('Docker Build & Push - Backend') {
            when {
                expression { return env.SHOULD_BUILD_BACKEND == "true" }
            }
            steps {
                container('docker') {
                    dir('devops-backend') {
                        sh "docker build --no-cache -t ${BACKEND_IMAGE}:${IMAGE_TAG} ."
                        sh "docker push ${BACKEND_IMAGE}:${IMAGE_TAG}"
                    }
                }
            }
        }

        // ============================================
        // 6. Docker 이미지 빌드 & 푸시 (Frontend)
        // ============================================
        stage('Docker Build & Push - Frontend') {
            when {
                expression { return env.SHOULD_BUILD_FRONTEND == "true" }
            }
            steps {
                container('docker') {
                    dir('devops-frontend') {
                        sh """
                            docker build --no-cache \
                                --build-arg VITE_KAKAO_CLIENT_ID=a3c925bb3ea42d42e7214bbed14cf347 \
                                --build-arg VITE_KAKAO_REDIRECT_URI=http://localhost:30180/kakao-auth \
                                --build-arg VITE_API_BASE_URL=/api/v1 \
                                -t ${FRONTEND_IMAGE}:${IMAGE_TAG} .
                        """
                        sh "docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}"
                    }
                }
            }
        }

        // ============================================
        // 7. GitOps 매니페스트 업데이트 (Downstream Job 트리거)
        // ============================================
        stage('Trigger k8s-manifests') {
            steps {
                script {
                    build job: 'university-k8s-manifests', 
                        parameters: [
                            string(name: 'IMAGE_TAG', value: "${IMAGE_TAG}"),
                            string(name: 'DID_BUILD_FRONTEND', value: "${env.SHOULD_BUILD_FRONTEND}"),
                            string(name: 'DID_BUILD_BACKEND', value: "${env.SHOULD_BUILD_BACKEND}")
                        ],
                        wait: true
                }
            }
        }
    }
}
