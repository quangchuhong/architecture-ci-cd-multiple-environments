# 03. Triển khai & cấu hình Jenkins cho Test/Staging trên EKS

## 1. Mục tiêu
- Jenkins chạy trên `EKS-devops-test`, namespace `jenkins-test`.
- Dùng Kubernetes agent (podTemplate) cho build Java/Python/.NET…
- Tích hợp GitLab, SonarQube, Trivy, Nexus, ECR.
- Phục vụ CI cho **Test** và **Staging**.

## 2. Cài đặt Jenkins bằng Helm

### 2.1. Chuẩn bị namespace & Helm repo

```bash
kubectl create namespace jenkins-test

helm repo add jenkins https://charts.jenkins.io
helm repo update
