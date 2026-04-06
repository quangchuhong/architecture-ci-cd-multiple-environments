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
