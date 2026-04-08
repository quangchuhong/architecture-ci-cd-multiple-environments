## CI CD ASP.NET Core

### 1. Kiểu triển khai phổ biến cho ASP.NET Core

Hiện đại nhất là dùng ASP.NET Core chạy trên Linux container, quy trình giống hệt Java/Python/Node:

  - Code ASP.NET Core (API/Website)
  - Build & test bằng dotnet CLI
  - dotnet publish → build artifact (self-contained hoặc framework-dependent)
  - Docker build → image ASP.NET Core
  - Trivy scan → push Nexus/ECR
  - GitOps/ArgoCD deploy lên EKS (dùng Linux container).
    
Nếu bạn còn ASP.NET “cổ điển” (Full .NET Framework) chỉ chạy được trên Windows, khi đó cần Windows agent + Windows container (phức tạp hơn). Ở đây mình tập trung ASP.NET Core (chạy Linux), phù hợp với EKS/Nexus hiện tại của bạn.

---

### 2. Cấu trúc ASP.NET Core project

Ví dụ:
```text
MyAspNetApp/
  MyAspNetApp.csproj
  Program.cs
  Startup.cs (hoặc minimal API)
  Controllers/
  Models/
  Services/
  Tests/
    MyAspNetApp.Tests.csproj
    UnitTests.cs
  Dockerfile
  Jenkinsfile

```

---

### 3. CI trên Jenkins – luồng chính

#### 3.1. Checkout

  - Jenkins lấy code từ Git (GitLab), branch develop / release/* / main.
    
  - Tính GIT_SHORT_SHA để dùng làm IMAGE_TAG.
    
#### 3.2. Build & test bằng dotnet CLI

Trong container có .NET SDK (ví dụ mcr.microsoft.com/dotnet/sdk:8.0):

  1. Restore packages:
```text
dotnet restore

```
  2. Build:
```text
dotnet build --configuration Release

```
3. Test:
```text
dotnet test --configuration Release --no-build
```
  - Nếu bất kỳ test nào fail → stage fail → dừng pipeline.

#### 3.3. Publish ứng dụng

Đóng gói app ASP.NET Core ra folder publish/:
```text
dotnet publish MyAspNetApp.csproj \
  --configuration Release \
  --output ./publish
```

Thư mục ./publish chứa:

  - MyAspNetApp.dll
  - Các file runtime cần thiết.

---

### 4. Docker build cho ASP.NET Core

Dockerfile kiểu 2-stage:
```text
# Stage 1: build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY . .
RUN dotnet restore
RUN dotnet publish MyAspNetApp.csproj -c Release -o /app/publish

# Stage 2: runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .

# Port app lắng nghe, ví dụ 8080
EXPOSE 8080

ENTRYPOINT ["dotnet", "MyAspNetApp.dll"]

```
