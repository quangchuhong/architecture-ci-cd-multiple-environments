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
---

### 6. Trivy scan image

Trong container trivy:
```text
trivy image --exit-code 0 --severity LOW,MEDIUM \
  docker-internal.gitlabonlinecom.click/dev-frontend/my-node-app:${IMAGE_TAG}

trivy image --exit-code 0 --severity HIGH,CRITICAL \
  docker-internal.gitlabonlinecom.click/dev-frontend/my-node-app:${IMAGE_TAG}

```

  - Nếu muốn chặn build khi có vuln HIGH/CRITICAL → đổi --exit-code 0 thành 1 ở dòng thứ hai.

---

### 7. Push image lên Nexus Docker internal

Trong container docker:
```text
docker login docker-internal.gitlabonlinecom.click -u ${REG_USER} --password-stdin

docker push docker-internal.gitlabonlinecom.click/dev-frontend/my-node-app:${IMAGE_TAG}

docker logout docker-internal.gitlabonlinecom.click

```
  - ${REG_USER} & ${REG_PASS} là Jenkins credentials trỏ đến user Nexus có quyền push vào repo docker-hosted.

---

### 8. (Tuỳ chọn) GitOps + ArgoCD

Nếu bạn dùng GitOps như Java/Python:

GitOps repo:
```text
gitops-repo/
  node-app/
    test/values.yaml
    staging/values.yaml
    prod/values.yaml

```

Trong values.yaml:
```text
image:
  repository: docker-internal.gitlabonlinecom.click/dev-frontend/my-node-app
  tag: <IMAGE_TAG>

```

### 8.1. Jenkins update GitOps (Test)

  - Clone GitOps repo.  
  - Dựa vào BRANCH_NAME:
    - develop → envPath = test, gitTargetBranch = develop
    - release/* → envPath = staging, gitTargetBranch = staging
    - main → envPath = prod, gitTargetBranch = main
      
Dùng yq:
```text
yq e '.image.repository = "docker-internal.gitlabonlinecom.click/dev-frontend/my-node-app"' -i values.yaml
yq e '.image.tag = "'${IMAGE_TAG}'"' -i values.yaml

```

Commit & push:
```text
git commit -am "Update my-node-app image tag to ${IMAGE_TAG} for ${envPath}" || echo "No changes"
git push origin ${gitTargetBranch}

```

ArgoCD:

  - Application Test: trỏ vào branch/path develop/node-app/test.
  - Auto-sync → deploy image mới vào namespace Test trên EKS.

### 8.2. Promote Staging & rollback

Flow giống Java/Python:

  1. Manual approval trong Jenkins để promote image Test → Staging.
     
  2. Jenkins update staging/values.yaml trên branch release/APP_VERSION.
     
  3. ArgoCD Staging sync deploy.
     
  4. Nếu lỗi:
     
    - Manual rollback → git revert HEAD trên release/APP_VERSION (staging)
    - ArgoCD sync rollback về image tag cũ.

---

### 9. Tóm tắt pipeline CI/CD Node.js
    
  1. Checkout
     
  2. npm ci / npm install
     
  3. npm test / npm run coverage
     
  4. (Optional) npm run lint
     
  5. docker build
     
  6. Trivy scan
     
  7. docker push Nexus Docker internal
     
  8. (Optional) Update GitOps (test/staging/prod) + ArgoCD deploy/rollback
