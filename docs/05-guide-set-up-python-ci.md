## CI cho app Python (Flaskr)

### 1. Mục tiêu

- Chạy **unit test + coverage** cho app Flask (Flaskr tutorial).
- Chạy **SonarQube scan** cho code Python.
- Build **Docker image**, scan bảo mật bằng **Trivy**, push lên **Nexus Docker**.
- (Tuỳ chọn) sau đó nối vào GitOps/ArgoCD giống pipeline Java.

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
