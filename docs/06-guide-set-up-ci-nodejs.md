## CI cho ứng dụng Node.js

### 1. Mục tiêu

- Chạy **unit test + coverage** cho app Node.js (Jest/Mocha).  
- Build **Docker image** cho app Node.js.  
- Scan bảo mật bằng **Trivy**.  
- Push image lên **Nexus Docker internal**.  
- (Tuỳ chọn) triển khai qua **GitOps + ArgoCD** giống Java/Python.

---

### 2. Cấu trúc cơ bản một app Node.js

Ví dụ:

```text
my-node-app/
  src/
    index.js
    routes/
    services/
  test/
    example.test.js        # Jest/Mocha test
  package.json
  package-lock.json        # hoặc yarn.lock / pnpm-lock.yaml
  Dockerfile
  Jenkinsfile
```

src/ → code chính.
test/ → unit test (Jest, Mocha, v.v.).
package.json → script build/test.
Dockerfile → đóng gói app vào container.
Jenkinsfile → pipeline CI/CD.
