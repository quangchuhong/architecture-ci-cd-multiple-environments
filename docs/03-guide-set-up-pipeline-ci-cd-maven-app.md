## Tổng quan workflow pipeline

Pipeline chạy trên Jenkins K8s agent (maven-pod) với 4 container chính:

        - jnlp: agent Jenkins
        - maven: build/test/sonar
        - docker: build image, save TAR, push Nexus
        - trivy: scan image từ file TAR
        
Các hệ thống liên quan:

        - Nexus Docker internal: docker-internal.gitlabonlinecom.click/dev-backend/...
        - Nexus RAW: lưu file .tar image
        - SonarQube: scan source + Quality Gate
        - GitLab gitops-repo: repo GitOps ArgoCD (thư mục test/staging/prod)
        - ArgoCD: tự deploy theo GitOps repo cho Test & Staging


#### workflow pipeline


<img width="1070" height="307" alt="image" src="https://github.com/user-attachments/assets/2f0378b6-9b8c-4b4c-832d-0d33cee1e275" />


```text
        ┌────────────┐
        │   Dev      │
        └─────┬──────┘
              │
              │ git push (develop / release/*)
              ▼
        ┌────────────┐
        │  GitLab    │
        └─────┬──────┘
              │ Webhook
              ▼
        ┌─────────────────────────┐
        │ Jenkins (maven-pod)     │
        │  - jnlp, maven, docker  │
        │  - trivy                │
        └────────┬────────────────┘
                 │
   ┌─────────────┼──────────────────────────────────────┐
   │             │                                      │
   ▼             ▼                                      ▼
[Maven Build] [SonarQube]                        [Docker Build]
   │             │                                 + save TAR
   │             │                                 + upload TAR RAW
   │             │                                     │
   ▼             │                                     ▼
[Test/Unit]   [Quality Gate]                    [Trivy scan TAR]
   │             │                                     │
   └──────OK─────┴─────────────────────────────────────┘
                 │
                 ▼
        [Push image lên Nexus Docker]
                 │
                 ▼
        [Update GitOps Test → values.yaml]
                 │
                 ▼
           ┌─────────┐
           │ ArgoCD  │
           └────┬────┘
                │ sync
                ▼
         EKS app-test (Test)

                 │
                 ▼
     [Manual approval Promote Staging]
                 │ (Approve)
                 ▼
 [Create/Use release/APP_VERSION branch GitOps]
 [Update GitOps Staging → staging/values.yaml]
                 │
                 ▼
           ┌─────────┐
           │ ArgoCD  │
           └────┬────┘
                │ sync
                ▼
         EKS app-stg (Staging)

                 │
                 ▼
   [Manual approval Rollback Staging (nếu lỗi)]
                 │ (Approve)
                 ▼
     [Git revert HEAD trên release/APP_VERSION]
                 │
                 ▼
           ┌─────────┐
           │ ArgoCD  │
           └────┬────┘
                │ sync
                ▼
     EKS app-stg rollback về image trước

```

## 2. Logic chi tiết các bước trong pipeline

### 2.1. CI: Build, Test, SonarQube

1. **Checkout & chuẩn bị**
   - Lấy source từ Git (branch: `develop` / `release/*` / `main`).
   - Tính `GIT_SHORT_SHA` (short commit hash) để dùng làm `IMAGE_TAG` và tên file TAR.

2. **Maven Build**
   - Biên dịch mã nguồn Java:
     - `mvn clean compile`.

3. **Maven Test**
   - (Hiện đang để skip; có thể bật lại)
   - Chạy unit test / integration test cho project.

4. **Scan Source (SonarQube)**
   - Chạy SonarQube Maven plugin:
     - `sonar.projectKey = shopping-cart`
     - `sonar.projectVersion = shopping-cart-${GIT_SHORT_SHA}`
   - Gửi kết quả phân tích source lên SonarQube server.

5. **Quality Gate**
   - Jenkins gọi `waitForQualityGate` để:
     - Đợi SonarQube tính xong Quality Gate.
     - Nếu result != `OK` → fail pipeline, dừng toàn bộ, **không build/push image**.

---

### 2.2. Build artifact & Docker image, lưu TAR, scan Trivy

6. **Build Source Code**
   - Đóng gói ứng dụng:
     - `mvn package -DskipTests=true`
   - Tạo JAR/WAR trong `target/`.

7. **Docker Build & Upload File Image TAR lên Nexus RAW**
   - Xác định `IMAGE_TAG` (dựa trên `GIT_SHORT_SHA`, có thể thêm prefix theo môi trường).
   - Trong container `docker`:
     - Build image:
       - `docker-internal.gitlabonlinecom.click/dev-backend/shopping-cart:${IMAGE_TAG}`
     - Lưu image ra file TAR:
       - `docker save ... -o image-shopping-cart-${IMAGE_TAG}.tar`
     - Upload file `.tar` lên Nexus RAW:
       - Repo: `raw-artifacts`
       - Path: `dev-backend/docker-tar/image-shopping-cart-${IMAGE_TAG}.tar`

8. **Trivy Scan Image (TAR từ Nexus RAW)**
   - Trong container `trivy`:
     - Dùng `curl` tải TAR từ Nexus RAW về workspace.
     - Scan bằng Trivy:
       - Lần 1: `LOW,MEDIUM` → chỉ log, không fail pipeline.
       - Lần 2: `HIGH,CRITICAL` → hiện vẫn `exit-code 0` (cảnh báo; có thể đổi thành `1` để chặn build nếu muốn).

9. **Push Docker image lên Nexus Docker internal**
   - Trong container `docker`:
     - `docker login docker-internal.gitlabonlinecom.click` bằng user nội bộ.
     - `docker push docker-internal.gitlabonlinecom.click/dev-backend/shopping-cart:${IMAGE_TAG}`
   - Image này sẽ được dùng cho Test/Staging/Prod thông qua GitOps.

---

### 2.3. GitOps & ArgoCD: Deploy Test

10. **Update GitOps Repo cho Test**
    - Clone GitOps repo (GitLab) về workspace.
    - Xác định env & branch GitOps:
      - `develop` → envPath = `test/`, gitTargetBranch = `develop`
      - `release/*` → envPath = `staging/`, gitTargetBranch = `staging`
      - `main` → envPath = `prod/`, gitTargetBranch = `main`
    - Vào thư mục `${envPath}`, dùng `yq`:
      - `.image.repository = "docker-internal.gitlabonlinecom.click/dev-backend/shopping-cart"`
      - `.image.tag = "${IMAGE_TAG}"`
    - Commit & push:
      - `"Update shopping-cart image tag to ${IMAGE_TAG} for ${envPath}"`
    - ArgoCD (Application Test) trỏ vào branch + path tương ứng:
      - Phát hiện thay đổi → sync → deploy image mới lên namespace Test trên EKS.

---

### 2.4. Promote lên Staging (cùng image đã qua Test)

11. **Manual Approval to Promote to Staging**
    - Dừng pipeline và hiển thị câu hỏi:
      - “Promote image ${IMAGE_TAG} to STAGING?”
    - Người có quyền (DevOps/QA/Lead) bấm **Approve** nếu Test OK.

12. **Create/Use Release Branch & Deploy Staging**
    - Clone GitOps repo sang thư mục riêng (vd. `app-release/`).
    - Xác định `RELEASE_BRANCH = release/${APP_VERSION}` (APP_VERSION là version ứng dụng, vd. 1.0.0).
    - Kiểm tra trên origin:
      - Nếu branch `release/${APP_VERSION}` **đã tồn tại**:
        - `git checkout release/${APP_VERSION}` → `git pull`.
      - Nếu **chưa tồn tại**:
        - `git checkout develop` → `git pull`.
        - `git checkout -b release/${APP_VERSION}` → `git push origin release/${APP_VERSION}`.
    - `cd staging/` trong GitOps repo:
      - Dùng `yq`:
        - `.image.repository = "docker-internal.gitlabonlinecom.click/dev-backend/shopping-cart"`
        - `.image.tag = "${IMAGE_TAG}"`
        - (option) `.appVersion = "${APP_VERSION}"` nếu muốn lưu version business.
    - Commit & push lên `release/${APP_VERSION}`:
      - `"Update shopping-cart image tag to ${IMAGE_TAG} for staging"`.
    - ArgoCD (Application Staging):
      - `targetRevision = release/${APP_VERSION}`, `path = staging/`.
      - Sync deploy image `${IMAGE_TAG}` vào namespace Staging trên EKS.

---

### 2.5. Rollback Staging (khi deploy staging bị lỗi)

13. **Manual Approval to Rollback Staging**
    - Nếu phát hiện lỗi trên Staging:
      - Pipeline dừng tại stage rollback và hỏi:
        - “Rollback STAGING từ image ${IMAGE_TAG}? (ArgoCD sẽ quay về version trước đó)”
      - Người vận hành bấm “Rollback” để kích hoạt rollback.

14. **Rollback GitOps Staging (Git Revert)**
    - Clone GitOps repo sang thư mục `gitops-rollback/`.
    - `git checkout release/${APP_VERSION}` → `git pull`.
    - Thiết lập `user.email` / `user.name` = Jenkins Bot.
    - Xem commit mới nhất:
      - `git log -1 --oneline` (thường là commit update `IMAGE_TAG` cho Staging).
    - Thực hiện:
      - `git revert --no-edit HEAD`:
        - Tạo commit mới đảo ngược commit trước (khôi phục `values.yaml` về trạng thái trước deploy lỗi).
    - `git push origin release/${APP_VERSION}`.
    - ArgoCD Staging:
      - Thấy commit revert mới → tự sync → rollback Deployment về image tag cũ (phiên bản chạy ổn trước đó).

---

## 3. Tóm tắt luồng chính

- **CI/Test**:
  - Checkout → Maven build/test → SonarQube + Quality Gate →  
  → Build JAR → Build Docker image → Save TAR & upload Nexus RAW → Trivy scan TAR → Push image lên Nexus Docker → Update GitOps `test/values.yaml` → ArgoCD deploy Test.

- **Promote Staging**:
  - Manual approval → Create/Use `release/APP_VERSION` branch trong GitOps repo → Update `staging/values.yaml` với image `${IMAGE_TAG}` → ArgoCD deploy Staging.

- **Rollback Staging**:
  - Manual approval rollback → `git revert HEAD` trên `release/APP_VERSION` (Staging) → ArgoCD sync → rollback về image tag trước đó.

