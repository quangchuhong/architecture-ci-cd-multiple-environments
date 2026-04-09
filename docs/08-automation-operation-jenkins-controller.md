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
```

### 3.1. ConfigMap JCasC – templates/configmap-jcasc.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "jenkins-controller.fullname" . }}-jcasc
data:
  jenkins.yaml: |
{{ .Files.Get "files/casc/jenkins.yaml" | nindent 4 }}

```

### 3.2. ConfigMap Job DSL – templates/configmap-jobdsl.yaml
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

### 3.3. Deployment – templates/deployment.yaml (phần chính)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "jenkins-controller.fullname" . }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "jenkins-controller.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "jenkins-controller.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
      annotations:
        checksum/config-jcasc: {{ include (print $.Template.BasePath "/configmap-jcasc.yaml") . | sha256sum }}
        checksum/config-jobdsl: {{ include (print $.Template.BasePath "/configmap-jobdsl.yaml") . | sha256sum }}
    spec:
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-jdk17
          env:
            - name: CASC_JENKINS_CONFIG
              value: /var/jenkins_home/casc/jenkins.yaml
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
            - name: jcasc
              mountPath: /var/jenkins_home/casc
            - name: jobdsl
              mountPath: /var/jenkins_home/job-dsl
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: {{ include "jenkins-controller.fullname" . }}
        - name: jcasc
          configMap:
            name: {{ include "jenkins-controller.fullname" . }}-jcasc
            items:
              - key: jenkins.yaml
                path: jenkins.yaml
        - name: jobdsl
          configMap:
            name: {{ include "jenkins-controller.fullname" . }}-jobdsl
            items:
              - key: projects.json
                path: projects.json
              - key: pipelines.groovy
                path: pipelines.groovy

// checksum/* giúp ArgoCD tự rollout Jenkins khi ConfigMap thay đổi.


```
---

## 4. JCasC – files/casc/jenkins.yaml

### 4.1. Users & Role-based Auth

Ví dụ đơn giản (local users + role theo phòng ban/env):
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
          # Dev - full quyền mọi job dưới dev/dev/*
          - name: "dev_dev_env_rw"
            pattern: "^dev/dev/.*"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "Job/Configure"
              - "View/Read"
            assignments:
              - "dev-user1"

          # Dev - read+build mọi job dưới dev/(staging|prod)/*
          - name: "dev_stg_prod_rb"
            pattern: "^dev/(staging|prod)/.*"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "View/Read"
            assignments:
              - "dev-user1"

          # DevSecOps - admin mọi job prod (mọi phòng ban)
          - name: "devsecops_prod_admin"
            pattern: "^.*/prod/.*"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "Job/Configure"
              - "Job/Delete"
              - "View/Read"
            assignments:
              - "devsecops-user1"

          # Tester - read+build mọi job dưới tester/*
          - name: "tester_build_rb"
            pattern: "^tester/.*"
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

```

### 4.2. Seed job chạy Job DSL
```yaml
  jobs:
    - script: >
        job('seed-projects') {
          description('Seed: đọc projects.json, tạo folder + pipeline')
          logRotator { numToKeep(5) }
          steps {
            dsl {
              external('/var/jenkins_home/job-dsl/pipelines.groovy')
              removeAction('DELETE')
              ignoreExisting(false)
            }
          }
        }

```
---

## 5. JSON cấu hình project/pipeline – files/job-dsl/projects.json

JSON là nơi nhập dữ liệu:

   - Phòng ban (dept).
   - Môi trường (env).
   - Tên project (name).
   - Mỗi pipeline có thể trỏ tới repo HTTPS và branch riêng.
     
Ví dụ:
```json
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

      "gitRepo": "https://gitlab.com/your-group/proj-a.git",
      "branch": "feature/default-branch",

      "pipelines": [
        {
          "name": "proj-a-dev-build",
          "jenkinsfile": "jenkins/Jenkinsfile-build",
          "gitRepo": "https://gitlab.com/your-group/proj-a-build.git",
          "branch": "feature/build-branch"
        },
        {
          "name": "proj-a-dev-deploy",
          "jenkinsfile": "jenkins/Jenkinsfile-deploy",
          "gitRepo": "https://gitlab.com/your-group/proj-a-deploy.git"
        },
        {
          "name": "proj-a-dev-scan",
          "jenkinsfile": "jenkins/Jenkinsfile-scan",
          "gitRepo": "https://gitlab.com/your-group/security-scan.git",
          "branch": "main"
        }
      ]
    },
    {
      "dept": "dev",
      "env": "staging",
      "name": "proj-a",
      "gitRepo": "https://gitlab.com/your-group/proj-a.git",
      "pipelines": [
        {
          "name": "proj-a-stg-ci",
          "jenkinsfile": "jenkins/Jenkinsfile-ci"
        }
      ]
    }
  ]
}

```

Ưu tiên giá trị:

   - Repo:
      - Nếu có pipeline.gitRepo → dùng repo đó.
      - Nếu không → dùng project.gitRepo.
        
   - Branch:
      1. pipeline.branch nếu có.
      2. Nếu không → project.branch nếu có.
      3. Nếu không → defaultBranchPerEnv[env].
      4. Nếu vẫn không → "main".

---

## 6. Engine Job DSL – files/job-dsl/pipelines.groovy

Groovy đọc JSON và tạo folder + pipeline.
```groovy
import groovy.json.JsonSlurper

// 1. Đọc JSON
def cfgText = new File('/var/jenkins_home/job-dsl/projects.json').text
def cfg = new JsonSlurper().parseText(cfgText)

def defaultBranchPerEnv = cfg.defaultBranchPerEnv ?: [:]
def gitCredId = cfg.gitCredentialId

// 2. Tạo folder: dept / env / project
cfg.projects.groupBy { it.dept }.each { dept, deptProjects ->
  folder(dept) {
    displayName(dept.toUpperCase())
    description("Phòng ban: ${dept}")
  }

  deptProjects.groupBy { it.env }.each { envName, envProjects ->
    folder("${dept}/${envName}") {
      displayName("${dept}-${envName}")
      description("Env: ${envName} của phòng ban ${dept}")
    }

    envProjects.groupBy { it.name }.each { projName, projConfigs ->
      folder("${dept}/${envName}/${projName}") {
        displayName("${projName} (${envName})")
        description("Project ${projName} - Dept: ${dept} - Env: ${envName}")
      }

      // 3. Tạo pipeline cho từng cấu hình project/env
      projConfigs.each { p ->
        def projectFolder      = "${dept}/${envName}/${projName}"
        def projectDefaultRepo = p.gitRepo
        def projectDefaultBr   = p.branch

        p.pipelines.each { pl ->
          def jobPath   = "${projectFolder}/${pl.name}"
          def gitRepo   = pl.gitRepo ?: projectDefaultRepo
          def gitBranch = pl.branch
                            ?: projectDefaultBr
                            ?: defaultBranchPerEnv[envName]
                            ?: "main"

          if (!gitRepo) {
            println "[WARN] Bỏ qua ${jobPath} vì không có gitRepo"
            return
          }

          pipelineJob(jobPath) {
            description("Pipeline '${pl.name}' for ${projName} (${dept}/${envName})")
            logRotator { numToKeep(20) }
            properties { disableConcurrentBuilds() }

            definition {
              cpsScm {
                scm {
                  git {
                    remote {
                      url(gitRepo)           // HTTPS URL
                      credentials(gitCredId) // Jenkins credential
                    }
                    branch("refs/heads/${gitBranch}")
                  }
                }
                scriptPath(pl.jenkinsfile)
              }
            }
          }
        }
      }
    }
  }
}

return this

```

Kết quả:

   - Folder Jenkins: dept/env/project.
   - Trong mỗi project có các job theo pipelines[].name, mỗi job gắn đúng repo + branch + Jenkinsfile.
