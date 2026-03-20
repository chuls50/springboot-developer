# Architecture Reference

## Table of Contents

1. [Layered Architecture](#layered-architecture)
2. [Hexagonal / Ports & Adapters](#hexagonal--ports--adapters)
3. [Multi-Module Maven](#multi-module-maven)
4. [Local Development with Docker Compose](#local-development-with-docker-compose)
5. [Event-Driven Patterns (In-Process)](#event-driven-patterns-in-process)
6. [Database Migration Strategy](#database-migration-strategy)

---

## Layered Architecture

The right default for most applications. Clear boundaries, minimal ceremony.

```
┌────────────────────────────────┐
│         Controller Layer        │  HTTP in/out · DTOs · @RestController
├────────────────────────────────┤
│          Service Layer          │  Business logic · @Service · @Transactional
├────────────────────────────────┤
│        Repository Layer         │  DB access · JpaRepository · @Repository
├────────────────────────────────┤
│         Entity / Domain         │  @Entity · value objects · enums
└────────────────────────────────┘
```

**Rules that prevent the most common mistakes:**

- Each layer calls only the layer directly below it. Controllers don't touch repositories.
- Entities never leave the service layer. Always map to DTOs before returning to controllers.
- No business logic in controllers or repositories. Controllers route; repositories fetch.

---

## Hexagonal / Ports & Adapters

Worth the added structure when your domain logic is complex enough to outlive the current framework, or when you need to test business rules in total isolation from infrastructure.

```
com.example.myapp/
├── domain/                        ← zero framework dependencies
│   ├── model/                     ← pure domain objects and value types
│   ├── port/
│   │   ├── in/                    ← use-case interfaces (what the app can do)
│   │   └── out/                   ← output port interfaces (what the app needs)
│   └── service/                   ← domain services implementing use cases
├── adapter/
│   ├── in/
│   │   └── web/                   ← @RestController maps HTTP → use case
│   └── out/
│       ├── persistence/           ← JPA implementations of output ports
│       └── messaging/             ← Kafka/SQS adapters
└── config/                        ← Spring wiring and bean definitions
```

**The critical constraint:** Nothing in `domain/` imports Spring or JPA. It tests with plain JUnit, no `@SpringBootTest`.

```java
// domain/port/in — defines what callers can do
public interface PlaceOrderUseCase {
    Order placeOrder(PlaceOrderCommand command);
}

// domain/port/out — defines what the domain needs (implemented by adapters)
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
}

// adapter/in/web — wires HTTP to the use case
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/orders")
public class OrderController {
    private final PlaceOrderUseCase placeOrderUseCase;

    @PostMapping
    public ResponseEntity<OrderResponse> place(@Valid @RequestBody PlaceOrderRequest req) {
        Order order = placeOrderUseCase.placeOrder(new PlaceOrderCommand(req.customerId(), req.items()));
        return ResponseEntity.ok(OrderResponse.from(order));
    }
}
```

---

## Multi-Module Maven

Adds value when teams own different modules independently, or when you want hard compile-time enforcement of layer boundaries.

```
my-platform/
├── pom.xml                    ← parent aggregator
├── common/                    ← shared DTOs, exceptions, utilities
│   └── pom.xml
├── user-service/              ← standalone Spring Boot app
│   └── pom.xml
└── order-service/             ← standalone Spring Boot app
    └── pom.xml
```

### Parent pom.xml

```xml
<groupId>com.example</groupId>
<artifactId>my-platform</artifactId>
<version>1.0.0-SNAPSHOT</version>
<packaging>pom</packaging>

<modules>
  <module>common</module>
  <module>user-service</module>
  <module>order-service</module>
</modules>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.3.5</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### Child service pom.xml

```xml
<parent>
  <groupId>com.example</groupId>
  <artifactId>my-platform</artifactId>
  <version>1.0.0-SNAPSHOT</version>
</parent>
<artifactId>order-service</artifactId>

<dependencies>
  <dependency>
    <groupId>com.example</groupId>
    <artifactId>common</artifactId>
    <version>${project.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

---

## Local Development with Docker Compose

Spring Boot 3.1+ auto-starts and auto-stops Docker Compose services when you run the app locally. Add the dependency, commit a `compose.yml`, and you're done.

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-docker-compose</artifactId>
  <scope>runtime</scope>
  <optional>true</optional>
</dependency>
```

```yaml
# compose.yml — checked into source control
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

Spring Boot detects `compose.yml` at startup, starts the services if they're not running, and connects automatically — no manual `spring.datasource.url` needed during local dev.

---

## Event-Driven Patterns (In-Process)

Use Spring Events for loosely coupling components within the same JVM. For cross-service events, prefer Kafka (see `references/integrations.md`).

### Publishing events

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order placeOrder(PlaceOrderCommand cmd) {
        Order order = // ... create and save order
        // Event is published AFTER the transaction commits thanks to @TransactionalEventListener
        eventPublisher.publishEvent(new OrderPlacedEvent(order.getId(), order.getUserId()));
        return order;
    }
}
```

### Listening to events

```java
@Component
@Slf4j
public class NotificationListener {

    // @TransactionalEventListener fires after the publishing transaction commits
    // Use BEFORE_COMMIT phase if the handler needs to participate in the same transaction
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async("taskExecutor")   // run in background — don't block the publishing thread
    public void onOrderPlaced(OrderPlacedEvent event) {
        log.info("Sending welcome email for order {}", event.orderId());
        emailService.sendOrderConfirmation(event.userId(), event.orderId());
    }
}
```

**Why `@TransactionalEventListener` over `@EventListener`?**
`@EventListener` fires immediately, even if the transaction later rolls back. `@TransactionalEventListener` with `AFTER_COMMIT` only fires if the transaction succeeded.

---

## Database Migration Strategy

| `ddl-auto` value | When to use                                               |
| ---------------- | --------------------------------------------------------- |
| `create-drop`    | Throwaway local testing only                              |
| `update`         | Never — silently drops columns, diverges from prod        |
| `validate`       | Production, staging, CI — fails fast if schema mismatches |
| `none`           | When Flyway/Liquibase fully owns the schema               |

### Flyway naming convention

```
V{version}__{description}.sql     ← versioned — run once in order
R__{description}.sql              ← repeatable — re-run when checksum changes (views, functions)
U{version}__{description}.sql     ← undo — requires Flyway Teams
```

### Handling column renames (safe pattern)

Never rename a column in a single migration if the app is deployed continuously — the old app will break. Use expand-contract:

1. `V5__add_full_name_column.sql` — add the new column
2. Deploy app that writes to both old and new columns
3. `V6__backfill_full_name.sql` — backfill data
4. Deploy app that reads from new column only
5. `V7__drop_first_name_last_name.sql` — drop old columns
