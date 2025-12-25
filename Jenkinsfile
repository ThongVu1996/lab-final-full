// ===============================
// 1. KHAI BÁO BIẾN GLOBAL (QUAN TRỌNG)
// ===============================
// Đặt ở ngoài pipeline để trạng thái được lưu xuyên suốt và cập nhật tức thì
def isAwsDeploy = false
def isBaseChanged = false

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        // ===============================
        // REGISTRY
        // ===============================
        HARBOR_DOMAIN  = "harbor.local.thongdev.site"
        HARBOR_PROJECT = "lab-final"
        AWS_REGION     = "ap-southeast-1"
        ECR_DOMAIN     = "605695176329.dkr.ecr.ap-southeast-1.amazonaws.com"

        // ===============================
        // IMAGES
        // ===============================
        IMG_BASE_HARBOR  = "${HARBOR_DOMAIN}/${HARBOR_PROJECT}/php-8.4-base"
        IMG_FE_HARBOR    = "${HARBOR_DOMAIN}/${HARBOR_PROJECT}/frontend"
        IMG_BE_HARBOR    = "${HARBOR_DOMAIN}/${HARBOR_PROJECT}/backend"
        IMG_NGINX_HARBOR = "${HARBOR_DOMAIN}/${HARBOR_PROJECT}/nginx-backend"

        IMG_BASE_ECR  = "${ECR_DOMAIN}/${HARBOR_PROJECT}/php-8.4-base"
        IMG_FE_ECR    = "${ECR_DOMAIN}/${HARBOR_PROJECT}/frontend"
        IMG_BE_ECR    = "${ECR_DOMAIN}/${HARBOR_PROJECT}/backend"
        IMG_NGINX_ECR = "${ECR_DOMAIN}/${HARBOR_PROJECT}/nginx-backend"

        // ===============================
        // GIT
        // ===============================
        GIT_FE_URL    = "https://gitlab.local.thongdev.site/tonylab/frontend.git"
        GIT_BE_URL    = "https://gitlab.local.thongdev.site/tonylab/backend.git"
        GIT_INFRA_URL = "https://github.com/ThongVu1996/lab-final.git"

        // ===============================
        // VERSION
        // ===============================
        BUILD_VERSION = "v${BUILD_NUMBER}"
        CHART_VERSION = "0.1.${BUILD_NUMBER}"

        // ===============================
        // CREDS
        // ===============================
        HARBOR_CREDS = "harbor-registry-creds"
        AWS_CREDS    = "aws-ecr-creds"
        GITLAB_CREDS = "gitlab-repository-creds"
        GITHUB_CREDS = "github-token-creds"

        BUILDX_BUILDER = "yorisoi-builder"
        HELM_WORKDIR   = "helm_build_area"
    }

    stages {

        stage('01. Checkout') {
            steps {
                cleanWs()
                checkout scm

                dir('frontend-src') {
                    git url: GIT_FE_URL, credentialsId: GITLAB_CREDS, branch: 'main'
                }
                dir('backend-src') {
                    git url: GIT_BE_URL, credentialsId: GITLAB_CREDS, branch: 'main'
                }
                dir('github-infra') {
                    git url: GIT_INFRA_URL, credentialsId: GITHUB_CREDS, branch: 'main'
                }
            }
        }

        stage('02. Login Registry & Buildx') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(credentialsId: HARBOR_CREDS, usernameVariable: 'H_USER', passwordVariable: 'H_PASS')
                    ]) {
                        sh 'echo $H_PASS | docker login ${HARBOR_DOMAIN} -u $H_USER --password-stdin'
                    }

                    withCredentials([
                        usernamePassword(credentialsId: AWS_CREDS, usernameVariable: 'AWS_KEY', passwordVariable: 'AWS_SECRET')
                    ]) {
                        withEnv([
                            "AWS_ACCESS_KEY_ID=${AWS_KEY}",
                            "AWS_SECRET_ACCESS_KEY=${AWS_SECRET}",
                            "AWS_DEFAULT_REGION=${AWS_REGION}"
                        ]) {
                            sh 'aws ecr get-login-password | docker login --username AWS --password-stdin ${ECR_DOMAIN}'
                        }
                    }

                    sh """
                      docker buildx inspect ${BUILDX_BUILDER} >/dev/null 2>&1 ||
                      docker buildx create --use --name ${BUILDX_BUILDER}
                    """
                }
            }
        }

        stage('03. Build Base (Local)') {
            when { changeset "docker/Dockerfile.base" }
            steps {
                script {
                    echo "⚠️ DETECTED: Dockerfile.base has changed!"
                    // Cập nhật biến Global
                    isBaseChanged = true 
                    
                    sh """
                      docker buildx build --platform linux/amd64,linux/arm64 \
                      -f docker/Dockerfile.base \
                      -t ${IMG_BASE_HARBOR}:latest \
                      --push .
                    """
                }
            }
        }

        stage('04. Build Apps (Local)') {
            steps {
                script {
                    dir('frontend-src') {
                        sh "docker buildx build --platform linux/amd64,linux/arm64 -t ${IMG_FE_HARBOR}:${BUILD_VERSION} --push ."
                    }

                    dir('backend-src') {
                        sh """
                          docker buildx build --platform linux/amd64,linux/arm64 \
                          --build-arg BASE_IMAGE_URL=${IMG_BASE_HARBOR}:latest \
                          -t ${IMG_BE_HARBOR}:${BUILD_VERSION} \
                          --push .
                        """
                        sh "docker buildx build --platform linux/amd64,linux/arm64 -f docker/nginx/Dockerfile -t ${IMG_NGINX_HARBOR}:${BUILD_VERSION} --push ."
                    }
                }
            }
        }

        stage('05. Helm Package (Local)') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(credentialsId: HARBOR_CREDS, usernameVariable: 'H_USER', passwordVariable: 'H_PASS')
                    ]) {
                        sh 'echo $H_PASS | helm registry login ${HARBOR_DOMAIN} -u $H_USER --password-stdin'
                    }

                    sh """
                      rm -rf ${HELM_WORKDIR}
                      mkdir -p ${HELM_WORKDIR}
                      cp -r charts ${HELM_WORKDIR}/
                    """

                    dir("${HELM_WORKDIR}/charts/yorisoi-local") {
                        sh "sed -i 's|tag: \".*\"|tag: \"${BUILD_VERSION}\"|' values.yaml"
                        sh "helm dependency update"
                        sh "helm package . --version ${CHART_VERSION} --app-version ${BUILD_VERSION}"
                        sh "helm push *.tgz oci://${HARBOR_DOMAIN}/${HARBOR_PROJECT}"
                    }
                }
            }
        }

        // ===============================
        // APPROVAL (INPUT CHUẨN)
        // ===============================
        stage('06. Approval AWS') {
            steps {
                script {
                    echo "WAITING FOR AWS APPROVAL..."

                    input(
                        message: "Deploy ${BUILD_VERSION} to AWS?",
                        ok: "Deploy"
                    )

                    // Cập nhật biến Global (Boolean) - Chắc chắn hoạt động
                    isAwsDeploy = true

                    echo "✅ USER APPROVED. isAwsDeploy = ${isAwsDeploy}"
                }
            }
        }

        // stage('07. Build Base (AWS)') {
        //     when {
        //         allOf {
        //             // Kiểm tra biến Global
        //             expression { return isAwsDeploy }
        //             expression { return isBaseChanged }
        //         }
        //     }
        //     steps {
        //         sh """
        //           docker buildx build --platform linux/amd64,linux/arm64 \
        //           -f docker/Dockerfile.base \
        //           -t ${IMG_BASE_ECR}:latest \
        //           --push .
        //         """
        //     }
        // }
        //
        // stage('08. Build Apps (AWS)') {
        //     // Kiểm tra biến Global
        //     when { expression { return isAwsDeploy } }
        //     steps {
        //         script {
        //             dir('frontend-src') {
        //                 sh "docker buildx build --platform linux/amd64,linux/arm64 -t ${IMG_FE_ECR}:${BUILD_VERSION} -t ${IMG_FE_ECR}:latest --push ."
        //             }
        //             dir('backend-src') {
        //                 sh """
        //                   docker buildx build --platform linux/amd64,linux/arm64 \
        //                   --build-arg BASE_IMAGE_URL=${IMG_BASE_ECR}:latest \
        //                   -t ${IMG_BE_ECR}:${BUILD_VERSION} -t ${IMG_BE_ECR}:latest \
        //                   --push .
        //                 """
        //                 sh "docker buildx build --platform linux/amd64,linux/arm64 -f docker/nginx/Dockerfile -t ${IMG_NGINX_ECR}:${BUILD_VERSION} -t ${IMG_NGINX_ECR}:latest --push ."
        //             }
        //         }
        //     }
        // }
          
        stage('07. Push Base (Skopeo)') {
            when {
                allOf {
                    // Kiểm tra biến Global
                    expression { return isAwsDeploy }
                    expression { return isBaseChanged }
                }
            }
            steps {
                script {
                    echo "🚀 [Skopeo] Promoting Base Image from Harbor to ECR..."
                    
                    // --all: Copy cả AMD64 và ARM64 (Multi-arch)
                    // docker://: Giao thức bắt buộc của Skopeo
                    sh "skopeo copy --all docker://${IMG_BASE_HARBOR}:latest docker://${IMG_BASE_ECR}:latest"
                }
            }
        }

        stage('08. Push Apps (Skopeo)') {
            // Kiểm tra biến Global
            when { expression { return isAwsDeploy } }
            steps {
                script {
                    echo "🚀 [Skopeo] Promoting App Images from Harbor to ECR..."

                    // ==============================
                    // 1. FRONTEND
                    // ==============================
                    // Copy từ Harbor -> ECR (Tag Version)
                    sh "skopeo copy --all docker://${IMG_FE_HARBOR}:${BUILD_VERSION} docker://${IMG_FE_ECR}:${BUILD_VERSION}"
                    // Copy nội bộ ECR -> ECR (Để tạo tag Latest - cực nhanh)
                    sh "skopeo copy --all docker://${IMG_FE_ECR}:${BUILD_VERSION} docker://${IMG_FE_ECR}:latest"

                    // ==============================
                    // 2. BACKEND
                    // ==============================
                    sh "skopeo copy --all docker://${IMG_BE_HARBOR}:${BUILD_VERSION} docker://${IMG_BE_ECR}:${BUILD_VERSION}"
                    sh "skopeo copy --all docker://${IMG_BE_ECR}:${BUILD_VERSION} docker://${IMG_BE_ECR}:latest"

                    // ==============================
                    // 3. NGINX
                    // ==============================
                    sh "skopeo copy --all docker://${IMG_NGINX_HARBOR}:${BUILD_VERSION} docker://${IMG_NGINX_ECR}:${BUILD_VERSION}"
                    sh "skopeo copy --all docker://${IMG_NGINX_ECR}:${BUILD_VERSION} docker://${IMG_NGINX_ECR}:latest"
                    
                    echo "✅ All Images promoted to AWS ECR successfully!"
                }
            }
        }
        stage('09. GitOps (AWS)') {
            // Kiểm tra biến Global
            when { expression { return isAwsDeploy } }
            steps {
                script {
                    dir('github-infra/environments/aws') {
                        sh "sed -i 's|tag: \".*\"|tag: \"${BUILD_VERSION}\"|' values.yaml"

                        withCredentials([
                            usernamePassword(credentialsId: GITHUB_CREDS, usernameVariable: 'U', passwordVariable: 'P')
                        ]) {
                            sh """
                              git add values.yaml
                              git commit -m 'chore(aws): deploy ${BUILD_VERSION} [skip ci]'
                              git push https://${U}:${P}@github.com/ThongVu1996/lab-final.git main
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                if (isAwsDeploy) {
                    echo "🎉 SUCCESS: Local + AWS deployment completed."
                } else {
                    echo "✅ SUCCESS: Local deployment only."
                }
            }
        }

        failure {
            script {
                if (isAwsDeploy) {
                    echo "❌ FAILURE: Error during AWS deployment."
                } else {
                    echo "❌ FAILURE: Error during Local deployment."
                }
            }
        }

        aborted {
            echo "⛔ ABORTED: Pipeline was cancelled."
        }

        always {
            script {
                currentBuild.description = "AWS=${isAwsDeploy} | ${BUILD_VERSION}"
            }
        }
    }
}
