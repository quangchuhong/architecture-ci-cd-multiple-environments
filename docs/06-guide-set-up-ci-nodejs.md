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

  - src/ → code chính.

  - test/ → unit test (Jest, Mocha, v.v.).

  - package.json → script build/test.

  - Dockerfile → đóng gói app vào container.

  - Jenkinsfile → pipeline CI/CD.
---

### 3. CI trên Jenkins – luồng tổng quát

  1. Checkout code từ Git (GitLab), branch develop / feature/* / release/* / main.
     
  2. Cài dependencies bằng npm / yarn / pnpm.
     
  3. Chạy unit test + coverage (Jest/Mocha + NYC).
     
  4. (Tuỳ chọn) Lint với ESLint.
     
  5. Build Docker image cho app Node.
     
  6. Trivy scan image.
     
  7. Push image lên Nexus Docker internal.
     
  8. (Tuỳ chọn) Update GitOps repo → ArgoCD deploy Test/Staging/Prod.
---

### 4. Chi tiết các bước trong pipeline

#### 4.1. Checkout & tính IMAGE_TAG

  - Jenkins lấy code từ Git.
  - Dùng git rev-parse --short HEAD để lấy GIT_SHORT_SHA.
  - IMAGE_TAG = GIT_SHORT_SHA (hoặc thêm prefix theo môi trường: test-, stg-, prod-).

#### 4.2. Cài Node & dependencies

Trong container có Node.js (ví dụ node:20-alpine):
```text
npm ci        # ưu tiên nếu có package-lock.json
# hoặc
npm install   # nếu không có lockfile

```
  - npm ci đảm bảo build lặp lại đúng dependency phiên bản đã lock.


#### 4.3. Unit Test + Coverage

Giả sử dùng Jest:

Trong package.json:
```text
"scripts": {
  "test": "jest --runInBand",
  "coverage": "jest --coverage --runInBand"
}

```
Trong Jenkins:
```text
npm run coverage
# hoặc
npm test

```
  - Jest sinh thư mục coverage/:
    - coverage/lcov.info
    - coverage/index.html
  - Có thể publish coverage report trong Jenkins bằng plugin (Cobertura/HTML Publisher…).
    
Nếu dùng Mocha + NYC:
```text
"scripts": {
  "test": "nyc mocha 'test/**/*.test.js'"
}

```
#### 4.4. (Tuỳ chọn) Lint

Nếu bạn bật lint:
```text
"scripts": {
  "lint": "eslint src test"
}

```
Trong Jenkins:
```text
npm run lint

```
---

### 5. Docker build cho Node.js

Dockerfile ví dụ:
```text
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

ENV NODE_ENV=production
EXPOSE 3000

CMD ["node", "src/index.js"]

```

Trong Jenkins (container docker:dind):
```text
docker build -t docker-internal.gitlabonlinecom.click/dev-frontend/my-node-app:${IMAGE_TAG} .

````
