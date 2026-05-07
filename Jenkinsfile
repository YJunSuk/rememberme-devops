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
        // 2. 백엔드 빌드 (Maven)
        // ============================================
        stage('Backend Build') {
            steps {
                container('maven') {
                    dir('devops-backend') {
                        sh 'mvn clean package -DskipTests -B'
                    }
                }
            }
        }

        // ============================================
        // 3. SonarQube 코드 분석
        // ============================================
        stage('SonarQube Analysis') {
            steps {
                container('maven') {
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
        }

        // ============================================
        // 4. Docker 로그인
        // ============================================
        stage('Docker Login') {
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
        // 7. GitOps 매니페스트 업데이트 (ArgoCD 트리거)
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
}
