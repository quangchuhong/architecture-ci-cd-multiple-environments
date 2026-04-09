# Jenkins Controller Helm + JCasC + Job DSL (GitOps Guide)

## 1. Mục tiêu

- Quản lý **Jenkins controller** như code:
  - Cấu hình Jenkins (user, role, seed job, plugin logic) bằng **JCasC**.
  - Cấu trúc **phòng ban / môi trường / project / pipeline** bằng **Job DSL**.
- Deploy & cập nhật Jenkins qua **Helm** (và/hoặc ArgoCD).

---

## 2. Cấu trúc Helm chart

```text
jenkins-controller/
  Chart.yaml
  values.yaml
  files/
    casc/
      jenkins.yaml              # JCasC
    job-dsl/
      org-structure.groovy      # Job DSL seed
  templates/
    configmap-jcasc.yaml        # CM chứa jenkins.yaml
    configmap-jobdsl.yaml       # CM chứa org-structure.groovy
    deployment.yaml             # Jenkins deployment (mount 2 CM)
    service.yaml                # Jenkins service
    pvc.yaml                    # Jenkins home (tuỳ chọn)
```
---

## 3. Cấu trúc Jenkins (folder & pipeline)

Quy ước folder Jenkins:
```text
<department>/<env>/<project>/<job>

```

ví dụ:
```text
dev/
  dev/
    proj-a/
      build
      deploy
    proj-b/
      build
  staging/
    proj-a/
      build
  prod/
    proj-a/
      build
devsecops/
  dev/
    sec-tools/
tester/
  dev/
    proj-a/
db/
  prod/
    db-maintenance/

```
  - Phòng ban: dev, devsecops, tester, db
  - Env: dev, staging, prod
  - Mỗi project: 2 pipeline chuẩn build, deploy.
    
---

## 4. JCasC – files/casc/jenkins.yaml

Định nghĩa:

  - Local users (hoặc LDAP).
  - Role-based Authorization Strategy.
  - Role theo phòng ban / env / pipeline (dùng regex pattern).
  - Seed job chạy Job DSL.
    
Ví dụ rút gọn:
```yaml
jenkins:
  systemMessage: "Jenkins managed by Helm + JCasC + Job DSL"

  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "Admin@123"
        - id: "dev-user1"
          password: "Dev@123"
        - id: "devsecops-user1"
          password: "DevSec@123"
        - id: "tester-user1"
          password: "Tester@123"
        - id: "db-user1"
          password: "DB@123"

  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
              - "Overall/Read"
            assignments:
              - "admin"
        project:
          # Dev - full quyền dev/dev/**/(build|deploy)
          - name: "dev_dev_env_rw"
            pattern: "^dev/dev/.*/(build|deploy)$"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "Job/Configure"
              - "View/Read"
            assignments:
              - "dev-user1"

          # Dev - read+build dev/(staging|prod)
          - name: "dev_stg_prod_rb"
            pattern: "^dev/(staging|prod)/.*/(build|deploy)$"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "View/Read"
            assignments:
              - "dev-user1"

          # DevSecOps - admin tất cả prod pipelines
          - name: "devsecops_prod_admin"
            pattern: "^.*/prod/.*/(build|deploy)$"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "Job/Configure"
              - "Job/Delete"
              - "View/Read"
            assignments:
              - "devsecops-user1"

          # Tester - read+build tester/**/build
          - name: "tester_build_rb"
            pattern: "^tester/.*/.*/build$"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "View/Read"
            assignments:
              - "tester-user1"

          # DB team - admin db/*
          - name: "db_team_admin"
            pattern: "^db/.*"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "Job/Configure"
              - "Job/Delete"
              - "View/Read"
            assignments:
              - "db-user1"

  jobs:
    - script: >
        job('seed-org-structure') {
          description('Seed job: tạo folder/pipeline theo ORG_STRUCTURE')
          logRotator { numToKeep(5) }
          steps {
            dsl {
              external('/var/jenkins_home/job-dsl/org-structure.groovy')
              removeAction('DELETE')
              ignoreExisting(false)
            }
          }
        }

```
