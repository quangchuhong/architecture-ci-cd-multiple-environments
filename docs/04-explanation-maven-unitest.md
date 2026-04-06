## Maven Test & Unit Test Strategy

### 1. Vai trò Maven test trong CI

- Maven chịu trách nhiệm:
  - Biên dịch source Java,
  - Chạy tất cả test trong `src/test/java`,
  - Đóng gói JAR/WAR.
- Trong pipeline Jenkins:
  - **Chỉ khi test pass** mới tiếp tục build Docker, scan Trivy, push Nexus, update GitOps, deploy ArgoCD.
- Lệnh khuyến nghị trong CI:
  - `mvn -B clean package`  
    → build + test + package (không skip test).

- Spring Boot  

  - Là framework ứng dụng Java (dựng app, REST API, data, security…).
  - Nó lo phần chạy ứng dụng: DI, cấu hình Bean, web server (Tomcat), JPA, Security…
    
- Mockito  

  - Là thư viện test (framework mock) dùng trong unit test.
  - Dùng để tạo mock/stub cho dependency (repository, service, client…) khi test logic của 1 class.
  - Không chạy app, không thay thế Spring; chỉ dùng trong src/test/java.
    
- Trong project Spring Boot:

  - Spring Boot: bạn dùng để viết service/controller/repo.
  - Mockito: bạn dùng trong test (thường kết hợp JUnit) để test service/repo mà không cần khởi động Spring.

---

### 2. Lifecycle Maven liên quan đến test

- `compile`  
  - Biên dịch `src/main/java/**` → `target/classes/`.
- `test`  
  - Biên dịch `src/test/java/**` → `target/test-classes/`.
  - Chạy toàn bộ test (JUnit/Mockito/Spring Test).
- `package`  
  - Thực hiện `validate → compile → test → package`.
  - Sinh JAR/WAR trong `target/`.

> **Lưu ý:**  
> - `mvn package` đã bao gồm chạy test.  
> - `-DskipTests=true` sẽ **bỏ qua** việc chạy test → chỉ nên dùng tạm thời khi chưa có test.

---

### 3. Các loại test trong project

Trong `pom.xml` (dependencies test):

- `spring-boot-starter-test` (scope `test`):
  - Spring Boot Test (`@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`),
  - JUnit 5,
  - Mockito core, AssertJ…
- `spring-security-test` (scope `test`):
  - Test Spring Security.
- `mockito-inline` (scope `test`):
  - Mockito inline (hỗ trợ mock final/static trong một số case).

→ Project sẵn sàng cho cả:

- **Unit test thuần với Mockito**,  
- **Spring Test** cho các flow quan trọng.

---

### 4. Unit test với Mockito (không Spring)

Mục tiêu:

- Test logic của 1 class/service trong isolation,
- Không cần khởi động Spring Boot, không cần DB thật.

Mẫu chung:

```java
@ExtendWith(MockitoExtension.class)
class SomeServiceTest {

    @InjectMocks
    SomeService service;

    @Mock
    SomeRepository repo;

    @Test
    void testLogic() {
        when(repo.find(...)).thenReturn(...);

        var result = service.doSomething();

        assertEquals(..., result);
        verify(repo).find(...);
    }
}
```

Trong dự án:

  - CollectionAccountLinkageServiceTest
  - CollectionBillingServiceTest
  - QuartzJobServiceTest
  - CollectionReconcileAppTest (test main() + config Tomcat/SSL)
    
đều là Mockito unit test, chạy rất nhanh, không dùng Spring context.
---

### 5. Spring Test (ít hơn, cho flow/config quan trọng)

Khi cần test:

  - wiring bean,
  - mapping controller,
  - query JPA,
  - security…
    
có thể dùng:

  - @SpringBootTest – full/near-full context (flow nhiều layer).
  - @WebMvcTest – web layer (controller, filter, advice).
  - @DataJpaTest – JPA + DB in-memory (H2).

Mẫu:
```java
@SpringBootTest
class SomeFlowIntegrationTest {
    @Autowired
    SomeService service;
}

@WebMvcTest(SomeController.class)
class SomeControllerTest {
    @Autowired
    MockMvc mockMvc;
    @MockBean
    SomeService service;
}

@DataJpaTest
class SomeRepositoryTest {
    @Autowired
    SomeRepository repo;
}

```

Tỷ lệ khuyến nghị:

  - 80–90% test = unit test + Mockito (nhanh, nhiều case nhỏ).
  - 10–20% test = Spring Test (chọn flow/config quan trọng để cover).

---

### 6. Test data & constant

Dự án hiện có các class chứa constant và JSON mẫu phục vụ test (không phải test case):

- ConstantTest:
  - profilesActive, jobId, dateRequest
  - JSON config report (configReport, configJob),
  - JSON billing, billingFail, billingJsonMappingFail…
- Các JSON reportConfig trong test Quartz, report…
  
Cách dùng:

- Import constant trong test để:
  - parse JSON làm input,
  - test mapping, generate report, xử lý điều kiện phức tạp.
    
Ví dụ:

```java
import static com.tcb.h2h.recon.collection.ConstantTest.billing;

@Test
void shouldParseBillingJson() {
    // parse billing JSON, assert mapping/output...
}

```
---

### 7. Tích hợp vào Jenkins Pipeline

Trong Jenkins (container maven):

  - Stage Maven nên là:

```text
mvn -B clean package

```

- Pipeline hiện tại:
  
  1. Checkout + tính GIT_SHORT_SHA.
  2. Maven build + test (unit & Spring Test).
  3. SonarQube scan + Quality Gate.
  4. Đóng gói JAR.
  5. Build Docker, save TAR, upload Nexus RAW.
  6. Trivy scan image từ TAR.
  7. Push image lên Nexus Docker.
  8. Update GitOps + ArgoCD deploy (Test → Staging), có manual approval + rollback.
     
Kết luận:

  - Maven test (unit + Spring Test) là lớp bảo vệ đầu tiên trong CI.
  - Chỉ sau khi mvn test/mvn package pass, pipeline mới tiếp tục build/push/deploy.
  - Mockito được dùng để test nhanh logic nghiệp vụ, Spring Test dùng cho một số flow/config quan trọng.
