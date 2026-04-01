# Guide: Cấu hình Nexus OSS cho Maven/Java & Python + test trên Amazon Linux 2

## 1. Chuẩn bị chung

Giả sử Nexus OSS đã chạy OK với URL:

- UI: `http://nexus.gitlabonlinecom.click/`

Trên **Nexus** ta sẽ tạo:

- Maven:
  - `maven-central`  (proxy)
  - `maven-releases` (hosted)
  - `maven-snapshots` (hosted)
  - `maven-all`      (group)
- Python:
  - `pypi-proxy`   (proxy)
  - `pypi-hosted`  (hosted)
  - `pypi-all`     (group)

Trên **client Amazon Linux 2** ta cấu hình Maven & pip để dùng Nexus.

---

## 2. Cấu hình Maven/Java trên Nexus OSS

### 2.1. Tạo repo Maven

Vào Nexus UI → **Administration → Repositories**:

1. **Maven Central proxy**

- Create repository → `maven2 (proxy)`
  - Name: `maven-central`
  - Remote storage: `https://repo1.maven.org/maven2/`
- Create.

2. **Maven hosted nội bộ**

- Create → `maven2 (hosted)`:
  - Name: `maven-releases`
  - Version policy: `Release`
- Create → `maven2 (hosted)`:
  - Name: `maven-snapshots`
  - Version policy: `Snapshot`

3. **Maven group**

- Create → `maven2 (group)`:
  - Name: `maven-all`
  - Members:
    - `maven-releases`
    - `maven-snapshots`
    - `maven-central`

Kết quả:

- Proxy: `http://nexus.gitlabonlinecom.click/repository/maven-central/`
- Hosted: `http://nexus.gitlabonlinecom.click/repository/maven-releases/`, `.../maven-snapshots/`
- Group: `http://nexus.gitlabonlinecom.click/repository/maven-all/`

---

## 3. Cấu hình Maven/Java trên client Amazon Linux 2

### 3.1. Cài Java & Maven

```bash
sudo amazon-linux-extras install java-openjdk11 -y
sudo yum install maven -y

java -version
mvn -version
```

### 3.2. Cấu hình settings.xml để dùng Nexus

Tạo (hoặc sửa) file ~/.m2/settings.xml:

```text
mkdir -p ~/.m2
cat > ~/.m2/settings.xml << 'EOF'
<settings>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>http://nexus.gitlabonlinecom.click/repository/maven-public/</url>
    </mirror>
  </mirrors>

  <profiles>
    <profile>
      <id>nexus</id>
      <repositories>
        <repository>
          <id>central</id>
          <url>http://nexus.gitlabonlinecom.click/repository/maven-public/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>nexus</activeProfile>
  </activeProfiles>
</settings>
EOF

```
### 3.3. Test Maven với Nexus

Tạo project test:
```text
mvn -B archetype:generate \
  -DgroupId=com.example \
  -DartifactId=nexus-maven-test \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false

cd nexus-maven-test
mvn clean package

```
Quan sát:

  - Maven download dependency qua Nexus (URL .../maven-public/...).

---

## 4. Cấu hình Python (PyPI) trên Nexus OSS

### 4.1. Tạo repo PyPI

Trong Nexus UI → **Repositories**:

  1. PyPI proxy
     
    - Create → pypi (proxy)
      - Name: pypi-proxy
      - Remote storage: https://pypi.org/
      
  2. PyPI hosted (internal)
    - Create → pypi (hosted)
      - Name: pypi-hosted
        
  3. PyPI group
    - Create → pypi (group)
      - Name: pypi-all
      - Members:
        - pypi-hosted
        - pypi-proxy
          
Endpoint group:
```text
http://nexus.gitlabonlinecom.click/repository/pypi-all/simple

```
---
## 5. Cấu hình Python/pip trên client Amazon Linux 2

### 5.1. Cài Python & pip
```text
sudo yum install python3 python3-pip -y

python3 --version
pip3 --version

```
### 5.2. Cấu hình pip dùng Nexus

Tạo ~/.pip/pip.conf
```text
mkdir -p ~/.pip
cat > ~/.pip/pip.conf << 'EOF'
[global]
index-url = http://nexus.gitlabonlinecom.click/repository/pypi-all/simple
trusted-host = nexus.gitlabonlinecom.click
EOF

```

### 5.3. Test pip với Nexus
```text
pip3 install requests
pip3 install flask

```
Quan sát:

  - pip download package từ http://nexus.gitlabonlinecom.click/repository/pypi-all/simple/...

---

## 6. Upload Python package lên Nexus (PyPI hosted)
   
### 6.1. Cài công cụ build & upload
```bash
pip3 install build twine

```
### 6.2. Tạo project Python đơn giản
```bash
mkdir pypi-nexus-test
cd pypi-nexus-test

cat > setup.py << 'EOF'
from setuptools import setup, find_packages

setup(
    name="pypi-nexus-test",
    version="0.1.0",
    packages=find_packages(),
)
EOF

mkdir pypi_nexus_test
touch pypi_nexus_test/__init__.py

```
### 6.3. Build package
```bash
python3 -m build
# tạo dist/*.tar.gz, *.whl

```
### 6.4. Upload lên pypi-hosted
```text
twine upload \
  --repository-url http://nexus.gitlabonlinecom.click/repository/pypi-hosted/ \
  dist/*

```

Nếu Nexus yêu cầu auth, twine sẽ hỏi username/password tương ứng user Nexus cho PyPI.

Sau đó cấu trúc group pypi-all đã chứa cả hosted + proxy, pip client sẽ:

  - Lấy package nội bộ từ pypi-hosted,
  - Lấy package bên ngoài từ pypi-proxy nếu hosted không có.
    
---

## 7. YUM (rpm) qua Nexus

Tùy yêu cầu:

  - Nếu bạn chỉ muốn cache 1 số repo public (EPEL, CentOS, Alma…), dùng YUM proxy.
  - Với Amazon Linux chính chủ, repo có dạng đặc biệt; proxy được nhưng thường không bắt buộc.
    Dưới đây là pattern chung.
    
### 7.1. Tạo repo YUM proxy trên Nexus

Vào Nexus:

  - Create → yum (proxy) (hoặc yum (proxy hosted) tùy version)
    - Name: yum-proxy-epel (ví dụ)
    - Remote storage: URL của repo YUM bạn muốn proxy, ví dụ:
      - https://download.fedoraproject.org/pub/epel/9/Everything/x86_64/(chỉ là ví dụ, tùy OS bạn dùng)
        
Sau khi tạo xong, bạn sẽ có URL:
```text
http://nexus.gitlabonlinecom.click/repository/yum-proxy-epel/

```

### 7.2. Cấu hình YUM trên Amazon Linux dùng Nexus
Trên client:

Tạo file /etc/yum.repos.d/nexus-epel.repo:
```text
[nexus-epel]
name=Nexus EPEL Proxy
baseurl=http://nexus.gitlabonlinecom.click/repository/yum-proxy-epel/
enabled=1
gpgcheck=0

```
Tắt hoặc ưu tiên thấp các repo EPEL gốc nếu bạn muốn tất cả đi qua Nexus.

Cập nhật metadata:
```bash
sudo yum clean all
sudo yum makecache

```
Test:
```text
sudo yum install htop

```
Nếu gói có trong EPEL, yum sẽ kéo qua Nexus proxy.

Lưu ý:  

```text
Với Amazon Linux 2/2023, các repo hệ điều hành (amzn2-core, amazonlinux) có thể không đơn giản để proxy, vì có auth/region/metadata riêng. 
Thường bạn không cần proxy các repo này trừ khi môi trường air-gapped.
Proxy EPEL/third-party qua Nexus là phổ biến hơn.
```
---

## 8. Cấu hình RAW repository trên Nexus OSS để upload / download file, artifact

RAW repo dùng để lưu **bất kỳ loại file nào**: `.zip`, `.rar`, `.tar.gz`, `.jar`, `.war`, `.exe`, `.msi`, `.sh`, `.ps1`, report, log…  
Rất tiện để làm “shared artifacts” / “installer store” nội bộ.

---

### 8.1. Tạo RAW hosted repo trong Nexus

Trong Nexus UI:

1. Vào **Administration → Repositories → Create repository → raw (hosted)**  
2. Điền:
   - **Name**: `raw-artifacts` (hoặc `files-internal`, `aws-tools`, `installers`…)
   - (Các option khác để mặc định)
3. Bấm **Create**.

Khi đó endpoint:

```text
http://nexus.gitlabonlinecom.click/repository/raw-artifacts/
```

### 8.2. Upload file bằng curl (CI/CD, script)

Cú pháp chung:
```text
curl -u <user>:<pass> \
  --upload-file <local-file> \
  "http://nexus.gitlabonlinecom.click/repository/raw-artifacts/<path-trong-repo>/<file-name>"

```

Ví dụ upload file .zip:

```text
curl -u admin:MyPassword \
  --upload-file awscliv2.zip \
  "http://nexus.gitlabonlinecom.click/repository/raw-artifacts/aws/awscliv2.zip"

```
Upload artifact build từ Jenkins (ví dụ target/app-1.0.0.jar):
```text
curl -u jenkins-ci:*** \
  --upload-file target/app-1.0.0.jar \
  "http://nexus.gitlabonlinecom.click/repository/raw-artifacts/dev-backend/shopping-cart/app-1.0.0.jar"

```
_Lưu ý bash:

Nếu password có ký tự đặc biệt như !, nên dùng 'user:pass' trong single-quote hoặc dùng file ~/.netrc để tránh lỗi shell.
_
