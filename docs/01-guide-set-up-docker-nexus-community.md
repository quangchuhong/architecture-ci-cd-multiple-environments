
# Guide: Cài đặt & cấu hình Docker Registry trên Nexus Repository OSS (Community)  
_kèm so sánh nhanh Nexus OSS vs Nexus Pro_

---

## 1. Mục tiêu

- Dùng **Nexus Repository OSS** làm Docker Registry nội bộ:
  - Docker proxy (Docker Hub)
  - Docker hosted (internal images)
  - Tách endpoint cho:
    - proxy (DevOps/SEC)
    - internal (các phòng ban)
- Phân quyền user theo **phòng ban / namespace** ở mức Docker registry.
- Hiểu rõ **giới hạn của Nexus OSS** và so với **Nexus Pro**.

Giả sử:

- Domain UI: `nexus.gitlabonlinecom.click`
- Docker endpoints:
  - Proxy: `docker-proxy.gitlabonlinecom.click`  → repo `docker-proxy`
  - Internal (hosted): `docker-internal.gitlabonlinecom.click` → repo `docker-hosted`
  - (Group `docker-all` có thể dùng để **pull** – OSS không cho push vào group)

---

## 2. Các bước chính

1. Cài Nexus 3 (OSS) trên K8s (EKS) hoặc VM.
2. Tạo repo Docker:
   - `docker-proxy` (remote: Docker Hub)
   - `docker-hosted` (internal images)
   - (optional) `docker-all` (group proxy + hosted, dùng cho **pull**).
3. Tạo **Service + Ingress** riêng cho:
   - proxy: `docker-proxy.gitlabonlinecom.click`
   - hosted: `docker-internal.gitlabonlinecom.click`
4. Cấu hình client (Docker daemon / Jenkins) để dùng registry nội bộ.
5. Thiết kế phân quyền theo phòng ban (Content Selector + Privilege + Role + User).
6. (Option) Bổ sung Kyverno để siết việc **dùng image đúng prefix** trong K8s.

---

## 3. Cấu hình repo Docker trong Nexus OSS

### 3.1. Docker proxy (Docker Hub)

Trong UI Nexus:

1. **Administration → Repositories → Create repository → docker (proxy)**  
2. Điền:
   - Name: `docker-proxy`
   - Remote storage: `https://registry-1.docker.io`
   - Docker Index: Use Docker Hub
   - HTTP Port: `8002` (ví dụ)
3. Create.

### 3.2. Docker hosted (internal)

1. **Create → docker (hosted)**  
2. Name: `docker-hosted`
3. HTTP Port: `8000`
4. Enable Docker V2 API.
5. Create.

### 3.3. Docker group (chỉ để pull – OSS không cho push)

1. **Create → docker (group)**  
2. Name: `docker-all`
3. HTTP Port: `8001`
4. Members:
   - `docker-hosted`
   - `docker-proxy`
5. Create.

> Lưu ý: **Nexus OSS không cho push vào group**, chỉ Pro mới cho.  
> Khi push phải dùng **hosted** (`docker-hosted`), group chỉ để tiện **pull**.

---

## 4. Service & Ingress trên EKS

Giả sử bạn đã có Deployment Nexus (port container 8081, 8000, 8001, 8002).

### 4.1. Service chung cho các connector docker

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nexus-docker
  namespace: nexus
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: nexus-repository-manager
    app.kubernetes.io/instance: nexus
  ports:
    - name: docker-hosted
      port: 8000
      targetPort: 8000
    - name: docker-all
      port: 8001
      targetPort: 8001
    - name: docker-proxy
      port: 8002
      targetPort: 8002
```

### 4.2. Ingress cho từng endpoint
  - **Proxy** (docker-proxy.gitlabonlinecom.click):
```text
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nexus-docker-proxy
  namespace: nexus
  annotations:
    alb.ingress.kubernetes.io/group.name: alb-apps
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/success-codes: "200-401"
    alb.ingress.kubernetes.io/subnets: <subnet-ids>
spec:
  ingressClassName: alb
  rules:
    - host: docker-proxy.gitlabonlinecom.click
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nexus-docker
                port:
                  number: 8002

```
  - **Internal hosted** (docker-internal.gitlabonlinecom.click):
```text
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nexus-docker-hosted
  namespace: nexus
  annotations:
    alb.ingress.kubernetes.io/group.name: alb-apps
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /v2/_catalog
    alb.ingress.kubernetes.io/success-codes: "200-399"
    alb.ingress.kubernetes.io/subnets: <subnet-ids>
spec:
  ingressClassName: alb
  rules:
    - host: docker-internal.gitlabonlinecom.click
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nexus-docker
                port:
                  number: 8000

```
---
## 5. Cấu hình Docker client trên Amazon Linux 2 (hoặc Jenkins agent)

/etc/docker/daemon.json:
```text
{
  "insecure-registries": [
    "docker-internal.gitlabonlinecom.click",
    "docker-proxy.gitlabonlinecom.click"
  ]
}

```

Restart Docker:
```text
sudo systemctl restart docker

```
**Test proxy:**
```text
docker login --username docker-proxy docker-proxy.gitlabonlinecom.click
docker pull docker-proxy.gitlabonlinecom.click/library/nginx:1.25

```
**Test hosted:**
```text
docker login docker-internal.gitlabonlinecom.click
docker tag nginx:1.25 docker-internal.gitlabonlinecom.click/dev-backend/nginx:1.25
docker push docker-internal.gitlabonlinecom.click/dev-backend/nginx:1.25
docker pull docker-internal.gitlabonlinecom.click/dev-backend/nginx:1.25

```
---

## 6. Phân quyền Docker theo phòng ban trên Nexus OSS

Mô hình:

- 1 repo hosted: `docker-hosted`
- Quy ước path image theo: **phòng ban / môi trường / dự án**:

```text
docker-internal.gitlabonlinecom.click/<dept>/<env>/<project>/<image>:<tag>

# Ví dụ:
docker-internal.gitlabonlinecom.click/dev-backend/test/shopping-cart/api:1.0.0
docker-internal.gitlabonlinecom.click/qa/test/shopping-cart/test-runner:20260331
```

### 6.1. Content Selector cho dev-backend

Vào **Administration → Security → Content Selectors → Create selector:**

  - Name: docker-dev-backend-selector
  - Expression:
```text
format == "docker" and path =~ "^/v2/dev-backend(/.*)?$"

```
Giải thích:

  - format == "docker": chỉ áp dụng cho repo Docker.
  - path =~ "^/v2/dev-backend(/.*)?$": chỉ match các HTTP path dạng:
    - /v2/dev-backend
    - /v2/dev-backend/...

### 6.2. Privilege từ Content Selector

Vào Security → Privileges → Create privilege → Repository Content Selector:

  - Repository: docker-hosted
  - Content Selector: docker-dev-backend-selector
  - Actions: read, browse, add, edit
  - ID: docker-hosted-dev-backend-rw
    
Privilege này cho phép read/browse/add/edit chỉ trong path dev-backend/** của repo docker-hosted.

### 6.3. Role & User cho phòng Dev-backend

Tạo Role:

  - Vào Security → Roles → Create role
    - ID: docker-dev-backend
    - Name: Docker Dev Backend
    - Privileges:
      - docker-hosted-dev-backend-rw
      - (tuỳ chọn) nx-anonymous nếu bạn muốn user này cũng có read/browse cơ bản như anonymous
        
Tạo User:

  - Vào Security → Users → Create local user
    - User ID: docker-dev-backend (hoặc dev-backend-ci)
    - Password: (mạnh, dùng cho CI)
    - Roles:
      - docker-dev-backend

### 6.4. Sử dụng trên client / Jenkins

Trên client (Amazon Linux 2 / Jenkins agent):
```text
docker login docker-internal.gitlabonlinecom.click -u docker-dev-backend

# OK: dev-backend/*
docker build -t docker-internal.gitlabonlinecom.click/dev-backend/my-app:1.0.0 .
docker push docker-internal.gitlabonlinecom.click/dev-backend/my-app:1.0.0

# Bị chặn (unauthorized) nếu push namespace khác:
docker push docker-internal.gitlabonlinecom.click/dev-frontend/my-app:1.0.0

```

Trong Jenkins:
```text
withCredentials([usernamePassword(
  credentialsId: 'nexus-docker-dev-backend',
  usernameVariable: 'REG_USER',
  passwordVariable: 'REG_PASS'
)]) {
  sh """
    echo "${REG_PASS}" | docker login docker-internal.gitlabonlinecom.click -u "${REG_USER}" --password-stdin

    docker build -t docker-internal.gitlabonlinecom.click/dev-backend/my-app:${IMAGE_TAG} .
    docker push docker-internal.gitlabonlinecom.click/dev-backend/my-app:${IMAGE_TAG}

    docker logout docker-internal.gitlabonlinecom.click
  """
}

```
  Lưu ý:  

  - Với Nexus OSS, mô hình trên chặn tốt push theo path,
  - Pull theo path vẫn có thể rộng hơn nếu bạn thêm các privilege read/browse toàn repo;
    nếu cần siết “deploy đúng prefix” trên K8s, nên kết hợp thêm policy Kyverno.
  - Giới hạn: để login & browse repo, Nexus thường yêu cầu thêm 1 số privilege read/browse view-level → rất khó chặn pull theo path tuyệt đối bằng Nexus OSS (như đã phân tích).
  - Giải pháp bổ sung: dùng Kyverno/OPA trên K8s để giới hạn image prefix theo namespace.
---

## 7. So sánh nhanh: Nexus OSS vs Nexus Pro (liên quan Docker & RBAC)

Bảng dưới tập trung vào các điểm ảnh hưởng trực tiếp tới thiết kế Docker Registry & phân quyền (RBAC) trong kiến trúc CI/CD:

| Tính năng / Khía cạnh                         | Nexus OSS (Community)                                                           | Nexus Pro (Thương mại)                                                     |
|----------------------------------------------|---------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| **Loại repo Docker**                         | Hỗ trợ: `docker (hosted)`, `docker (proxy)`, `docker (group)`                  | Giống OSS                                                                  |
| **Deploy (push) vào group repo**             | **KHÔNG hỗ trợ** – push vào group bị chặn, chỉ dùng group để pull              | **CÓ hỗ trợ** – cho phép push trực tiếp vào group                          |
| **RBAC (User / Role / Privilege)**           | Có, nhưng cấu hình chi tiết bằng Privilege ID; dễ “dính” quyền rộng ngoài ý   | Có, UI & tính năng RBAC đầy đủ hơn, hỗ trợ use case enterprise tốt hơn    |
| **Content Selector (lọc theo path)**         | Có; dùng CSEL + Repository Content Selector; áp dụng được cho Docker           | Có; tích hợp sâu hơn, dễ kết hợp với policy nâng cao                      |
| **Chặn push theo path (namespace)**          | Làm được (ví dụ chỉ cho phép push `dev-backend/**`)                            | Làm được; kết hợp tốt với các tính năng RBAC khác                         |
| **Chặn pull theo path (chặt tuyệt đối)**     | Khó / giới hạn – do cần `read/browse` view-level để login; khó cấm chỉ 1 prefix| Làm tốt hơn với cơ chế RBAC & policy nâng cao                             |
| **Tích hợp scan/policy (security)**          | Không có IQ/Policy server kèm; phải dùng tool ngoài (Trivy, Black Duck, …)     | Có thể tích hợp **IQ Server** (policy, license, vuln scan, gate build)    |
| **High Availability / Cluster**              | Không – 1 node (single instance)                                               | Có – hỗ trợ HA, multi-node, replication                                   |
| **Quản lý & audit multi-team / multi-project** | Cơ bản; phải tự tổ chức repo/role/selector thủ công                            | Có tính năng & UI giúp quản lý nhiều team/dự án dễ hơn                    |
| **Support chính thức từ Sonatype**           | Không (dùng community channel)                                                 | Có (support enterprise, SLA, tư vấn)                                      |
| **Giá / License**                             | Miễn phí (OSS, Apache 2.0)                                                     | Trả phí license + support                                                  |

**Kết luận sử dụng trong kiến trúc này:**

- **Nexus OSS**:  
  - Phù hợp nếu:
    - Muốn tiết kiệm chi phí license,
    - Số team / dự án không quá lớn,
    - Chấp nhận:
      - Push được giới hạn theo path (nhờ Content Selector),
      - Pull theo path khó siết tuyệt đối → bù lại bằng policy ở K8s (Kyverno, OPA).
- **Nexus Pro**:  
  - Nên cân nhắc khi:
    - Có nhiều team/phòng ban, yêu cầu phân quyền & audit chặt,
    - Muốn đơn giản hoá CI/CD (push thẳng vào group),
    - Cần tính năng security/policy/HA ở mức enterprise.

---

### 8. Gợi ý khi tích hợp vào CI/CD (Jenkins)

  - Pipeline base-image (DevOps/SEC):
    - Pull từ docker-proxy.gitlabonlinecom.click (repo proxy)
    - Scan Trivy/Black Duck
    - Push sang docker-internal.gitlabonlinecom.click/dev-backend/... (repo hosted)
      
  - Pipeline ứng dụng:
    - Chỉ dùng base đã “approved” ở hosted.
    - Chỉ push/pull vào docker-internal.gitlabonlinecom.click/<dept>/... bằng user/role phòng ban.
