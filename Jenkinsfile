pipeline {
    agent any

    environment {
        // --- 1. CẤU HÌNH HARBOR ---
        HARBOR_DOMAIN       = "harbor.local.thongdev.site"
        HARBOR_PROJECT      = "lab-final" 

        // --- 2. CẤU HÌNH IMAGE TÊN ---
        IMAGE_FRONTEND      = "${HARBOR_DOMAIN}/${HARBOR_PROJECT}/frontend"
        IMAGE_BACKEND_PHP   = "${HARBOR_DOMAIN}/${HARBOR_PROJECT}/backend"
        IMAGE_BACKEND_NGINX = "${HARBOR_DOMAIN}/${HARBOR_PROJECT}/nginx-backend"
        IMAGE_PHP_BASE      = "${HARBOR_DOMAIN}/${HARBOR_PROJECT}/php-8.4-base"

        // --- 3. CẤU HÌNH HELM CHART ---
        CHART_BASE_PATH     = "charts/yorisoi-stack"
        CHART_LOCAL_PATH    = "charts/yorisoi-local"
        
        // --- 4. CẤU HÌNH GIT REPO ---
        GIT_URL_FRONTEND    = "https://gitlab.local.thongdev.site/tonylab/frontend.git"
        GIT_URL_BACKEND     = "https://gitlab.local.thongdev.site/tonylab/backend.git"
        GIT_BRANCH          = "main"

        // --- 5. VERSIONING ---
        BUILD_VERSION       = "v${env.BUILD_NUMBER}"
        CHART_VERSION       = "0.1.${env.BUILD_NUMBER}"

        // --- 6. CREDENTIALS ---
        HARBOR_CREDS_ID     = 'harbor-registry-creds'
        GIT_CREDS_ID        = 'gitlab-repository-creds'
        
        BUILD_HELM_DIR      = "helm_build_area"
    }

    stages {
        // --- STAGE 1: LẤY CODE ---
        stage('1. Checkout Code') {
            steps {
                script {
                    cleanWs()
                    // 1.1 Repo Infra (chứa Jenkinsfile và Dockerfile.base)
                    checkout scm
                    
                    // 1.2 Repo Frontend
                    dir('frontend-src') {
                        git url: "${GIT_URL_FRONTEND}", credentialsId: GIT_CREDS_ID, branch: GIT_BRANCH
                    }
                    // 1.3 Repo Backend
                    dir('backend-src') {
                        git url: "${GIT_URL_BACKEND}", credentialsId: GIT_CREDS_ID, branch: GIT_BRANCH
                    }
                }
            }
        }

        // --- STAGE 2: BUILD BASE IMAGE (CHỈ CHẠY KHI CÓ THAY ĐỔI) ---
        stage('2. Build PHP Base Image') {
            when {
                changeset "docker/Dockerfile.base"
            }
            steps {
                script {
                    echo "🛠️ Phát hiện thay đổi trong Dockerfile.base. Đang cập nhật Base Image..."
                    withCredentials([usernamePassword(credentialsId: HARBOR_CREDS_ID, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh 'echo $PASS | docker login ${HARBOR_DOMAIN} -u $USER --password-stdin'
                    }
                    sh "docker buildx create --use --name yorisoi-builder || docker buildx use yorisoi-builder"
                    sh """
                        docker buildx build --platform linux/amd64,linux/arm64 \
                        -f docker/Dockerfile.base \
                        -t ${IMAGE_PHP_BASE}:latest --push .
                    """
                }
            }
        }

        // --- STAGE 3: BUILD APPS ---
        stage('3. Build & Push Multi-Platform Images') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: HARBOR_CREDS_ID, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh 'echo $PASS | docker login ${HARBOR_DOMAIN} -u $USER --password-stdin'
                    }
                    sh "docker buildx use yorisoi-builder || docker buildx create --use --name yorisoi-builder"

                    // 3.1 Build Frontend (React)
                    dir('frontend-src') {
                        echo "🚀 Building Frontend..."
                        sh """
                            docker buildx build --platform linux/amd64,linux/arm64 \
                            --cache-from type=registry,ref=${IMAGE_FRONTEND}:cache \
                            --cache-to type=registry,ref=${IMAGE_FRONTEND}:cache,mode=max \
                            -t ${IMAGE_FRONTEND}:${BUILD_VERSION} --push .
                        """
                    }

                    // 3.2 Build Backend PHP (Kế thừa từ Base Image)
                    dir('backend-src') {
                        echo "🚀 Building Backend PHP-FPM..."
                        sh """
                            docker buildx build --platform linux/amd64,linux/arm64 \
                            --cache-from type=registry,ref=${IMAGE_BACKEND_PHP}:cache \
                            --cache-to type=registry,ref=${IMAGE_BACKEND_PHP}:cache,mode=max \
                            -t ${IMAGE_BACKEND_PHP}:${BUILD_VERSION} --push .
                        """
                    }

                    // 3.3 Build Backend Nginx (Sidecar)
                    // Lưu ý: Đảm bảo Dockerfile nginx nằm trong backend-src/docker/nginx/
                    dir('backend-src') { // Đứng ở gốc backend-src để thấy thư mục public/
                        echo "🚀 Building Backend Nginx Sidecar..."
                        sh """
                            docker buildx build --platform linux/amd64,linux/arm64 \
                            -f docker/nginx/Dockerfile \
                            --cache-from type=registry,ref=${IMAGE_BACKEND_NGINX}:cache \
                            --cache-to type=registry,ref=${IMAGE_BACKEND_NGINX}:cache,mode=max \
                            -t ${IMAGE_BACKEND_NGINX}:${BUILD_VERSION} --push .
                        """
                    }
                }
            }
        }

        // --- STAGE 4: HELM ---
        stage('4. Package & Push Helm Charts') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: HARBOR_CREDS_ID, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh 'echo $PASS | helm registry login ${HARBOR_DOMAIN} -u $USER --password-stdin'
                    }
                    
                    sh "rm -rf ${BUILD_HELM_DIR} && mkdir -p ${BUILD_HELM_DIR}"
                    sh "cp -r charts ${BUILD_HELM_DIR}/"

                    // 4.1 Base Chart
                    dir("${BUILD_HELM_DIR}/${CHART_BASE_PATH}") {
                        sh "helm package . --version ${CHART_VERSION} --app-version ${BUILD_VERSION}"
                        sh "helm push \$(ls *.tgz) oci://${HARBOR_DOMAIN}/${HARBOR_PROJECT}"
                    }

                    // 4.2 Local Chart (Wrapper)
                    dir("${BUILD_HELM_DIR}/${CHART_LOCAL_PATH}") {
                        // Cập nhật tag cho TẤT CẢ images (Frontend, Backend, Nginx-Backend)
                        sh "sed -i 's|tag: \".*\"|tag: \"${BUILD_VERSION}\"|g' values.yaml"
                        
                        sh "helm dependency update"
                        sh "helm package . --version ${CHART_VERSION} --app-version ${BUILD_VERSION}"
                        sh "helm push \$(ls *.tgz) oci://${HARBOR_DOMAIN}/${HARBOR_PROJECT}"
                    }
                }
            }
        }
    }

    post {
        success { echo "✅ HOÀN TẤT: Version ${BUILD_VERSION} đã sẵn sàng trên ArgoCD." }
        failure { echo "❌ THẤT BẠI: Vui lòng kiểm tra log buildx hoặc kết nối Harbor." }
    }
}
