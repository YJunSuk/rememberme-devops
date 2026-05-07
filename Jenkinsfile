pipeline {
    agent any

    // ============================================
    // GitHub Webhook 자동 트리거 설정
    // ============================================
    triggers {
        githubPush()  // GitHub Push 이벤트 수신 시 자동 빌드
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        GITHUB_TOKEN = credentials('github-token')
        SONARQUBE_TOKEN = credentials('sonarqube-token')
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
        // 2. 백엔드 빌드 (Maven)
        // ============================================
        stage('Backend Build') {
            steps {
                dir('devops-backend') {
                    sh 'mvn clean package -DskipTests -B'
                }
            }
        }

        // ============================================
        // 3. SonarQube 코드 분석
        // ============================================
        stage('SonarQube Analysis') {
            steps {
                dir('devops-backend') {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            mvn sonar:sonar \
                                -Dsonar.projectKey=rememberme-backend \
                                -Dsonar.projectName="RememberMe Backend" \
                                -Dsonar.java.binaries=target/classes
                        '''
                    }
                }
            }
        }

        // ============================================
        // 4. Docker 이미지 빌드 & 푸시 (Backend)
        // ============================================
        stage('Docker Build & Push - Backend') {
            steps {
                dir('devops-backend') {
                    sh """
                        docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} .
                        docker tag ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest
                        echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${BACKEND_IMAGE}:latest
                    """
                }
            }
        }

        // ============================================
        // 5. Docker 이미지 빌드 & 푸시 (Frontend)
        // ============================================
        stage('Docker Build & Push - Frontend') {
            steps {
                dir('devops-frontend') {
                    sh """
                        docker build \
                            --build-arg VITE_KAKAO_CLIENT_ID=a3c925bb3ea42d42e7214bbed14cf347 \
                            --build-arg VITE_KAKAO_REDIRECT_URI=http://localhost:30180/kakao-auth \
                            --build-arg VITE_API_BASE_URL=/api/v1 \
                            -t ${FRONTEND_IMAGE}:${IMAGE_TAG} .
                        docker tag ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest
                        echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:latest
                    """
                }
            }
        }

        // ============================================
        // 6. GitOps 매니페스트 업데이트 (ArgoCD 트리거)
        // ============================================
        stage('Update GitOps Manifest') {
            steps {
                sh """
                    git clone https://\$GITHUB_TOKEN_USR:\$GITHUB_TOKEN_PSW@${GITOPS_REPO} gitops-repo
                    cd gitops-repo

                    # Backend 이미지 태그 업데이트
                    sed -i "s|image: ${BACKEND_IMAGE}:.*|image: ${BACKEND_IMAGE}:${IMAGE_TAG}|g" rememberme-backend/deployment.yaml

                    # Frontend 이미지 태그 업데이트
                    sed -i "s|image: ${FRONTEND_IMAGE}:.*|image: ${FRONTEND_IMAGE}:${IMAGE_TAG}|g" rememberme-frontend/deployment.yaml

                    git config user.email "jenkins@rememberme.com"
                    git config user.name "Jenkins CI"
                    git add .
                    git commit -m "Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"
                    git push origin main
                """
            }
        }
    }

    post {
        always {
        }
        success {
            echo "✅ Pipeline completed successfully! Image tag: ${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed. Check the logs for details."
        }
    }
}
