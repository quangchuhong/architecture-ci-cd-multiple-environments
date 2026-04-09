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
