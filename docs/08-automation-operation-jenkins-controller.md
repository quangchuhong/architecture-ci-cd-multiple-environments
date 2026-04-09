# Jenkins Pipeline Provisioning bằng Job DSL + JSON

## Mục tiêu

- Tự động tạo **folder** và **pipeline** trên Jenkins từ **file JSON cấu hình**, không phải sửa Groovy.
- Mỗi pipeline:
  - Có **tên tự do** (không bị fix `build` / `deploy`).
  - Gắn với **Git repo HTTPS** (GitLab/GitHub…).
  - Chỉ định **branch** và **Jenkinsfile path**.

---

## 1. Cấu trúc file & Helm

Trong chart `jenkins-controller`:

```text
jenkins-controller/
  files/
    job-dsl/
      projects.json          # file JSON cấu hình input
      pipelines.groovy       # engine Job DSL đọc JSON, tạo pipeline
  templates/
    configmap-jobdsl.yaml    # ConfigMap mount 2 file trên vào Jenkins
```

templates/configmap-jobdsl.yaml:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "jenkins-controller.fullname" . }}-jobdsl
data:
  projects.json: |
{{ .Files.Get "files/job-dsl/projects.json" | nindent 4 }}
  pipelines.groovy: |
{{ .Files.Get "files/job-dsl/pipelines.groovy" | nindent 4 }}

```

Deployment mount ConfigMap này vào:
```yaml
volumeMounts:
  - name: jobdsl
    mountPath: /var/jenkins_home/job-dsl
volumes:
  - name: jobdsl
    configMap:
      name: {{ include "jenkins-controller.fullname" . }}-jobdsl

```
---

## 2. File JSON cấu hình – files/job-dsl/projects.json
Đây là nơi nhập dữ liệu đầu vào: phòng ban, môi trường, project, pipelines, git repo.

Ví dụ:
```text
{
  "gitCredentialId": "gitlab-https-token", 
  "defaultBranchPerEnv": {
    "dev": "develop",
    "staging": "staging",
    "prod": "main"
  },
  "projects": [
    {
      "dept": "dev",
      "env": "dev",
      "name": "proj-a",

      "gitRepo": "https://gitlab.com/your-group/proj-a.git",   // OPTIONAL: default cho project
      "branch": "feature/init",                                 // OPTIONAL: default cho project

      "pipelines": [
        {
          "name": "proj-a-dev-build",
          "jenkinsfile": "jenkins/Jenkinsfile-build",

          "gitRepo": "https://gitlab.com/your-group/proj-a-build.git",   // repo riêng
          "branch": "feature/build-branch"                               // branch riêng (optional)
        },
        {
          "name": "proj-a-dev-deploy",
          "jenkinsfile": "jenkins/Jenkinsfile-deploy",

          "gitRepo": "https://gitlab.com/your-group/proj-a-deploy.git"   // repo khác
          // không khai branch -> dùng branch của project / defaultBranchPerEnv
        },
        {
          "name": "proj-a-dev-scan",
          "jenkinsfile": "Jenkinsfile",
          "gitRepo": "https://gitlab.com/your-group/security-scan.git",
          "branch": "main"
        }
      ]
    }
  ]
}


```
