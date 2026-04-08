## CI cho app Python (Flaskr)

### 1. Mục tiêu

- Chạy **unit test + coverage** cho app Flask (Flaskr tutorial).
- Chạy **SonarQube scan** cho code Python.
- Build **Docker image**, scan bảo mật bằng **Trivy**, push lên **Nexus Docker**.
- (Tuỳ chọn) sau đó nối vào GitOps/ArgoCD giống pipeline Java.

<img width="919" height="261" alt="image" src="https://github.com/user-attachments/assets/8df84685-e89d-419e-8128-bb0ed19f8fec" />


---

### 2. Cấu trúc app Flaskr

Repo (ví dụ):

```text
ci-cd-python-flask-app/
  flaskr/          # mã nguồn Flask app
  tests/           # unit test (pytest)
  setup.py / pyproject.toml
  Dockerfile
  Jenkinsfile
```
- flaskr/ là package ứng dụng (code Python).
- tests/ chứa file test_*.py – test dùng pytest.
- setup.py/pyproject.toml định nghĩa package để pip install -e ..

---

### 3. Unit test + coverage (pytest + coverage)

Trong pipeline Jenkins, trong container python:3.11-slim:

1. Tạo và activate virtualenv:
```text
python -m venv .venv
. .venv/bin/activate

```

2. Cài app & deps test:
```text
apt-get update -y && apt-get install -y curl unzip  # để dùng curl/unzip cho sonar
pip install --upgrade pip
pip install -e .
pip install '.[test]' coverage   # theo hướng dẫn Flaskr

```

---

### 4. SonarQube Scan cho Python (SonarScanner CLI)

Thay vì Maven plugin, dùng SonarScanner CLI:

1. Tải sonar-scanner CLI (một lần trong stage test):
```text
curl -sSLo sonar-scanner.zip \
  https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
unzip sonar-scanner.zip
mv sonar-scanner-5.0.1.3006-linux sonar-scanner

```

2. Trong stage scan, dùng withSonarQubeEnv và chạy:
```text
timeout(time: 5, unit: 'MINUTES') {
  def qg = waitForQualityGate()
  if (qg.status != 'OK') {
    error "Quality Gate failed: ${qg.status}"
  }
}

```

---

### 5. Build & scan Docker image

#### 5.1. Dockerfile (Python/Flask)
```text
FROM python:3.11-slim

WORKDIR /app

COPY . .

RUN pip install --upgrade pip \
    && pip install -e . \
    && pip install '.[test]'

ENV FLASK_APP=flaskr
ENV FLASK_ENV=production

EXPOSE 5000

CMD ["flask", "run", "--host=0.0.0.0"]

```

#### 5.2. Stage Docker Build + lưu TAR lên Nexus RAW

Trong container docker:24-dind:

Build image:
  
```text
docker build -t docker-internal.gitlabonlinecom.click/dev-backend/flaskr:${IMAGE_TAG} .

```

Lưu TAR:
```text
TARFILE_NAME="image-flaskr-${IMAGE_TAG}.tar"
docker save docker-internal.gitlabonlinecom.click/dev-backend/flaskr:${IMAGE_TAG} -o ${TARFILE_NAME}

```

Upload TAR lên Nexus RAW (repo raw-artifacts):

```text
curl -u "${NX_USER}:${NX_PASS}" \
  --upload-file ${TARFILE_NAME} \
  "http://nexus.gitlabonlinecom.click/repository/raw-artifacts/dev-backend/docker-tar/${TARFILE_NAME}"

```

### 6. Trivy scan image từ TAR (Nexus RAW)

Trong container trivy:

1. Tải TAR từ Nexus RAW:
```text
curl -u "${NX_USER}:${NX_PASS}" \
  -o ${TARFILE_NAME} \
  "http://nexus.gitlabonlinecom.click/repository/raw-artifacts/dev-backend/docker-tar/${TARFILE_NAME}"

```
2. Scan bằng Trivy:
```text
trivy image --input ${TARFILE_NAME} --exit-code 0 --severity LOW,MEDIUM
trivy image --input ${TARFILE_NAME} --exit-code 1 --severity HIGH,CRITICAL

```
---

### 7. Push image lên Nexus Docker internal

Trong container docker:

```text
docker login docker-internal.gitlabonlinecom.click -u ${REG_USER} --password-stdin
docker push docker-internal.gitlabonlinecom.click/dev-backend/flaskr:${IMAGE_TAG}
docker logout docker-internal.gitlabonlinecom.click

```

REG_USER/PASS là credentials Jenkins cho user Nexus có quyền trên docker-hosted.

---

### 8. (Tuỳ chọn) GitOps + ArgoCD như Java

GitOps repo: có test/staging/prod/values.yaml cho app Flask.

Jenkins update:
```text
image:
  repository: docker-internal.gitlabonlinecom.click/dev-backend/flaskr
  tag: <IMAGE_TAG>

```

ArgoCD:

  - Application Test: trỏ vào develop/test/.
  - Application Staging: trỏ vào release/APP_VERSION/staging/.
  - Application Prod: trỏ vào main/prod/.
    
Flow promote + rollback giống y hệt pipeline Java:

  1. CI trên develop: build/test/scan/push → update GitOps test → ArgoCD Test.
  2. Manual approve → update GitOps staging (release/APP_VERSION) → ArgoCD Staging.
  3. Nếu lỗi → rollback bằng git revert trên branch GitOps staging → ArgoCD sync rollback.

---

### 9. Jenkinsfile CI CD python
```bash
podTemplate(
  label: 'flask-pod',
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
      name: 'python',
      image: 'python:3.11-slim',
      command: 'cat',
      ttyEnabled: true,
      resourceRequestCpu: '500m',
      resourceRequestMemory: '512Mi',
      resourceLimitCpu: '1000m',
      resourceLimitMemory: '1Gi'
    ),
    containerTemplate(
      name: 'docker',
      image: 'docker:24-dind',
      privileged: true,
      ttyEnabled: true,
      command: 'dockerd-entrypoint.sh',
      args: '--insecure-registry=docker-internal.gitlabonlinecom.click'
    ),
    containerTemplate(
      name: 'trivy',
      image: 'aquasec/trivy:0.69.3',
      command: 'cat',
      ttyEnabled: true
    )
  ],
) {

  def SONARQUBE_SERVER  = "SonarQube"
  def SONAR_PROJECT_KEY = "python-flask-app"

  def APP_NAME      = "flaskr"
  def NEXUS_REGISTRY = "docker-internal.gitlabonlinecom.click"
  def IMAGE_NAME     = "dev-backend/${APP_NAME}"   // path trên Nexus Docker

  node('flask-pod') {

    stage('Checkout') {
      echo "[Checkout] Lấy source từ SCM"
      checkout scm
      script {
        try {
          GIT_SHORT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        } catch (e) {
          GIT_SHORT_SHA = env.BUILD_NUMBER
        }
        IMAGE_TAG = GIT_SHORT_SHA
        echo "GIT_SHORT_SHA = ${GIT_SHORT_SHA}, IMAGE_TAG = ${IMAGE_TAG}"
      }
    }

    stage('Python Unit Test & Coverage') {
      container('python') {
        sh '''
        python -m venv .venv
        . .venv/bin/activate

        # Cài thêm curl và unzip cho image python:3.11-slim (Debian-based)
        apt-get update -y
        apt-get install -y curl unzip

        # cài app và deps test theo hướng dẫn Flaskr
        pip install --upgrade pip
        pip install -e .
        pip install '.[test]' coverage

        # chạy pytest kèm coverage
        coverage run -m pytest
        coverage report
        coverage xml || true  # nếu sau này muốn publish trong Jenkins


        # cài sonar-scanner CLI (Linux)
        if ! command -v sonar-scanner >/dev/null 2>&1; then
        curl -sSLo sonar-scanner.zip \
            https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
        unzip sonar-scanner.zip
        mv sonar-scanner-5.0.1.3006-linux sonar-scanner
        fi
        '''
      }
    }

    stage('SonarQube Scan (Python)') {
    container('python') {
        withSonarQubeEnv(SONARQUBE_SERVER) {
        sh """
            . .venv/bin/activate
            export PATH="\$PWD/sonar-scanner/bin:\$PATH"

            sonar-scanner \
            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
            -Dsonar.projectName=${APP_NAME} \
            -Dsonar.projectVersion=${APP_NAME}-${GIT_SHORT_SHA} \
            -Dsonar.sources=flaskr \
            -Dsonar.tests=tests \
            -Dsonar.language=py \
            -Dsonar.python.version=3.11 \
            -Dsonar.sourceEncoding=UTF-8 \
            -Dsonar.python.coverage.reportPaths=coverage.xml
        """
        }
    }
    }


    stage('Quality Gate') {
      script {
        timeout(time: 5, unit: 'MINUTES') {
          def qg = waitForQualityGate()
          if (qg.status != 'OK') {
            error "Quality Gate failed: ${qg.status}"
          }
        }
      }
    }

    stage('Docker Build') {
      container('docker') {

        withCredentials([usernamePassword(
        credentialsId: 'nexus-raw-creds',
        usernameVariable: 'NX_USER',
        passwordVariable: 'NX_PASS'
    )]) {
            sh """
            echo "[Docker] Build image ${NEXUS_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
            docker build -t ${NEXUS_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
            docker save ${NEXUS_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} -o image-flaskr-${IMAGE_TAG}.tar

            # Cài curl nếu image docker:dind không có
            if ! command -v curl >/dev/null 2>&1; then
                apk add --no-cache curl
            fi

            echo "[RAW] Upload image-flaskr-${IMAGE_TAG}.tar lên Nexus RAW"
            ls -lh
            curl -v -w "\nHTTP_CODE:%{http_code}\n" \
                -u "${NX_USER}:${NX_PASS}" \
                --upload-file image-flaskr-${IMAGE_TAG}.tar \
                "http://nexus.gitlabonlinecom.click/repository/raw-artifacts/dev-backend/docker-tar/image-flaskr-${IMAGE_TAG}.tar"

            """
        }
      }
    }

    stage('Trivy Scan Image TAR FILE (from Nexus RAW)') {
    container('trivy') {
        withCredentials([usernamePassword(
        credentialsId: 'nexus-raw-creds',   // user có quyền trên raw-artifacts
        usernameVariable: 'NX_USER',
        passwordVariable: 'NX_PASS'
        )]) {
        sh """
            # Cài curl nếu chưa có (alpine base)
            if ! command -v curl >/dev/null 2>&1; then
            apk add --no-cache curl
            fi

            echo "[Trivy] Download TAR from Nexus RAW: image-flaskr-${IMAGE_TAG}.tar"
            curl -u "${NX_USER}:${NX_PASS}" \
            -o image-flaskr-${IMAGE_TAG}.tar \
            "http://nexus.gitlabonlinecom.click/repository/raw-artifacts/dev-backend/docker-tar/image-flaskr-${IMAGE_TAG}.tar"

            echo "[Trivy] Scan image TAR image-flaskr-${IMAGE_TAG}.tar"

            # Scan LOW + MEDIUM (không fail pipeline)
            trivy image --input image-flaskr-${IMAGE_TAG}.tar \
            --exit-code 0 \
            --severity LOW,MEDIUM

            # Scan HIGH + CRITICAL (tạm để exit-code 0, đổi 1 nếu muốn fail)
            trivy image --input image-flaskr-${IMAGE_TAG}.tar \
            --exit-code 0 \
            --severity HIGH,CRITICAL
        """
        }
    }
    }


    stage('Push Image to Nexus Docker') {
      container('docker') {
        withCredentials([usernamePassword(
          credentialsId: 'docker-user-internal',   // Jenkins credentials: user push vào Nexus Docker
          usernameVariable: 'REG_USER',
          passwordVariable: 'REG_PASS'
        )]) {
          sh """
            echo "${REG_PASS}" | docker login ${NEXUS_REGISTRY} -u "${REG_USER}" --password-stdin

            echo "[Nexus] Push image ${NEXUS_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
            docker push ${NEXUS_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}

            docker logout ${NEXUS_REGISTRY}
          """
        }
      }
    }

  } // node
} // podTemplate

```
