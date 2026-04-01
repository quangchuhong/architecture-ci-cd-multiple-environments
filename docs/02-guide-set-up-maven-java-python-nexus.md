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
