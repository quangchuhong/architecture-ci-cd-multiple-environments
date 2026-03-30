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
```text
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

```

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

## 11. Sizing cho 100 pipeline concurrent (Dev + QA + Tester)

Bối cảnh:

- Nhiều QA, Tester, Dev cùng lúc chạy build/test trên **EKS-devops-test**.
- Số lượng pipeline concurrent dự kiến: **~100** (đa số là Test).
- Mỗi pipeline gồm 2–4 stage nặng:
  - Build + Unit Test (Maven/Python/.NET/…)
  - Docker build
  - Security scan (Trivy/Black Duck)

Mục tiêu:

- Đảm bảo cụm **DevOps-Core** (GitLab, Jenkins master, SonarQube, Nexus) **luôn ổn định**.
- Cho phép **30–40 pipeline nặng** chạy song song; phần còn lại chờ trong queue (Jenkins).
- Tự động scale node group **CI-Agent** theo nhu cầu, tránh over-provision.

---

### 11.1. Phân tách node group

Trên cluster `EKS-devops-test`:

1. **Node group DevOps-Core** (ổn định):
   - Chạy: Jenkins controller, GitLab, SonarQube, Nexus, ArgoCD-test (nếu có).
   - Ví dụ:
     - instance type: `m5.xlarge` (4 vCPU, 16 GiB)
     - node count: 3–4 node (tùy tải).

2. **Node group CI-Agent** (autoscale, chỉ cho Jenkins agents):
   - Chạy: pod agent Jenkins (maven, docker, trivy, python, .net…).
   - Autoscale theo số lượng pipeline.
   - Sử dụng `nodeSelector` / `taints` để:
     - DevOps-Core pod **không** chạy trên CI-Agent node.
     - Jenkins agents **chỉ** chạy trên CI-Agent node.

Ví dụ `nodeSelector` cho agents:

```yaml
# Trong podTemplate Jenkins
spec:
  nodeSelector:
    role: ci-agent
```
### 11.2. Ước lượng resource cho 1 pipeline

Với 1 Jenkins agent kiểu “maven + docker + trivy”:

- Container `maven`:
  - requests: ~0.5 vCPU, 1–2 GiB RAM
  - limits: 1 vCPU, 2–3 GiB RAM
- Container `docker`:
  - requests: ~0.5 vCPU, 1 GiB RAM
  - limits: 1 vCPU, 2 GiB RAM
- Container `trivy`:
  - requests: ~0.3 vCPU, 0.5–1 GiB RAM
  - limits: 0.5 vCPU, 1–2 GiB RAM

**Peak mỗi pod agent (1 pipeline):**

- CPU: ~2–2.5 vCPU  
- RAM: ~4–5 GiB

---

### 11.3. Sizing CI-Agent node group cho ~100 pipeline

Thực tế không nên cho 100 pipeline **đều peak cùng lúc**, mà:

- Cho phép **30–40 pipeline nặng** chạy song song.
- Các pipeline còn lại vào queue (Jenkins quản lý).

#### Phương án 1 – nhiều node vừa (linh hoạt)

- Node group `ci-agent`:
  - instance type: `m5.xlarge` (4 vCPU, 16 GiB RAM)
  - min: 0–2 node
  - max: 20–25 node

Tổng tài nguyên tối đa:

- CPU: 4 * 25 = 100 vCPU  
- RAM: 16 * 25 = 400 GiB

→ Cho phép ~30–40 pod agent nặng chạy song song (mỗi pod ~2–2.5 vCPU).

#### Phương án 2 – ít node to hơn

- Node group `ci-agent`:
  - instance type: `m5.2xlarge` (8 vCPU, 32 GiB RAM)
  - min: 0–1 node
  - max: 10–12 node

Tổng tài nguyên tối đa:

- CPU: 8 * 12 = 96 vCPU  
- RAM: 32 * 12 = 384 GiB

→ Tương tự, đủ cho ~30–40 pipeline nặng.

---

### 11.4. Giới hạn trong Jenkins (Kubernetes Cloud)

Để tránh agent “nở” quá nhiều pod:

Ví dụ JCasC (hoặc config Jenkins) cho Kubernetes Cloud:

```yaml
jenkins:
  clouds:
    - kubernetes:
        name: "kubernetes"
        containerCapStr: "50"   # tổng agents tối đa toàn cloud
        templates:
          - name: "maven-docker-trivy"
            label: "maven-pod"
            namespace: "jenkins-test"
            instanceCapStr: "25"  # tối đa 25 pod kiểu này
```

### 11.5. ResourceQuota namespace Jenkins

Trong namespace `jenkins-test`, đặt quota để giới hạn tổng CPU/RAM cho toàn bộ Jenkins agents:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: jenkins-agents-quota
  namespace: jenkins-test
spec:
  hard:
    requests.cpu: "80"        # tối đa 80 vCPU requested
    limits.cpu: "100"
    requests.memory: "160Gi"
    limits.memory: "220Gi"
    pods: "60"                # tối đa 60 pod (agents + controller)
```

Tác dụng:

  - Nếu Jenkins muốn tạo thêm pod agent vượt quá quota (ví dụ >60 pod) → pod sẽ Pending → job queue.

  - Bảo vệ cluster khỏi bị “ăn” hết tài nguyên bởi CI.

---

### 11.6. Tối ưu riêng cho môi trường Test (nhiều QA/Tester)

1. Tách job “build artifact” và job “chạy test”:

  - Build image/binary 1 lần (Dev pipeline).
  - QA/Tester chạy job test (UI/API/regression) tái sử dụng image/binary đã build:
    - Giảm số lần docker build.
    - Giảm tải CPU/IO cho node CI-Agent.

2. Cache Maven/NuGet/Python packages:

  - Mount PVC cho ~/.m2/repository, .nuget, pip cache:
    - Giảm thời gian build.
    - Giảm tải network tới Maven Central/NuGet/PyPI.

3. Ưu tiên dùng agent nhẹ cho test:

  - Job chạy test tự động (không build) dùng agent nhẹ:
    - Container chỉ cần runtime + test framework (pytest, junit, selenium…).
    - resources: 0.5–1 vCPU, 1–2 GiB RAM.

4. Scheduling khung giờ:

  - Batch test lớn (regression full) nên chạy vào:
    - ngoài giờ cao điểm Dev (tối/đêm/cuối tuần),
    - hoặc dùng Jenkins throttleConcurrentBuilds để giới hạn.
---
### 11.7. Node group DevOps-Core

Để đảm bảo các DevOps tools lõi (GitLab, Jenkins controller, SonarQube, Nexus) **không bị ảnh hưởng** khi CI load cao (nhiều Jenkins agent chạy), nên tách riêng 1 node group **DevOps-Core**.

#### 11.7.1. Mục tiêu

- Đảm bảo:
  - GitLab luôn phản hồi tốt (MR, review, pipeline view).
  - Jenkins controller không bị OOM/treo khi nhiều agent chạy.
  - SonarQube, Nexus ổn định khi nhiều job phân tích/đẩy artifact.
- Mọi tải build/test nặng (CPU/RAM/IO cao) **chỉ** chạy trên node group `ci-agent`.

#### 11.7.2. Cấu hình gợi ý

Node group `devops-core` trên `EKS-devops-test`:

- instance type: `m5.xlarge` (4 vCPU, 16 GiB RAM)
- node count: 3–4 node
- dùng cho:
  - Pod GitLab
  - Pod Jenkins controller
  - Pod SonarQube
  - Pod Nexus
  - (Optional) ArgoCD-test, monitoring core

Node group `ci-agent`:

- chỉ chạy Jenkins agents (pod build/test).

#### 11.7.3. Taints & Tolerations (ý tưởng)

1. **Thêm taint lên node devops-core**

Ví dụ:

```bash
kubectl taint nodes <devops-core-node-name> role=devops-core:NoSchedule
```
Taint này có nghĩa: chỉ những pod nào có tolerations tương ứng mới được schedule lên node này.

2. **Thêm tolerations cho các pod core**
   
Ví dụ (trong deployment Jenkins controller):

```text
spec:
  template:
    spec:
      tolerations:
        - key: "role"
          operator: "Equal"
          value: "devops-core"
          effect: "NoSchedule"
      nodeSelector:
        role: devops-core
```
Tương tự cho GitLab, SonarQube, Nexus.

3. **Không thêm tolerations cho Jenkins agents**
   
  - PodTemplate Jenkins agent (maven, docker, trivy, python, …) không khai báo tolerations với role=devops-core.
  - Như vậy:
    - Agents sẽ bị Kubernetes từ chối schedule lên node devops-core.
    - Chỉ được chạy trên node group ci-agent.
      
#### 11.7.4. Kết quả

  - Node group devops-core luôn chỉ chứa:
    - GitLab
    - Jenkins controller
    - SonarQube
    - Nexus
  - Node group ci-agent chịu toàn bộ tải:
    - Build/test, Docker build, Trivy/Black Duck scan.
  - Khi QA/Tester/Dev bắn nhiều job:
    - Autoscaler chỉ mở rộng ci-agent nodes.
    - Hệ thống core DevOps vẫn ổn định, không bị “ăn hết” CPU/RAM.
---
## 12. Monitoring cho hạ tầng DevOps Tools trên EKS

Mục tiêu:

- Giám sát **trạng thái & hiệu năng** của:
  - GitLab, Jenkins, SonarQube, Nexus, ArgoCD (nếu có)
  - Cụm `EKS-devops-test`, `EKS-devops-prod`
- Cảnh báo sớm khi:
  - Jenkins queue dài, agent thiếu.
  - GitLab/Sonar/Nexus lỗi, chậm.
  - Node CPU/RAM/Storage gần full.
  - Pod DevOps core bị CrashLoop/Restart nhiều.

### 12.1. Kiến trúc monitoring tổng thể

Trên mỗi cluster DevOps (`EKS-devops-test`, `EKS-devops-prod`) triển khai:

- **Prometheus** – thu thập metrics.
- **Grafana** – dashboard.
- **Loki** (hoặc EFK: Elasticsearch + Fluentd + Kibana) – logging.
- **Alertmanager** – cảnh báo (email, Slack, Teams…).

Namespace gợi ý:

```text
EKS-devops-test:
  monitoring/
    - prometheus
    - grafana
    - loki / fluent-bit
    - alertmanager
```
### 12.2. Metrics quan trọng theo từng tool

**Jenkins**

**- Controller:**

  - JVM heap usage, GC time.
  - Executor usage (% busy).
  - Build queue length (số job đợi).
  - Thời gian response HTTP.
    
**- Agents:**

  - Số active agents.
  - Thời gian spawn agent (từ pending → running).
    
**- Alert gợi ý:**

  - Queue length > N trong M phút.
  - Không có agent khả dụng trong X phút.
  - Jenkins HTTP 5xx.
    
**- GitLab**
  - Web/API:
    - HTTP 4xx/5xx.
    - Thời gian response.
      
  - Sidekiq:
    - Job queue size, failed jobs.
      
  - Database:
    - Connection count, slow queries (nếu có exporter).
      
  - Alert:
    - HTTP 5xx tăng.
    - Sidekiq queue backlog.
      
**- SonarQube**

  - JVM:
    - Heap usage, GC.
      
  - Background tasks:
    - Thời gian xử lý analysis.
    - Số task fail.
      
  - Alert:
    - Background tasks failing liên tục.
    - Memory usage > ~80% kéo dài.
      
**Nexus**

  - JVM, thread count.
  - Repository metrics:
    - Request rate (QPS) / error rate.
      
  - Disk usage:
    - Dung lượng volume Nexus (PVC) > 80–90%.
      
  - Alert:
    - Disk > 80%.
    - Error rate tăng bất thường.
      
**EKS (cluster & node)**

  - Node:
    - CPU, Memory, Disk, Pod count.
      
  - Pod:
   - Restart count.
   - CrashLoopBackOff.
     
**Alert:**

  - Node NotReady.
  - Pod core (GitLab/Jenkins/Sonar/Nexus) CrashLoop.

### 12.3. Logging

**Sử dụng Loki (hoặc EFK) để thu log:**

  - Jenkins:
    - jenkins container log.
      
  - GitLab:
    - web, api, sidekiq, gitaly log.
      
  - SonarQube:
    - web, ce, es log.
      
  - Nexus:
    - application log.
    - K8s events (optional).
      
**Từ đó:**
  - Xây dashboard:
    - Xem error rate theo service.
    - Tìm lỗi build, lỗi deploy nhanh.
      
  - Alert:
    - Keyword-based (ví dụ: SEVERE, FATAL, OutOfMemoryError).
