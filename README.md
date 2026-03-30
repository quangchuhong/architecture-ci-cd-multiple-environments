# Kiến trúc CI/CD 3 môi trường (Test / Staging / Prod) với 2 hạ tầng DevOps (Test / Prod)

## 1. Mục tiêu & phạm vi

Thiết kế hệ thống CI/CD dùng chung cho **tất cả loại ứng dụng**:

- Ngôn ngữ / nền tảng: **Maven/Java, Python, .NET/ASP.NET, Shell, AWS Lambda, service Linux/Windows, DB job, fintech/banking apps…**
- DevOps Tools trên **EKS + AWS**:
  - **GitLab** (chung): quản lý code, branch, MR, quyền phòng ban.
  - **Jenkins**: 
    - Cụm **DevOps-Test**: build CI cho Test & Staging.
    - Cụm **DevOps-Prod**: Jenkins riêng cho các project Prod.
  - **SonarQube, Trivy/Black Duck, Nexus** (artifact, image, S3) → **chỉ dùng ở môi trường Test**.
  - **EKS**: cluster chạy app (namespace test/stg/prod).
  - **ECR / Nexus Docker**: registry container (có thể dùng ECR là nguồn chính).
  - **S3**: lưu artifact phụ, report, backup.

Yêu cầu:

- a. Hạ tầng DevOps tools tách **2 môi trường**: **DevOps-Test** và **DevOps-Prod**.
- b. GitLab dùng **chung**, phân môi trường/phòng ban bằng branch, group, quyền.
- c. Jenkins:
  - DevOps-Test Jenkins: build cho **Test + Staging**.
  - DevOps-Prod Jenkins: build riêng cho **Prod**.
- d. SonarQube/Trivy/BlackDuck/Nexus chỉ phục vụ **Test CI**, Staging dùng **CD qua ArgoCD**.
- e. Khi Staging OK → MR lên branch Prod → Jenkins/ArgoCD Prod deploy.
- f. Cấu trúc project: **tên-phòng-ban / tên-môi-trường / tên-dự-án**.
- g. Phân quyền user theo **phòng ban, môi trường, dự án** (Dev, App, Test, QA, DB, Sec, DevOps).
- h. Kiến trúc đủ generic cho đa dạng tech stack.

---

### Toàn cảnh hạ tầng DevOps + App

                ┌──────────────────────────────────────┐
                │             AWS Cloud                │
                └──────────────────────────────────────┘

      ┌───────────────────────────┐        ┌───────────────────────────┐
      │    EKS-devops-test        │        │    EKS-devops-prod        │
      │ (hạ tầng DevOps cho CI)   │        │ (hạ tầng DevOps cho Prod) │
      ├───────────────────────────┤        ├───────────────────────────┤
      │ Namespace gitlab          │        │ Namespace jenkins-prod    │
      │  ┌─────────────────────┐ │        │  ┌─────────────────────┐  │
      │  │      GitLab         │◄┼────────┼──┤  Jenkins Prod        │  │
      │  └─────────────────────┘ │        │  └─────────────────────┘  │
      │ Namespace jenkins-test    │        └───────────────────────────┘
      │  ┌─────────────────────┐ │
      │  │  Jenkins Test/Stg   │ │
      │  └─────────────────────┘ │
      │ Namespace sonarqube      │
      │  ┌─────────────────────┐ │
      │  │    SonarQube        │ │
      │  └─────────────────────┘ │
      │ Namespace nexus          │
      │  ┌─────────────────────┐ │
      │  │      Nexus          │ │
      │  └─────────────────────┘ │
      │ Namespace security-scan  │
      │  ┌───────────┐ ┌──────┐ │
      │  │  Trivy    │ │BDuck │ │
      │  └───────────┘ └──────┘ │
      └───────────────────────────┘

                     │ CI/CD
                     ▼

      ┌────────────────────────────────────────────────────┐
      │                    EKS-app                        │
      │           (cluster chạy ứng dụng)                 │
      ├────────────────────────────────────────────────────┤
      │ Namespace app-test   ──> Env Test                 │
      │ Namespace app-stg    ──> Env Staging              │
      │ Namespace app-prod   ──> Env Prod                 │
      └────────────────────────────────────────────────────┘

                 ┌───────────────┐     ┌───────────────┐
                 │     ECR       │     │      S3       │
                 └───────────────┘     └───────────────┘
                 (image registry)      (artifact/logs)



### Luồng CI/CD theo nhánh

             ┌────────────┐
Dev pushes → │   GitLab   │
             └─────┬──────┘
       Webhook      │
                    │
        ┌───────────▼─────────────┐
        │   Jenkins Test/Stg      │ (EKS-devops-test)
        └─────────┬───────────────┘
                  │
                  │ Build/Test/Scan/Push
                  │
                  ▼
        ┌────────────────────────────┐
        │  SonarQube / Trivy / Nexus│
        │        + ECR / S3         │
        └────────────────────────────┘
                  │
                  │ Deploy
                  │
        ┌─────────▼───────────┐
        │       EKS-app       │
        │  app-test / app-stg │
        └─────────┬───────────┘
                  │
   Staging OK     │ MR
   ───────────────┘
             ┌────────────┐
             │   GitLab   │
             │  branch    │
             │   main     │
             └─────┬──────┘
                   │ Webhook
                   ▼
        ┌──────────────────────┐
        │   Jenkins Prod       │ (EKS-devops-prod)
        └─────────┬────────────┘
                  │
                  │ Deploy Prod (ArgoCD Prod / Helm)
                  ▼
            ┌─────────────┐
            │  EKS-app    │
            │  app-prod   │
            └─────────────┘


## 2. Phân tách hạ tầng DevOps

### 2.1. Cụm DevOps-Test (`EKS-devops-test`)

Namespace gợi ý:

- `gitlab`        – GitLab (chung cho tất cả env).
- `jenkins-test`  – Jenkins Test/Staging.
- `sonarqube`     – SonarQube + PostgreSQL.
- `nexus`         – Nexus Repository (Maven, NPM, Docker test).
- `security-scan` – Trivy/Black Duck (nếu deploy dạng service riêng).
- (optional) `argocd-test` – ArgoCD cho môi trường Test (nếu muốn GitOps test).

Dùng cho:

- CI & quality gate cho **Test/Staging**.
- CD cho **Test** (Jenkins/ArgoCD).

### 2.2. Cụm DevOps-Prod (`EKS-devops-prod`)

Namespace:

- `jenkins-prod` – Jenkins chỉ phục vụ project Prod.
- (optional) `argocd-prod` – ArgoCD Prod.

Không chạy Sonar/Trivy/Nexus tại đây (dùng artifact đã qua kiểm tra ở DevOps-Test).

---

## 3. EKS ứng dụng & môi trường runtime

Có 2 lựa chọn:

### Lựa chọn tiết kiệm (đề xuất)

- 1 cluster `EKS-app` với namespace:
  - `app-test`
  - `app-stg`
  - `app-prod`
- NetworkPolicy, ResourceQuota, IRSA tách riêng theo namespace.

### Lựa chọn tách biệt cao (nhiều tiền hơn)

- `EKS-app-test`  – Test.
- `EKS-app-stg`   – Staging.
- `EKS-app-prod`  – Prod (cách ly mạnh, dedicated).

Kiến trúc dưới đây vẫn đúng cho cả 2 lựa chọn (khác nhau ở kube context/namespace/cluster).

---

## 4. GitLab – cấu trúc project & branch

### 4.1. Cấu trúc group/project

Theo nguyên tắc: **tên-phòng-ban / tên-3-môi-trường / tên-dự-án**

Ví dụ:

- `dev-backend/`:
  - `test/shopping-cart`
  - `staging/shopping-cart`
  - `prod/shopping-cart`
- `qa-automation/`:
  - `test/regression-suite`
- `db-team/`:
  - `test/schema-migration`
  - `prod/schema-migration`

Thực tế có thể:

- 1 repo app duy nhất (source) dùng cho cả test/stg/prod, mapping env theo **branch**:
  - `develop`    → Test
  - `release/*`  → Staging
  - `main`       → Prod

### 4.2. Branch & MR flow

- Dev làm việc trên branch feature:
  - `feature/<task>` → MR vào `develop`.
- QA Test:
  - Code merge vào `develop` → Jenkins CI + deploy Test.
- Chuẩn bị release:
  - Tạo `release/x.y.z` (từ `develop`) → Jenkins CI + deploy Staging.
- Prod:
  - Sau khi Staging OK → MR từ `release/x.y.z` → `main`.
  - Jenkins-prod &/hoặc ArgoCD-prod deploy Prod.

### 4.3. Phân quyền GitLab

Phòng ban:

- **Dev**:
  - `Developer` trên repo app team mình.
- **App (App team – business owners)**:
  - Có thể `Maintainer` trên repo app, quản lý MR & tag release.
- **Test / QA**:
  - `Developer` trên repo test-automation,  
  - `Reporter` hoặc `Developer` (chỉ trên nhánh test/staging) trên app repo.
- **DB**:
  - `Maintainer` trên repo schema/migration.
- **Sec**:
  - `Reporter` trên app repo, `Developer/Maintainer` trên security policy repo.
- **DevOps**:
  - `Maintainer` trên infra repos, template CI/CD, GitOps repo.

MR rule:

- Branch `main`, `release/*` = **protected**:
  - Chỉ allow merge qua MR với ≥1–2 approver (Lead/DevOps/Sec).
- Tag release:
  - Chỉ Lead/AppOps/DevOps được tạo.

---

## 5. Jenkins – kiến trúc CI/CD

### 5.1. Jenkins trên DevOps-Test

- Jenkins controller + agent trên `EKS-devops-test:jenkins-test`.
- Kubernetes Cloud:
  - Agent templates:
    - `maven-agent`    – build Java/Maven.
    - `python-agent`   – Python, automation test.
    - `.net-agent`     – nếu build .NET (Windows agent có thể là VM riêng).
    - `docker-trivy`   – build + Trivy scan.
    - `generic-shell`  – shell, db job, v.v.

**CI cho Test + Staging:**

- **Test pipeline** (trigger từ `develop`):
  - Build & test.
  - SonarQube, Trivy/Black Duck, Nexus.
  - Deploy Test (`app-test`) → Jenkins or ArgoCD-test.

- **Staging pipeline** (trigger từ `release/*`):
  - Build (có thể reuse artifact từ Test để đảm bảo artifact bất biến).
  - Sonar/Trivy optional (vì đã check ở test).
  - Deploy Staging (`app-stg`) qua ArgoCD:
    - Jenkins update GitOps repo (sửa `image.tag`).
    - ArgoCD sync Staging.

### 5.2. Jenkins trên DevOps-Prod

- Chạy trên `EKS-devops-prod:jenkins-prod`.
- Pipeline Prod chỉ:
  - Lấy artifact/image đã được build & scan ở DevOps-Test.
  - Deploy Prod (qua ArgoCD-prod hoặc kubectl/Helm).
- Không build lại code, **không Sonar/Trivy tại Prod** (chỉ deploy đã pass ở Staging).

---

## 6. SonarQube, Trivy/Black Duck, Nexus (chỉ ở DevOps-Test)

### 6.1. SonarQube

- Dùng cho:
  - Java/Maven, Python, .NET, JS/TS, infra-as-code…
- Jenkins:
  - `withSonarQubeEnv` + `waitForQualityGate`.
  - Quality Gate fail → block merge lên `release/*` / `main`.

### 6.2. Trivy / Black Duck

- Trivy: scan Docker image:
  - Stage trong Jenkins Test:
    - Scan LOW/MEDIUM (không fail) → log cho Dev/QA.
    - Scan HIGH/CRITICAL (fail CI) với dự án bắt buộc security.
- Black Duck:
  - Scan dependency (SCA), license compliance.

### 6.3. Nexus Repository

- Lưu:
  - Maven: `*-SNAPSHOT`, `*-RELEASE`.
  - (optional) Docker test registry (nội bộ).
  - S3: backup tar image, report scan.

---

## 7. Staging & Prod – CD với ArgoCD

### 7.1. Test

- Có thể dùng ArgoCD-test:
  - Mỗi app một `Application` trỏ vào gitops-repo test.
- Jenkins Test:
  - Update gitops-repo (values/manifest) → ArgoCD-test auto-sync.

### 7.2. Staging

- GitOps repo có path `staging/<project>/values.yaml`.
- Jenkins DevOps-Test:
  - Sau build & scan ok:
    - Update `staging/values.yaml` với `image.tag = stg-<sha>`.
- ArgoCD Staging (có thể chạy trên `EKS-devops-test` hoặc `EKS-app`):
  - Sync vào namespace `app-stg`.

### 7.3. Prod

- MR từ release → main (GitLab):
  - Có approval của Lead, Sec, DevOps.
- Jenkins Prod:
  - Trigger bởi merge vào `main`.
  - Lấy image tag đã được approve (cùng `<sha>` với staging).
  - Update gitops-repo prod (`prod/<project>/values.yaml`).
- ArgoCD Prod:
  - Sync vào namespace `app-prod`.

---

## 8. Phân quyền user multi-phòng ban / môi trường / dự án

### 8.1. Quy ước phòng ban

- **Dev** – Developer, implement feature.
- **App** – Application owner / business team.
- **Test** – manual tester.
- **QA** – automation & quality assurance.
- **DB** – database team.
- **Sec** – security team.
- **DevOps** – platform/CI/CD.

### 8.2. Quyền trên GitLab

- Theo project (tên-phòng-ban / env / project):

Ví dụ `dev-backend/test/shopping-cart`:

- Dev:
  - `Developer` (push, create MR vào `develop`).
- QA:
  - `Reporter` hoặc `Developer` (chỉ trên test branch).
- App:
  - `Maintainer` (quản lý MR, tag).
- DevOps:
  - `Maintainer` (cấu hình CI, secret).

`dev-backend/prod/shopping-cart`:

- Dev:
  - `Reporter` (xem code, MR).
- App Lead:
  - `Maintainer`.
- Sec:
  - `Reporter` (review code, scan report).
- DevOps:
  - `Maintainer`.

### 8.3. Quyền trên Jenkins

- **DevOps**:
  - Jenkins admin.
- **Dev/QA/Test**:
  - Thư mục job:
    - `/test/<project>`: Build + Read.
    - `/staging/<project>`: Dev build allowed (tuỳ policy), QA read.
    - `/prod/<project>`: Read; build chỉ ProdOps/DevOps.
- **Leader / chuyên gia phòng ban**:
  - Có quyền admin trong folder dự án (hoặc Role “Project Admin”).

---

## 9. Áp dụng cho đa dạng công nghệ

Kiến trúc này áp dụng chung:

- **Maven/Java**:
  - Jenkins container `maven` + Sonar + Trivy + Nexus.
- **Python**:
  - Jenkins container `python`:
    - `pip install`, pytest, flake8, Sonar (Python rules), Docker.
- **.NET / ASP.NET**:
  - Windows agent (VM) hoặc Linux + `dotnet` SDK + container build.
- **Shell & Linux job**:
  - Jenkins generic agent chạy script.
- **AWS Lambda**:
  - Build artifact (zip), deploy bằng `aws cli`/Terraform/CloudFormation.
- **DB**:
  - Flyway/liquibase job trong Jenkins; review & approval nghiêm ngặt trước Prod.

---

## 10. Tự động hoá & an toàn

- >80% auto:
  - Build, test, quality scan, security scan, push artifact, deploy test/staging.
- Prod:
  - Tự động build & chuẩn bị, nhưng:
    - MR review multi-role (App, DevOps, Sec).
    - Manual approval stage trong Jenkins Prod trước deploy.
- Logging/audit:
  - GitLab MR history, Jenkins build history, ArgoCD sync history.

---
