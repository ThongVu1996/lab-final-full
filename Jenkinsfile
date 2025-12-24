pipeline {
    agent any

    environment {
        // --- 1. CẤU HÌNH HARBOR ---
        HARBOR_DOMAIN       = "harbor.local.thongdev.site"
        HARBOR_PROJECT      = "lab-final" 

        // --- 2. CẤU HÌNH HELM CHART ---
        // Đường dẫn đến thư mục chứa chart trong repo Infra
        CHART_BASE_PATH     = "charts/yorisoi-stack"
        CHART_LOCAL_PATH    = "charts/yorisoi-local"
        
        // --- 3. CẤU HÌNH GIT REPO (MULTI-REPO) ---
        // Đường dẫn Git của Frontend và Backend
        GIT_URL_FRONTEND    = "https://gitlab.local.thongdev.site/tonylab/frontend.git"
        GIT_URL_BACKEND     = "https://gitlab.local.thongdev.site/tonylab/backend.git"
        
        // Branch cần build (thường là main hoặc master)
        GIT_BRANCH          = "main"

        // --- 4. VERSIONING ---
        // Tag cho Docker Image: v1, v2... (Dựa trên Build Number)
        BUILD_VERSION       = "v${env.BUILD_NUMBER}"
        
        // Version cho Helm Chart: 0.1.x 
        CHART_VERSION       = "0.1.${env.BUILD_NUMBER}"

        // --- 5. CREDENTIALS ---
        // ID đăng nhập Harbor (Username/Password)
        HARBOR_CREDS_ID     = 'harbor-registry-creds'
        // ID đăng nhập GitLab (SSH Key hoặc Username/Password)
        GIT_CREDS_ID        = 'gitlab-repository-creds'
        
        // --- 6. THƯ MỤC LÀM VIỆC TẠM THỜI ---
        BUILD_HELM_DIR      = "helm_build_area"
    }

    stages {
        // --- Stage 1: Lấy Source Code từ 3 Repo ---
        stage('1. Checkout Code') {
            steps {
                script {
                    // Xóa sạch workspace trước khi bắt đầu
                    cleanWs()
                    
                    // 1.1 Checkout Repo Infra (Repo chứa Jenkinsfile này)
                    checkout scm
                    echo "✅ Đã lấy code Infra (Charts & Config)"

                    // 1.2 Checkout Repo Frontend
                    // Tạo thư mục riêng 'frontend-src' và lấy code vào đó
                    dir('frontend-src') {
                        git url: "${GIT_URL_FRONTEND}", credentialsId: GIT_CREDS_ID, branch: GIT_BRANCH
                    }
                    echo "✅ Đã lấy code Frontend vào thư mục frontend-src"

                    // 1.3 Checkout Repo Backend
                    // Tạo thư mục riêng 'backend-src' và lấy code vào đó
                    dir('backend-src') {
                        git url: "${GIT_URL_BACKEND}", credentialsId: GIT_CREDS_ID, branch: GIT_BRANCH
                    }
                    echo "✅ Đã lấy code Backend vào thư mục backend-src"
                }
            }
        }

        // --- Stage 2: Build & Push Docker Images ---
        stage('2. Build & Push Docker Images') {
            steps {
                script {
                    echo "🚀 Bắt đầu build Docker Images..."
                    
                    // Đăng nhập vào Harbor registry
                    withCredentials([usernamePassword(credentialsId: HARBOR_CREDS_ID, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                         sh 'echo $PASS | docker login ${HARBOR_DOMAIN} -u $USER --password-stdin'
                    }

                    // 2.1 Build Frontend
                    // Chuyển vào thư mục chứa code Frontend vừa checkout
                    dir('frontend-src') {
                        // Tạo builder ảo nếu chưa có (để hỗ trợ build đa nền tảng)
                        sh "docker buildx create --use || true"
                        
                        // Lệnh quan trọng: --platform linux/amd64
                        // Lưu ý: Khi dùng buildx, ta dùng --push ngay trong lệnh build
                        sh """
                            docker buildx build --platform linux/amd64,linux/arm64 \
                            -t ${HARBOR_DOMAIN}/${HARBOR_PROJECT}/frontend:${BUILD_VERSION} \
                            --push .
                        """
                    }
                    // 2.2 Build Backend
                    // Chuyển vào thư mục chứa code Backend vừa checkout
                    dir('backend-src') {
                         sh "docker buildx create --use || true"
                         
                         sh """
                            docker buildx build --platform linux/amd64,linux/arm64 \
                            -t ${HARBOR_DOMAIN}/${HARBOR_PROJECT}/backend:${BUILD_VERSION} \
                            --push .
                        """
                    }
                    echo "✅ Đã đẩy Docker Images version: ${BUILD_VERSION}"
                }
            }
        }

        // --- Stage 3: Đóng gói và đẩy Helm Charts ---
        stage('3. Package & Push Helm Charts') {
            steps {
                script {
                    // Đăng nhập vào Helm Registry (Harbor OCI)
                    withCredentials([usernamePassword(credentialsId: HARBOR_CREDS_ID, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                         sh 'echo $PASS | helm registry login ${HARBOR_DOMAIN} -u $USER --password-stdin'
                    }
                    
                    // Tạo thư mục build tạm và copy toàn bộ folder charts vào đó
                    // Việc này để giữ cấu trúc đường dẫn tương đối (../yorisoi-stack) cho lệnh dependency
                    sh "rm -rf ${BUILD_HELM_DIR} && mkdir -p ${BUILD_HELM_DIR}"
                    sh "cp -r charts ${BUILD_HELM_DIR}/"

                    // --- BƯỚC 3.1: XỬ LÝ BASE CHART (yorisoi-stack) ---
                    echo "📦 Đang đóng gói Base Chart..."
                    dir("${BUILD_HELM_DIR}/${CHART_BASE_PATH}") {
                        // Đóng gói chart thành file .tgz với version động
                        sh "helm package . --version ${CHART_VERSION} --app-version ${BUILD_VERSION}"
                        // Đẩy file .tgz lên Harbor
                        sh "helm push \$(ls *.tgz) oci://${HARBOR_DOMAIN}/${HARBOR_PROJECT}"
                    }

                    // --- BƯỚC 3.2: XỬ LÝ WRAPPER CHART (yorisoi-local) ---
                    echo "📦 Đang đóng gói Wrapper Chart..."
                    dir("${BUILD_HELM_DIR}/${CHART_LOCAL_PATH}") {
                        
                        // Cập nhật file values.yaml: Thay thế tag ảnh bằng version vừa build
                        // Tìm chuỗi 'tag: "..."' và đổi thành 'tag: "v1..."'
                        sh "sed -i 's|tag: \".*\"|tag: \"${BUILD_VERSION}\"|g' values.yaml"
                        
                        // Cập nhật dependencies: Tải Base Chart từ thư mục bên cạnh vào charts/
                        sh "helm dependency update"
                        
                        // Đóng gói chart Wrapper thành file .tgz
                        sh "helm package . --version ${CHART_VERSION} --app-version ${BUILD_VERSION}"
                        // Đẩy file .tgz lên Harbor
                        sh "helm push \$(ls *.tgz) oci://${HARBOR_DOMAIN}/${HARBOR_PROJECT}"
                    }

                    // Đăng xuất khỏi Helm Registry
                    sh "helm registry logout ${HARBOR_DOMAIN}"
                }
            }
        }
    }

    post {
        success {
            echo "✅✅✅ QUY TRÌNH HOÀN TẤT THÀNH CÔNG ✅✅✅"
            echo "ArgoCD sẽ tự động phát hiện Helm Chart version: ${CHART_VERSION}"
        }
        failure {
            echo "❌❌❌ CÓ LỖI XẢY RA ❌❌❌"
        }
    }
}
