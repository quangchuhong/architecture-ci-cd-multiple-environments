## CI CD ASP.NET Core

### 1. Kiểu triển khai phổ biến cho ASP.NET Core

Hiện đại nhất là dùng ASP.NET Core chạy trên Linux container, quy trình giống hệt Java/Python/Node:

  - Code ASP.NET Core (API/Website)
  - Build & test bằng dotnet CLI
  - dotnet publish → build artifact (self-contained hoặc framework-dependent)
  - Docker build → image ASP.NET Core
  - Trivy scan → push Nexus/ECR
  - GitOps/ArgoCD deploy lên EKS (dùng Linux container).
    
Nếu bạn còn ASP.NET “cổ điển” (Full .NET Framework) chỉ chạy được trên Windows, khi đó cần Windows agent + Windows container (phức tạp hơn). Ở đây mình tập trung ASP.NET Core (chạy Linux), phù hợp với EKS/Nexus hiện tại của bạn.

---

### 2. Cấu trúc ASP.NET Core project

Ví dụ:
```text
MyAspNetApp/
  MyAspNetApp.csproj
  Program.cs
  Startup.cs (hoặc minimal API)
  Controllers/
  Models/
  Services/
  Tests/
    MyAspNetApp.Tests.csproj
    UnitTests.cs
  Dockerfile
  Jenkinsfile

```

---

### 3. CI trên Jenkins – luồng chính

#### 3.1. Checkout

  - Jenkins lấy code từ Git (GitLab), branch develop / release/* / main.
    
  - Tính GIT_SHORT_SHA để dùng làm IMAGE_TAG.
    
#### 3.2. Build & test bằng dotnet CLI

Trong container có .NET SDK (ví dụ mcr.microsoft.com/dotnet/sdk:8.0):

  1. Restore packages:
```text
dotnet restore

```
  2. Build:
```text
dotnet build --configuration Release

```
3. Test:
```text
dotnet test --configuration Release --no-build
```
  - Nếu bất kỳ test nào fail → stage fail → dừng pipeline.

#### 3.3. Publish ứng dụng

Đóng gói app ASP.NET Core ra folder publish/:
```text
dotnet publish MyAspNetApp.csproj \
  --configuration Release \
  --output ./publish
```

Thư mục ./publish chứa:

  - MyAspNetApp.dll
  - Các file runtime cần thiết.

---

### 4. Docker build cho ASP.NET Core

Dockerfile kiểu 2-stage:
```text
# Stage 1: build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY . .
RUN dotnet restore
RUN dotnet publish MyAspNetApp.csproj -c Release -o /app/publish

# Stage 2: runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .

# Port app lắng nghe, ví dụ 8080
EXPOSE 8080

ENTRYPOINT ["dotnet", "MyAspNetApp.dll"]

```

Trong Jenkins (container docker:dind):
```text
docker build -t docker-internal.gitlabonlinecom.click/dev-backend/my-aspnet-app:${IMAGE_TAG} .

```
---

### 5. Trivy scan + push Nexus Docker

Y hệt các app khác:

Trivy scan:
```text
trivy image --exit-code 0 --severity LOW,MEDIUM \
  docker-internal.gitlabonlinecom.click/dev-backend/my-aspnet-app:${IMAGE_TAG}

trivy image --exit-code 0 --severity HIGH,CRITICAL \
  docker-internal.gitlabonlinecom.click/dev-backend/my-aspnet-app:${IMAGE_TAG}

```

Push Nexus Docker:
```text
docker login docker-internal.gitlabonlinecom.click -u <user> -p <pass>
docker push docker-internal.gitlabonlinecom.click/dev-backend/my-aspnet-app:${IMAGE_TAG}
docker logout docker-internal.gitlabonlinecom.click

```

<user>/<pass> là credentials Jenkins trỏ đến user trên Nexus có quyền push vào docker-hosted.

---

### 6. (Tuỳ chọn) GitOps + ArgoCD

Nếu bạn đã có:

  - GitOps repo: gitops-repo-deploy-argocd-aspnet, cấu trúc:

```text
gitops-repo/
  aspnet/
    test/values.yaml
    staging/values.yaml
    prod/values.yaml

```

Trong values.yaml:

```text
image:
  repository: docker-internal.gitlabonlinecom.click/dev-backend/my-aspnet-app
  tag: <IMAGE_TAG>

```

#### 6.1. Jenkins update GitOps

  - Test (develop):
    
```text
yq e '.image.repository = "docker-internal.gitlabonlinecom.click/dev-backend/my-aspnet-app"' -i values.yaml
yq e '.image.tag = "'${IMAGE_TAG}'"' -i values.yaml
git commit -am "Update my-aspnet-app image tag to ${IMAGE_TAG} for test" || echo "No changes"
git push origin develop

```
  - Staging (release/APP_VERSION) & Prod (main) như Java/Python:
    - Manual approval,
    - Cập nhật staging/values.yaml trên branch release/APP_VERSION,
    - git revert rollback nếu cần → ArgoCD sync lại.

---

### 7. Tóm tắt Jenkins CI cho ASP.NET Core

1. Checkout
   
2. dotnet restore
   
3. dotnet build -c Release
   
4. dotnet test -c Release --no-build
   
5. dotnet publish -c Release -o ./publish (nếu muốn dùng publish trong Docker)
   
6. Docker build → docker-internal.gitlabonlinecom.click/dev-backend/my-aspnet-app:${IMAGE_TAG}
   
7. Trivy scan
   
8. Push Nexus
   
9. (Nếu GitOps) Update GitOps test/staging/prod → ArgoCD deploy.

---

### 8. Jenkins pipeline CI cho ASP.NET Core

```bash
podTemplate(
  label: 'aspnet-core-pod',
  containers: [
    containerTemplate(
      name: 'jnlp',
      image: 'jenkins/inbound-agent:latest-jdk17',
      args: '${computer.jnlpmac} ${computer.name}',
      resourceRequestCpu: '200m',
      resourceRequestMemory: '256Mi',
      resourceLimitCpu: '500m',
      resourceLimitMemory: '512Mi'
    ),
    containerTemplate(
      name: 'dotnet',
      image: 'mcr.microsoft.com/dotnet/sdk:8.0',
      command: 'cat',
      ttyEnabled: true,
      resourceRequestCpu: '500m',
      resourceRequestMemory: '1Gi',
      resourceLimitCpu: '1000m',
      resourceLimitMemory: '2Gi'
    ),
    containerTemplate(
      name: 'docker',
      image: 'docker:24-dind',
      privileged: true,
      ttyEnabled: true
    ),
    containerTemplate(
      name: 'trivy',
      image: 'aquasec/trivy:0.69.3',
      command: 'cat',
      ttyEnabled: true
    ),
    containerTemplate(
      name: 'git',
      image: 'bitnami/git:latest',
      command: 'cat',
      ttyEnabled: true
    )
  ]
) {
  // ======= CONFIG =======
  def APP_NAME        = "aspnet-core-app"
  def CONFIGURATION   = "Release"

  // Docker / Registry (Nexus/ECR/…) 
  def REGISTRY_URL    = "your-registry.example.com"
  def REGISTRY_CRED   = "registry-credentials-id"
  def IMAGE_NAME      = "your-team/aspnet-core-app"
  def IMAGE_TAG       = ""
  def TARFILE_NAME    = ""

  // GitOps (ArgoCD)
  def GITOPS_REPO_URL       = "https://git.example.com/org/gitops-repo.git"
  def GITOPS_CREDENTIALS_ID = "gitops-https-token"
  def GITOPS_DEFAULT_BRANCH = "main"   // nhánh chính của gitops repo

  // Đường dẫn tới manifest/values để ArgoCD đọc
  def K8S_DEPLOY_FILE_PATH  = "apps/aspnet-core-app/deployment.yaml"   // nếu dùng plain manifest
  // hoặc dùng values.yaml của Helm: "envs/dev/values.yaml" (tùy bạn)

  def GIT_SHORT_SHA = ""

  node('aspnet-core-pod') {
    stage('Checkout') {
      echo '[Checkout] Lấy source từ SCM'
      checkout scm

      script {
        try {
          GIT_SHORT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        } catch (e) {
          GIT_SHORT_SHA = env.BUILD_NUMBER
        }
      }
    }

    stage('Restore & Build') {
      container('dotnet') {
        sh """
          dotnet --info
          dotnet restore
          dotnet build --configuration ${CONFIGURATION} --no-restore
        """
      }
    }

    stage('Test') {
      container('dotnet') {
        sh """
          dotnet test --configuration ${CONFIGURATION} --no-build --collect:"XPlat Code Coverage"
        """
      }
      // có thể thêm: junit '*/TestResults/**/*.xml'
    }

    stage('Publish') {
      container('dotnet') {
        sh """
          dotnet publish \
            --configuration ${CONFIGURATION} \
            --output ./publish \
            --no-build
          ls -lh publish/
        """
      }
    }

    stage('Docker Build (local)') {
      container('docker') {
        script {
          def branch = env.BRANCH_NAME ?: ''
          def envPrefix
          if (branch == 'develop') {
            envPrefix = 'dev'
          } else if (branch.startsWith('release/')) {
            envPrefix = 'stg'
          } else if (branch == 'main') {
            envPrefix = 'prod'
          } else {
            envPrefix = 'dev'
          }

          IMAGE_TAG    = "${envPrefix}-${GIT_SHORT_SHA}"
          TARFILE_NAME = "image-${APP_NAME}-${IMAGE_TAG}.tar"

          echo "[Docker] IMAGE_TAG = ${IMAGE_TAG}, TARFILE_NAME = ${TARFILE_NAME}"
        }

        withCredentials([usernamePassword(
          credentialsId: REGISTRY_CRED,
          usernameVariable: 'REG_USER',
          passwordVariable: 'REG_PASS'
        )]) {
          sh """
            echo "${REG_PASS}" | docker login ${REGISTRY_URL} -u "${REG_USER}" --password-stdin

            # Build image từ Dockerfile (multi-stage Dockerfile cho ASP.NET Core)
            docker build -t ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG} .

            # Lưu image ra file TAR (dùng cho scan/offline lưu trữ nếu cần)
            docker save ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG} -o ${TARFILE_NAME}
            ls -lh ${TARFILE_NAME}
          """
        }
      }
    }

    stage('Trivy Scan Image (tar)') {
      container('trivy') {
        sh """
          ls -lh
          echo "[Trivy] Scan file tar ${TARFILE_NAME}"

          # Scan LOW + MEDIUM (không fail pipeline)
          trivy image --input ${TARFILE_NAME} \
            --exit-code 0 \
            --severity LOW,MEDIUM

          # Scan HIGH + CRITICAL (có thể fail pipeline, set exit-code 1 nếu muốn fail)
          trivy image --input ${TARFILE_NAME} \
            --exit-code 0 \
            --severity HIGH,CRITICAL
        """
      }
    }

    stage('Push Image to Registry') {
      container('docker') {
        withCredentials([usernamePassword(
          credentialsId: REGISTRY_CRED,
          usernameVariable: 'REG_USER',
          passwordVariable: 'REG_PASS'
        )]) {
          sh """
            echo "${REG_PASS}" | docker login ${REGISTRY_URL} -u "${REG_USER}" --password-stdin

            echo "[Registry] Push image ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
            docker push ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Update GitOps Repo (ArgoCD)') {
      container('git') {
        script {
          // Map branch -> envPath + git branch
          def branch = env.BRANCH_NAME ?: ''
          def envPath
          def gitTargetBranch

          if (branch == 'develop') {
            envPath       = 'dev'
            gitTargetBranch = 'develop'
          } else if (branch.startsWith('release/')) {
            envPath       = 'staging'
            gitTargetBranch = 'staging'
          } else if (branch == 'main') {
            envPath       = 'prod'
            gitTargetBranch = 'main'
          } else {
            envPath       = 'dev'
            gitTargetBranch = 'develop'
          }

          echo "[GitOps] Update env=${envPath}, branch=${gitTargetBranch}, image tag=${IMAGE_TAG}"

          withCredentials([usernamePassword(
            credentialsId: GITOPS_CREDENTIALS_ID,
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_PASS'
          )]) {
            sh """
              rm -rf gitops-work
              mkdir -p gitops-work
              cd gitops-work

              # clone với user + token
              git config --global credential.helper store
              GITOPS_URL_WITH_AUTH=${GITOPS_REPO_URL.replace('https://', "https://${GIT_USER}:${GIT_PASS}@")}
              git clone "\${GITOPS_URL_WITH_AUTH}" .
              git checkout ${gitTargetBranch} || git checkout -b ${gitTargetBranch}

              # Ví dụ: nếu dùng plain manifest yaml
              # Giả sử file nằm ở: ${envPath}/${K8S_DEPLOY_FILE_PATH}
              DEPLOY_FILE="${envPath}/${K8S_DEPLOY_FILE_PATH}"
              echo "[GitOps] Update file: \${DEPLOY_FILE}"

              # Cài yq nếu chưa có (Debian/Ubuntu base)
              if ! command -v yq >/dev/null 2>&1; then
                apt-get update -y || true
                apt-get install -y wget || true
                wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
                chmod +x /usr/local/bin/yq
              fi

              # Cập nhật trường image trong manifest (đường key tuỳ cấu trúc file)
              # Ví dụ manifest có: spec.template.spec.containers[0].image
              yq e '.spec.template.spec.containers[0].image = "${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}"' -i "\${DEPLOY_FILE}"

              git config user.email "jenkins-bot@example.com"
              git config user.name  "Jenkins Bot"

              git status
              git commit -am "Update ${APP_NAME} image to ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG} (${envPath})" || echo "No changes to commit"
              git push origin ${gitTargetBranch}
            """
          }
        }
      }
    }
  }
}

```
