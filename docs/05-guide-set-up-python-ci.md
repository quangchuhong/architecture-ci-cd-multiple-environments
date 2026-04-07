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
