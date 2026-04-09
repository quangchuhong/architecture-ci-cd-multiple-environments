# Jenkins Controller – Helm + JCasC + Job DSL + JSON Pipelines

## 1. Mục tiêu

- Quản lý **Jenkins controller** bằng GitOps (Helm + ArgoCD).
- Cấu hình Jenkins bằng **Jenkins Configuration as Code (JCasC)**.
- Tự động tạo **folder + pipeline** theo cấu trúc tổ chức.
- Khai báo **project + pipeline + Git repo** qua **file JSON**, không cần sửa Groovy.

---

## 2. Kiến trúc tổng quan

### 2.1. Thành phần chính

1. **Helm chart `jenkins-controller`**
   - Deploy Jenkins (Deployment + Service + PVC).
   - Mount ConfigMap chứa:
     - `files/casc/jenkins.yaml` – JCasC.
     - `files/job-dsl/projects.json` – danh sách project/pipeline.
     - `files/job-dsl/pipelines.groovy` – engine Job DSL.

2. **JCasC – `files/casc/jenkins.yaml`**
   - Định nghĩa:
     - Local users (hoặc LDAP).
     - Role-based Authorization Strategy.
     - Role theo phòng ban / môi trường (regex).
     - **Seed job** để chạy Job DSL.

3. **Job DSL + JSON**
   - `files/job-dsl/projects.json`: input – mô tả project/pipeline/repo.
   - `files/job-dsl/pipelines.groovy`: engine – đọc JSON, tạo folder + pipeline.

---

## 3. Cấu trúc Helm chart

```text
jenkins-controller/
  Chart.yaml
  values.yaml
  files/
    casc/
      jenkins.yaml            # JCasC
    job-dsl/
      projects.json           # danh sách project + pipeline
      pipelines.groovy        # engine DSL đọc JSON, tạo job
  templates/
    configmap-jcasc.yaml
    configmap-jobdsl.yaml
    deployment.yaml
    service.yaml
    pvc.yaml
