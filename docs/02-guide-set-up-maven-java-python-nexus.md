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
