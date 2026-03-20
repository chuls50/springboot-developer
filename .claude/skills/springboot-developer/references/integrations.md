# Integration Patterns Reference

## Table of Contents

1. [Resilience4j (Circuit Breaker, Retry, Rate Limiter)](#resilience4j)
2. [WebClient (Reactive HTTP Client)](#webclient)
3. [RestClient (Spring Boot 3.2+)](#restclient-spring-boot-32)
4. [Spring Kafka](#spring-kafka)
5. [External API Integration Best Practices](#api-integration-best-practices)

---

## Resilience4j

Resilience4j provides fault tolerance patterns as decorators around any functional interface, method, or lambda. Integrates seamlessly with Spring Boot through starter packages.

### Dependencies

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot3</artifactId>
  <version>2.2.0</version>
</dependency>
```

### Circuit Breaker

Prevents cascading failures by opening the circuit after a threshold of failures. Configurable fallback logic.

**Configuration:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        registerHealthIndicator: true
        slidingWindowSize: 10 # last N calls tracked
        minimumNumberOfCalls: 5 # min calls before circuit can open
        permittedNumberOfCallsInHalfOpenState: 3 # test calls before closing
        failureRateThreshold: 50 # open if >50% fail
        waitDurationInOpenState: 30s # how long to wait before half-open
        slowCallDurationThreshold: 2s # call is "slow" if >2s
        slowCallRateThreshold: 50 # open if >50% are slow
```

**Usage:**

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentClient paymentClient;

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @TimeLimiter(name = "paymentService")
    public CompletableFuture<PaymentResponse> processPayment(PaymentRequest req) {
        return CompletableFuture.supplyAsync(() -> paymentClient.charge(req));
    }

    private CompletableFuture<PaymentResponse> paymentFallback(PaymentRequest req, Exception ex) {
        log.warn("Payment service unavailable, queuing for later: {}", ex.getMessage());
        return CompletableFuture.completedFuture(
            new PaymentResponse("QUEUED", "Payment queued due to service unavailability")
        );
    }
}
```

### Retry

Automatically retries failed calls with exponential backoff.

**Configuration:**

```yaml
resilience4j:
  retry:
    instances:
      inventoryService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - org.springframework.web.client.ResourceAccessException
          - java.net.ConnectException
        ignoreExceptions:
          - com.example.myapp.exception.BusinessException # don't retry business logic errors
```

**Usage:**

```java
@Retry(name = "inventoryService")
public InventoryResponse checkStock(Long productId) {
    return inventoryClient.getStock(productId);
}
```

### Rate Limiter

Limits the number of calls per time window. Useful for API quotas or protecting downstream services.

**Configuration:**

```yaml
resilience4j:
  ratelimiter:
    instances:
      externalApi:
        limitForPeriod: 50 # max 50 calls
        limitRefreshPeriod: 1s # per second
        timeoutDuration: 0s # 0 = fail immediately if limit exceeded; >0 = wait
```

**Usage:**

```java
@RateLimiter(name = "externalApi")
public ExternalApiResponse callExternalApi(String query) {
    return externalApiClient.search(query);
}
```

### Bulkhead

Isolates calls to limit concurrent requests. Semaphore-based (thread pool isolation without actual threads).

**Configuration:**

```yaml
resilience4j:
  bulkhead:
    instances:
      heavyOperation:
        maxConcurrentCalls: 10
        maxWaitDuration: 100ms
```

**Usage:**

```java
@Bulkhead(name = "heavyOperation")
public Report generateReport(Long customerId) {
    return reportGenerator.generate(customerId);
}
```

---

## WebClient

Non-blocking, reactive HTTP client. Spring's recommended replacement for `RestTemplate`. Supports streaming, backpressure, and declarative timeouts.

### Configuration

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder()
            .defaultHeader(HttpHeaders.USER_AGENT, "MyApp/1.0")
            .filter(logRequest())
            .filter(logResponse());
    }

    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(req -> {
            log.debug("Request: {} {}", req.method(), req.url());
            return Mono.just(req);
        });
    }

    private ExchangeFilterFunction logResponse() {
        return ExchangeFilterFunction.ofResponseProcessor(res -> {
            log.debug("Response: {}", res.statusCode());
            return Mono.just(res);
        });
    }
}
```

### Basic Usage

```java
@Service
@RequiredArgsConstructor
public class GitHubService {

    private final WebClient.Builder webClientBuilder;

    public Mono<GithubUser> getUser(String username) {
        WebClient client = webClientBuilder
            .baseUrl("https://api.github.com")
            .build();

        return client.get()
            .uri("/users/{username}", username)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError,
                res -> Mono.error(new UserNotFoundException(username)))
            .onStatus(HttpStatusCode::is5xxServerError,
                res -> Mono.error(new GithubApiException("GitHub API unavailable")))
            .bodyToMono(GithubUser.class)
            .timeout(Duration.ofSeconds(5))
            .retry(2);
    }
}
```

### POST with Request Body

```java
public Mono<OrderResponse> createOrder(CreateOrderRequest req) {
    return webClient.post()
        .uri("/orders")
        .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(req)
        .retrieve()
        .bodyToMono(OrderResponse.class);
}
```

### Error Handling Strategies

```java
// Strategy 1: onStatus — customize per status code
.retrieve()
.onStatus(HttpStatusCode::is4xxClientError, res ->
    res.bodyToMono(String.class)
       .flatMap(body -> Mono.error(new ApiException("Client error: " + body))))

// Strategy 2: onErrorResume — catch all errors and provide fallback
.retrieve()
.bodyToMono(Product.class)
.onErrorResume(WebClientResponseException.class, ex -> {
    log.warn("Product service failed, returning cache: {}", ex.getMessage());
    return getFallbackProduct(productId);
})

// Strategy 3: doOnError — log but propagate
.retrieve()
.bodyToMono(Product.class)
.doOnError(ex -> log.error("Failed to fetch product: {}", ex.getMessage()))
```

### Timeouts

```java
// Connection timeout (how long to wait for TCP handshake)
@Bean
public WebClient webClient(WebClient.Builder builder) {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
        .responseTimeout(Duration.ofSeconds(10));

    return builder
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}

// Per-request timeout (override global)
webClient.get()
    .uri("/slow-endpoint")
    .httpRequest(req -> req.getNativeRequest().responseTimeout(Duration.ofSeconds(30)))
    .retrieve()
    .bodyToMono(String.class);
```

---

## RestClient (Spring Boot 3.2+)

Synchronous HTTP client with a fluent API. Replacement for `RestTemplate` when you don't need reactive features.

### Configuration

```java
@Configuration
public class RestClientConfig {

    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.USER_AGENT, "MyApp/1.0")
            .requestInterceptor((req, body, execution) -> {
                log.debug("Request: {} {}", req.getMethod(), req.getURI());
                return execution.execute(req, body);
            })
            .build();
    }
}
```

### Basic Usage

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final RestClient restClient;

    public ProductResponse getProduct(Long id) {
        return restClient.get()
            .uri("/products/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
                throw new ProductNotFoundException("Product not found: " + id);
            })
            .body(ProductResponse.class);
    }

    public ProductResponse createProduct(CreateProductRequest req) {
        return restClient.post()
            .uri("/products")
            .contentType(MediaType.APPLICATION_JSON)
            .body(req)
            .retrieve()
            .body(ProductResponse.class);
    }
}
```

### With Resilience4j

```java
@CircuitBreaker(name = "productApi", fallbackMethod = "getProductFallback")
@Retry(name = "productApi")
public ProductResponse getProduct(Long id) {
    return restClient.get()
        .uri("/products/{id}", id)
        .retrieve()
        .body(ProductResponse.class);
}

private ProductResponse getProductFallback(Long id, Exception ex) {
    log.warn("Product API unavailable, returning cached data: {}", ex.getMessage());
    return cache.get(id).orElse(ProductResponse.unavailable(id));
}
```

---

## Spring Kafka

Production-grade Kafka integration with automatic serialization, consumer groups, and error handling.

### Dependencies

```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
```

### Configuration

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: my-app-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: com.example.myapp.event
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all # wait for all replicas to ack
```

### Producer

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;

    public void publishOrderPlaced(Order order) {
        OrderPlacedEvent event = new OrderPlacedEvent(
            order.getId(), order.getUserId(), order.getTotal()
        );

        kafkaTemplate.send("order-events", order.getId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish order event: {}", ex.getMessage());
                } else {
                    log.info("Published order event: partition={}, offset={}",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

### Consumer

```java
@Component
@Slf4j
public class OrderEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void handleOrderPlaced(OrderPlacedEvent event) {
        log.info("Received order event: {}", event);
        // Process event: send email, update analytics, etc.
    }

    // Manual acknowledgment for exactly-once processing
    @KafkaListener(topics = "order-events", groupId = "warehouse-service",
                   containerFactory = "manualAckKafkaListenerContainerFactory")
    public void processOrder(OrderPlacedEvent event, Acknowledgment ack) {
        try {
            warehouseService.reserveStock(event.orderId());
            ack.acknowledge();  // commit offset only after successful processing
        } catch (Exception ex) {
            log.error("Failed to process order, will retry: {}", ex.getMessage());
            // Do NOT ack — message will be redelivered
        }
    }
}
```

### Error Handling

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderPlacedEvent>
            kafkaListenerContainerFactory(
                ConsumerFactory<String, OrderPlacedEvent> consumerFactory) {

        var factory = new ConcurrentKafkaListenerContainerFactory<String, OrderPlacedEvent>();
        factory.setConsumerFactory(consumerFactory);
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new FixedBackOff(1000L, 3L)  // retry 3 times with 1s between attempts
        ));
        return factory;
    }

    // Dead Letter Topic (DLT) pattern
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderPlacedEvent>
            kafkaListenerContainerFactoryWithDlt(
                ConsumerFactory<String, OrderPlacedEvent> consumerFactory,
                KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate) {

        var factory = new ConcurrentKafkaListenerContainerFactory<String, OrderPlacedEvent>();
        factory.setConsumerFactory(consumerFactory);

        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate),
            new FixedBackOff(1000L, 3L)
        );
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }
}
```

---

## API Integration Best Practices

### 1. Always Set Timeouts

Without timeouts, a slow upstream service can exhaust your thread pool.

```java
// WebClient
.httpRequest(req -> req.getNativeRequest().responseTimeout(Duration.ofSeconds(10)))

// RestClient (via RequestFactory)
@Bean
public RestClient restClient(RestClient.Builder builder) {
    HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(5000);
    factory.setConnectionRequestTimeout(5000);
    return builder.requestFactory(factory).build();
}
```

### 2. Use Circuit Breakers for External Services

Prevent cascading failures when dependencies are down.

```java
@CircuitBreaker(name = "externalApi", fallbackMethod = "fallback")
@Retry(name = "externalApi")
public ApiResponse callExternalApi(String query) {
    return restClient.get().uri("/search?q={query}", query).retrieve().body(ApiResponse.class);
}
```

### 3. Retry Idempotent Operations Only

Only retry GET requests and operations designed to be idempotent. Never auto-retry POST/PUT without idempotency keys.

```yaml
resilience4j:
  retry:
    instances:
      readOnlyApi:
        retryExceptions:
          - java.net.SocketTimeoutException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException # 4xx = client error, don't retry
```

### 4. Structured Logging for Integration Points

```java
@Slf4j
public class ExternalApiClient {
    public ApiResponse call(String query) {
        long start = System.currentTimeMillis();
        try {
            ApiResponse response = restClient.get().uri("/search?q={query}", query)
                .retrieve().body(ApiResponse.class);
            log.info("API call succeeded: query={}, duration={}ms", query, System.currentTimeMillis() - start);
            return response;
        } catch (Exception ex) {
            log.error("API call failed: query={}, duration={}ms, error={}",
                query, System.currentTimeMillis() - start, ex.getMessage());
            throw ex;
        }
    }
}
```

### 5. Health Checks for Dependencies

Expose circuit breaker state via Actuator:

```yaml
management:
  health:
    circuitbreakers:
      enabled: true
```

```json
// GET /actuator/health
{
  "status": "UP",
  "components": {
    "circuitBreakers": {
      "status": "UP",
      "details": {
        "paymentService": {
          "status": "UP",
          "details": {
            "state": "CLOSED",
            "failureRate": "12.5%",
            "slowCallRate": "0.0%"
          }
        }
      }
    }
  }
}
```

### 6. Correlation IDs

Pass correlation IDs through the entire request chain for distributed tracing.

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String correlationId = req.getHeader("X-Correlation-ID");
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }
        MDC.put("correlationId", correlationId);
        res.setHeader("X-Correlation-ID", correlationId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear();
        }
    }
}

// Propagate to downstream services
webClient.get()
    .uri("/downstream")
    .header("X-Correlation-ID", MDC.get("correlationId"))
    .retrieve()
    .bodyToMono(String.class);
```
