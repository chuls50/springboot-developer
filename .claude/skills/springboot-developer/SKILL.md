---
name: spring-boot-development
description: >
  Use this skill whenever the user is building, scaffolding, debugging, or improving a Spring Boot
  application. Triggers include any mention of "Spring Boot", "Spring application", "REST API with
  Java", "@SpringBootApplication", "Spring Data JPA", "Spring Security", "application.properties",
  "application.yml", "Maven", "Gradle", "microservice in Java", "Spring controller", "Spring service",
  Hibernate, Flyway, Liquibase, Testcontainers, Actuator, Micrometer, or any request to create Java
  backend APIs, configure Spring beans, write Spring tests, set up Spring Boot projects, fix Spring
  errors, add authentication/JWT to a Java app, or work with Spring caching/scheduling/async.
  Use this skill even if the user says "Java REST API" without naming Spring — Spring Boot is the
  default Java web framework and should be assumed. Also trigger for adding features like pagination,
  validation, caching, Kafka, or resilience to existing Spring apps, and for diagnosing Spring errors
  like NoSuchBeanDefinitionException, LazyInitializationException, circular dependencies, and N+1 queries.
---

# Spring Boot Development

Comprehensive guide for building production-grade Spring Boot 3.x applications. Covers everything
from scaffolding through testing, observability, and deployment readiness.

> **Version baseline:** Spring Boot 3.3+ / Java 21. Java 17 is the minimum for Boot 3.x.
> Note: Spring Boot 4.0 arrived November 2025 — core patterns here still apply; `javax.*` → `jakarta.*` migration was the major 3.0 change.

## Quick Reference

| Task                                        | Section                                                  |
| ------------------------------------------- | -------------------------------------------------------- |
| New project / project structure             | [Scaffolding](#scaffolding)                              |
| REST controllers, DTOs, error handling      | [REST Layer](#rest-layer)                                |
| JPA entities, repositories, service layer   | [Data Layer](#data-layer)                                |
| Database migrations (Flyway)                | [Migrations](#database-migrations-flyway)                |
| Caching, async, scheduling                  | [Cross-cutting Concerns](#cross-cutting-concerns)        |
| JWT security                                | [Security](#security)                                    |
| Testing (unit, integration, Testcontainers) | [Testing](#testing)                                      |
| Actuator, observability, structured logging | [Production Readiness](#production-readiness)            |
| Virtual threads, modern Java features       | [Modern Java & Boot 3.x](#modern-java--boot-3x-features) |
| Common errors & fixes                       | [Troubleshooting](#troubleshooting)                      |
| Architecture deep-dive                      | `references/architecture.md`                             |
| Full JWT + OAuth2 security setup            | `references/security.md`                                 |
| Resilience4j, WebClient, Kafka patterns     | `references/integrations.md`                             |

---

## Scaffolding

### Maven `pom.xml` starter

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.3.5</version>   <!-- Check https://spring.io/projects/spring-boot for latest 3.x -->
</parent>

<properties>
  <java.version>21</java.version>
</properties>

<dependencies>
  <dependency><groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId></dependency>
  <dependency><groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId></dependency>
  <!-- Add your JDBC driver, e.g. postgresql -->
  <dependency><groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId><optional>true</optional></dependency>
  <dependency><groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
  <dependency><groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId><scope>test</scope></dependency>
</dependencies>
```

### Standard package layout

```
com.example.myapp/
├── MyAppApplication.java          ← @SpringBootApplication entry point
├── config/                        ← @Configuration beans (security, cache, async, etc.)
├── controller/                    ← @RestController — thin HTTP routing only
├── service/                       ← @Service — business logic, owns @Transactional boundary
├── repository/                    ← @Repository / JpaRepository interfaces
├── entity/                        ← @Entity / @Table domain objects
├── dto/                           ← Request/Response records — never expose entities directly
├── mapper/                        ← MapStruct mappers (entity ↔ DTO)
└── exception/                     ← Custom exceptions + @RestControllerAdvice handler
```

### Entry point

```java
@SpringBootApplication
public class MyAppApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyAppApplication.class, args);
    }
}
```

---

## REST Layer

### Controller pattern — keep controllers thin

Controllers route HTTP traffic. All logic belongs in services.

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public ResponseEntity<Page<ProductResponse>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(productService.findAll(PageRequest.of(page, size)));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> get(@PathVariable Long id) {
        return ResponseEntity.ok(productService.findById(id));
    }

    @PostMapping
    public ResponseEntity<ProductResponse> create(@Valid @RequestBody CreateProductRequest req) {
        ProductResponse created = productService.create(req);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}").buildAndExpand(created.id()).toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<ProductResponse> update(
            @PathVariable Long id, @Valid @RequestBody UpdateProductRequest req) {
        return ResponseEntity.ok(productService.update(id, req));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Use Java records for DTOs

Records are immutable, concise, and serialise cleanly with Jackson.

```java
// Request DTO — validation annotations live here, not on the entity
public record CreateProductRequest(
        @NotBlank @Size(max = 255) String name,
        @NotNull @DecimalMin("0.01") BigDecimal price,
        @NotNull Long categoryId
) {}

// Response DTO — only expose what clients need
public record ProductResponse(Long id, String name, BigDecimal price, String categoryName) {}
```

### Global exception handler

Centralise error handling so controllers stay clean. Return consistent `ErrorResponse` shapes.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage()).toList();
        return ResponseEntity.badRequest()
                .body(new ErrorResponse("VALIDATION_ERROR", String.join("; ", errors)));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex, WebRequest request) {
        // Log here; never expose stack traces to clients
        return ResponseEntity.internalServerError()
                .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
    }
}

public record ErrorResponse(String code, String message) {}
```

---

## Data Layer

### Entity

```java
@Entity
@Table(name = "products")
@Getter @Setter @NoArgsConstructor
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 255)
    private String name;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    // ALWAYS lazy on associations — fetch eagerly only when you explicitly need it
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(nullable = false)
    private LocalDateTime updatedAt;
}
```

### Repository

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    // Spring derives the query automatically from the method name
    List<Product> findByNameContainingIgnoreCase(String name);

    // Use @EntityGraph to avoid N+1 when you know you'll need an association
    @EntityGraph(attributePaths = {"category"})
    Optional<Product> findWithCategoryById(Long id);

    // JPQL for anything complex — avoid native SQL unless you must
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max")
    Page<Product> findByPriceRange(
            @Param("min") BigDecimal min, @Param("max") BigDecimal max, Pageable pageable);
}
```

### Service layer — transactions live here

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)   // read-only by default; selectively override with @Transactional
public class ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper mapper;

    public Page<ProductResponse> findAll(Pageable pageable) {
        return productRepository.findAll(pageable).map(mapper::toResponse);
    }

    public ProductResponse findById(Long id) {
        return productRepository.findById(id)
                .map(mapper::toResponse)
                .orElseThrow(() -> new EntityNotFoundException("Product not found: " + id));
    }

    @Transactional
    public ProductResponse create(CreateProductRequest req) {
        Product product = mapper.toEntity(req);
        return mapper.toResponse(productRepository.save(product));
    }

    @Transactional
    public ProductResponse update(Long id, UpdateProductRequest req) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("Product not found: " + id));
        mapper.updateEntity(req, product);      // MapStruct updates in-place
        return mapper.toResponse(product);      // no explicit save() — dirty-checking handles it
    }

    @Transactional
    public void delete(Long id) {
        if (!productRepository.existsById(id)) {
            throw new EntityNotFoundException("Product not found: " + id);
        }
        productRepository.deleteById(id);
    }
}
```

**Why no `save()` on update?** Within a transaction, Hibernate tracks dirty state automatically. Calling `save()` on an already-managed entity is harmless but redundant.

### N+1 rule of thumb

- Every `@ManyToOne` / `@OneToMany` → `FetchType.LAZY`
- When you need the association: use `@EntityGraph` or JOIN FETCH in the query
- Never call a repository inside a loop — always fetch a collection and process it

---

## Database Migrations (Flyway)

Never rely on `spring.jpa.hibernate.ddl-auto: update` in any environment other than a throwaway local DB. Use Flyway for all schema changes.

```
src/main/resources/db/migration/
├── V1__create_categories_table.sql
├── V2__create_products_table.sql
└── V3__add_product_stock_column.sql
```

```sql
-- V2__create_products_table.sql
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    price       NUMERIC(10,2) NOT NULL,
    category_id BIGINT NOT NULL REFERENCES categories(id),
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_category ON products(category_id);
```

Flyway runs automatically on startup when it's on the classpath. In `application.yml`:

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
  jpa:
    hibernate:
      ddl-auto: validate # validate schema matches entities; fail fast if not
```

---

## Configuration & Profiles

### Prefer YAML, externalise secrets

```yaml
# application.yml — shared defaults, no secrets
spring:
  application:
    name: my-app
  datasource:
    url: ${DB_URL} # always from environment
    username: ${DB_USER}
    password: ${DB_PASS}
  jpa:
    show-sql: false
    properties:
      hibernate.format_sql: true
      hibernate.default_batch_fetch_size: 20 # prevents N+1 for collections

server:
  port: 8080
  shutdown: graceful # finish in-flight requests on SIGTERM (Boot 2.3+)

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true # /actuator/health/liveness and /readiness for Kubernetes
  server:
    port: 8081 # run Actuator on a separate internal port
```

### Typed configuration with `@ConfigurationProperties`

Prefer this over scattering `@Value` annotations throughout the codebase.

```java
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProperties(
    @NotBlank String jwtSecret,
    @Min(3600) long jwtExpirationSeconds,
    Cors cors
) {
    public record Cors(List<String> allowedOrigins) {}
}

// Register it:
@SpringBootApplication
@ConfigurationPropertiesScan
public class MyAppApplication { ... }
```

### Multi-profile layout

```
src/main/resources/
├── application.yml            ← shared defaults
├── application-dev.yml        ← local overrides (H2 or Docker DB, debug SQL, etc.)
└── application-prod.yml       ← production (loaded from env/vault, never committed with secrets)
```

Activate: `SPRING_PROFILES_ACTIVE=dev` or `--spring.profiles.active=prod`

---

## Cross-cutting Concerns

### Caching

```java
// 1. Enable caching
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        // Use Redis in production: RedisCacheManager.builder(...)
        return new ConcurrentMapCacheManager("products", "categories");
    }
}

// 2. Annotate service methods
@Cacheable(value = "products", key = "#id")
public ProductResponse findById(Long id) { ... }

@CachePut(value = "products", key = "#result.id()")
@Transactional
public ProductResponse update(Long id, UpdateProductRequest req) { ... }

@CacheEvict(value = "products", key = "#id")
@Transactional
public void delete(Long id) { ... }
```

Cache keys collide across environments — always namespace them or use separate Redis DBs per environment.

### Async execution

```java
// 1. Enable async and define a thread pool (don't use the default SimpleAsyncTaskExecutor)
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(4);
        exec.setMaxPoolSize(16);
        exec.setQueueCapacity(100);
        exec.setThreadNamePrefix("async-");
        exec.initialize();
        return exec;
    }
}

// 2. Annotate service methods (must be called from another bean — self-invocation skips AOP proxy)
@Async("taskExecutor")
public CompletableFuture<Void> sendWelcomeEmail(String email) {
    emailClient.send(email, "Welcome!", "...");
    return CompletableFuture.completedFuture(null);
}
```

With virtual threads enabled (Java 21+, see [Modern Java](#modern-java--boot-3x-features)), custom thread pools become less important for I/O-bound work.

### Scheduling

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {}

@Component
@Slf4j
public class ReportScheduler {

    // fixedRate: run every N ms regardless of how long the task took
    @Scheduled(fixedRateString = "${app.report.interval-ms:60000}")
    public void generateDailyReport() { ... }

    // cron: standard cron expression — "0 0 2 * * *" = 2am every day
    @Scheduled(cron = "${app.report.cron:0 0 2 * * *}")
    public void cleanupOldData() { ... }
}
```

---

## Security

For the full JWT filter chain, `SecurityFilterChain`, `UserDetailsService`, and OAuth2 resource server pattern, read `references/security.md`. Quick checklist:

- [ ] `SecurityFilterChain` bean (not the deprecated `WebSecurityConfigurerAdapter`)
- [ ] `SessionCreationPolicy.STATELESS` — no HTTP sessions for REST APIs
- [ ] `OncePerRequestFilter` for JWT extraction and validation
- [ ] `UserDetailsService` backed by your `UserRepository`
- [ ] `BCryptPasswordEncoder` for passwords
- [ ] `CorsConfigurationSource` bean — not `@CrossOrigin` on every controller
- [ ] Actuator on a separate port with its own security rules
- [ ] CSRF disabled for stateless APIs

---

## Testing

Good tests give you confidence to change things. Prefer real dependencies over mocks where the cost is low.

### Unit test — service layer

```java
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {

    @Mock ProductRepository productRepository;
    @Mock ProductMapper mapper;
    @InjectMocks ProductService productService;

    @Test
    void findById_returnsDTO_whenProductExists() {
        Product product = new Product();
        product.setId(1L);
        ProductResponse dto = new ProductResponse(1L, "Widget", new BigDecimal("9.99"), "Tools");

        when(productRepository.findById(1L)).thenReturn(Optional.of(product));
        when(mapper.toResponse(product)).thenReturn(dto);

        assertThat(productService.findById(1L).id()).isEqualTo(1L);
        verify(productRepository).findById(1L);
    }

    @Test
    void findById_throws_whenNotFound() {
        when(productRepository.findById(99L)).thenReturn(Optional.empty());
        assertThatThrownBy(() -> productService.findById(99L))
                .isInstanceOf(EntityNotFoundException.class)
                .hasMessageContaining("99");
    }
}
```

### Repository slice test — fast, no web layer

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // use real DB with Testcontainers
@Testcontainers
class ProductRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired ProductRepository repo;

    @Test
    void findByNameContaining_isCaseInsensitive() {
        repo.save(buildProduct("Blue Widget", "5.00"));
        assertThat(repo.findByNameContainingIgnoreCase("widget")).hasSize(1);
    }
}
```

### Integration test — full HTTP stack

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class ProductControllerIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @Autowired ProductRepository productRepository;

    @BeforeEach   // keep tests isolated — wipe state between runs
    void setUp() { productRepository.deleteAll(); }

    @Test
    void createProduct_returns201WithLocation() throws Exception {
        var req = new CreateProductRequest("Gadget", new BigDecimal("19.99"), 1L);

        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
                .andExpect(status().isCreated())
                .andExpect(header().exists("Location"))
                .andExpect(jsonPath("$.name").value("Gadget"));
    }

    @Test
    void getProduct_returns404_whenNotFound() throws Exception {
        mockMvc.perform(get("/api/v1/products/9999"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.code").value("NOT_FOUND"));
    }
}
```

**Tip:** Share a static `postgres` container across all test classes to avoid the startup cost on every test file. Extract it into a base class or a JUnit 5 `@ExtendWith` extension.

### Why Testcontainers over H2?

H2 in-memory databases hide dialect differences — query behaviour, constraint enforcement, and function availability differ between H2 and Postgres/MySQL. Testcontainers runs a real image, so what passes locally will pass in CI. The cost is a one-time Docker pull; subsequent runs reuse the cached image.

---

## Production Readiness

### Actuator configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true # /actuator/health/liveness  /actuator/health/readiness
  server:
    port: 8081 # keep management endpoints off the public-facing port
```

**Never expose `env`, `beans`, or `heapdump` publicly.** If you can't run Actuator on a separate port, protect it with Spring Security scoped to a ROLE_ADMIN check.

### Custom health indicator

```java
@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {

    private final ExternalServiceClient client;

    @Override
    public Health health() {
        try {
            client.ping();
            return Health.up().withDetail("service", "payment-gateway").build();
        } catch (Exception ex) {
            return Health.down()
                    .withDetail("service", "payment-gateway")
                    .withDetail("error", ex.getMessage())
                    .build();
        }
    }
}
```

### Structured logging (Boot 3.4+)

Structured JSON logs make log aggregators (ELK, Loki, Datadog) far more powerful than plaintext scraping.

```yaml
logging:
  structured:
    format:
      console: ecs # Elastic Common Schema; use 'logstash' for ELK stacks
```

Or manually add trace IDs from Micrometer Tracing to every log line:

```java
@Slf4j
@Service
public class OrderService {
    public void processOrder(Order order) {
        log.info("Processing order orderId={} userId={}", order.getId(), order.getUserId());
        // trace/spanId are injected automatically by Micrometer if you have a tracing bridge
    }
}
```

### Observability with Micrometer

```xml
<!-- Tracing bridge (choose one) -->
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-otel</artifactId></dependency>
<!-- Metrics exporter -->
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-registry-prometheus</artifactId></dependency>
```

```java
// Instrument a method — generates both a metric AND a trace span automatically
@Observed(name = "order.processing", contextualName = "process-order")
public void processOrder(Order order) { ... }
```

### Kubernetes probes

```yaml
# deployment.yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8081 }
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8081 }
  initialDelaySeconds: 10
  periodSeconds: 5
```

Liveness = is the process alive and not deadlocked? (restart if fails)
Readiness = is the app ready to accept traffic? (stop routing if fails, but don't restart)

---

## Modern Java & Boot 3.x Features

### Virtual threads (Java 21 + Spring Boot 3.2+)

One-line change that dramatically improves throughput for I/O-bound workloads (DB calls, HTTP calls, file I/O) at the cost of nothing. Not beneficial for CPU-intensive code.

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Virtual threads replace the servlet thread pool with JVM-managed lightweight threads. You don't need to rethink your blocking code — existing `@Async`, JDBC, and WebClient code all benefit automatically.

### Declarative HTTP clients (Boot 3.1+)

```java
@HttpExchange("https://api.payments.example.com")
public interface PaymentClient {

    @GetExchange("/charges/{id}")
    ChargeResponse getCharge(@PathVariable String id);

    @PostExchange("/charges")
    ChargeResponse createCharge(@RequestBody CreateChargeRequest req);
}

// Register as a bean:
@Bean
PaymentClient paymentClient(WebClient.Builder builder) {
    WebClient webClient = builder.baseUrl("https://api.payments.example.com").build();
    return HttpServiceProxyFactory.builderFor(WebClientAdapter.create(webClient))
            .build().createClient(PaymentClient.class);
}
```

### Java records everywhere

Use records for DTOs, configuration properties, value objects, and event types. They're immutable, auto-generate `equals`/`hashCode`/`toString`, and serialise correctly with Jackson.

### Pattern matching and sealed classes for domain modelling

```java
public sealed interface PaymentResult permits PaymentResult.Success, PaymentResult.Failure {
    record Success(String transactionId, BigDecimal amount) implements PaymentResult {}
    record Failure(String errorCode, String message) implements PaymentResult {}
}

// Usage
PaymentResult result = paymentService.process(req);
String response = switch (result) {
    case PaymentResult.Success s -> "Paid: " + s.transactionId();
    case PaymentResult.Failure f -> "Failed: " + f.message();
};
```

---

## Troubleshooting

| Error                              | Likely Cause                                                 | Fix                                                                                                                    |
| ---------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `NoSuchBeanDefinitionException`    | Class not in component scan scope or missing annotation      | Check the package path; add `@Component`/`@Service`; or define an explicit `@Bean`                                     |
| `LazyInitializationException`      | Accessing a lazy-loaded association after the session closed | Add `@Transactional` to the caller, use `@EntityGraph`, or fetch in a JOIN FETCH query                                 |
| `BeanCurrentlyInCreationException` | Circular dependency                                          | Restructure — two beans shouldn't need each other at construction time. Use `@Lazy` only as a last resort              |
| `DataIntegrityViolationException`  | DB constraint violated (null, unique, FK)                    | Check nullable columns, unique indexes, FK references; validate input before persisting                                |
| `ConverterNotFoundException`       | MapStruct mapper missing or not scanned                      | Ensure the `@Mapper` class is in the component scan; check for compile errors in generated code                        |
| `HttpMessageNotReadableException`  | Malformed JSON / wrong content-type header                   | Ensure `Content-Type: application/json`; check request body shape against the DTO                                      |
| Unexpected `403 Forbidden`         | Spring Security blocking a request                           | Trace through `SecurityFilterChain` permit rules; check JWT filter ordering; enable Security debug logging temporarily |
| N+1 queries                        | Lazy association loaded per row in a loop                    | Use `@EntityGraph` / JOIN FETCH; set `hibernate.default_batch_fetch_size` for collection associations                  |
| Tests slow to start                | Full `@SpringBootTest` loading everything                    | Use `@WebMvcTest` (web layer only) or `@DataJpaTest` (JPA only) for focused unit-level tests                           |
| Port already in use                | Previous instance still running                              | `lsof -i :8080 \| grep LISTEN` then `kill <PID>`                                                                       |
| Schema validation fails on startup | Entity doesn't match DB schema                               | Run Flyway migration; check field lengths, column names, nullability                                                   |

---

## Key Principles

These aren't arbitrary rules — each has a reason.

- **DTOs, not entities, cross layer boundaries.** Exposing JPA entities in API responses couples your API schema to your persistence model and can accidentally trigger lazy-loading outside a transaction.
- **Transactions belong in the service layer.** Controllers shouldn't know about transactions; repositories shouldn't start them.
- **Validate at the boundary.** `@Valid` on controller method parameters means bad input is rejected before it reaches business logic.
- **Use constructor injection.** Field injection hides dependencies and makes classes harder to test. `@RequiredArgsConstructor` makes constructor injection concise.
- **Never hardcode secrets.** Environment variables or secrets managers only. Nothing sensitive in committed config files — not even test defaults.
- **Paginate everything unbounded.** `Page<T>` not `List<T>` for any endpoint that could return a large result set.
- **Migrations, not ddl-auto.** `validate` in non-dev environments; Flyway owns the schema.

---

## Reference Files

Read these when you need deeper coverage:

- `references/architecture.md` — Layered vs hexagonal, multi-module Maven, event-driven patterns, Docker Compose for local dev
- `references/security.md` — Full JWT implementation (jjwt 0.12.x), `SecurityFilterChain`, `UserDetailsService`, OAuth2 resource server, method-level security
- `references/integrations.md` — Resilience4j (circuit breaker, retry, rate limiter), Spring Kafka, WebClient with timeouts and retry, RestClient (Boot 3.2+)
