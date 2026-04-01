
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
    
