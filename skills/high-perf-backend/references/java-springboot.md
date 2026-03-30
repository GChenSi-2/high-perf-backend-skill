# Java: Spring Boot 3 + Microservices Reference

## Table of Contents
1. [Project Setup & Structure](#project-setup)
2. [Spring Boot 3 Performance Configuration](#performance-config)
3. [Virtual Threads (Project Loom)](#virtual-threads)
4. [Reactive vs Imperative](#reactive-vs-imperative)
5. [Microservice Patterns with Spring Cloud](#microservice-patterns)
6. [Data Access & JPA Optimization](#data-access)
7. [Caching](#caching)
8. [Security](#security)
9. [Observability](#observability)
10. [Testing Strategy](#testing)
11. [Containerization & Deployment](#deployment)
12. [GraalVM Native Image](#native-image)

---

## Project Setup & Structure {#project-setup}

### Recommended Project Layout (Multi-Module Maven/Gradle)

```
my-platform/
├── platform-bom/              # Bill of Materials - version management
│   └── pom.xml
├── platform-common/           # Shared DTOs, exceptions, utils
│   └── src/main/java/
├── service-api-gateway/       # Spring Cloud Gateway
├── service-user/              # User domain microservice
│   ├── src/main/java/com/example/user/
│   │   ├── UserApplication.java
│   │   ├── adapter/           # Inbound/outbound adapters (controllers, clients)
│   │   │   ├── in/web/        # REST controllers
│   │   │   └── out/persistence/ # Repository implementations
│   │   ├── application/       # Use cases / application services
│   │   │   ├── port/in/       # Inbound ports (interfaces)
│   │   │   └── port/out/      # Outbound ports (interfaces)
│   │   ├── domain/            # Domain model (entities, value objects)
│   │   └── config/            # Spring configuration classes
│   ├── src/main/resources/
│   │   ├── application.yml
│   │   └── db/migration/      # Flyway migrations
│   └── src/test/java/
├── service-order/             # Order domain microservice
├── infra/                     # Docker Compose, Kubernetes manifests
│   ├── docker-compose.yml
│   ├── k8s/
│   └── monitoring/
└── pom.xml                    # Parent POM
```

Use Hexagonal Architecture (Ports & Adapters) within each service to keep business logic
independent of frameworks and infrastructure.

### Parent POM Essentials (Spring Boot 3.2+)

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.x</version>
</parent>

<properties>
    <java.version>21</java.version>
    <spring-cloud.version>2023.0.x</spring-cloud.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Always pin Spring Cloud version compatible with your Spring Boot version. Check the
[compatibility matrix](https://spring.io/projects/spring-cloud).

---

## Spring Boot 3 Performance Configuration {#performance-config}

### application.yml Production Template

```yaml
server:
  port: 8080
  shutdown: graceful                     # Graceful shutdown
  tomcat:
    threads:
      max: 200                           # Tune based on load testing
      min-spare: 20
    max-connections: 8192
    accept-count: 100
    connection-timeout: 5s
    keep-alive-timeout: 60s

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s      # Max wait for graceful shutdown
  jackson:
    serialization:
      write-dates-as-timestamps: false
    default-property-inclusion: non_null  # Smaller JSON payloads
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/${DB_NAME}
    hikari:
      maximum-pool-size: 20             # Start conservative, tune with monitoring
      minimum-idle: 5
      idle-timeout: 300000              # 5 minutes
      max-lifetime: 1800000             # 30 minutes
      connection-timeout: 5000          # 5 seconds — fail fast
      leak-detection-threshold: 30000   # Detect leaked connections
  jpa:
    open-in-view: false                 # CRITICAL: disable OSIV for performance
    hibernate:
      ddl-auto: validate                # Never auto-create in production
    properties:
      hibernate:
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
        generate_statistics: false       # Enable temporarily for debugging

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true                    # Kubernetes probes
  metrics:
    tags:
      application: ${spring.application.name}
```

Key performance flags to always set:
- `spring.jpa.open-in-view: false` — prevents lazy loading in the controller layer, which causes N+1 queries
- `server.shutdown: graceful` — drains requests before stopping
- HikariCP `leak-detection-threshold` — catches connection leaks early

---

## Virtual Threads (Project Loom) {#virtual-threads}

Spring Boot 3.2+ has first-class virtual thread support. Use them for I/O-bound workloads.

### Enabling Virtual Threads

```yaml
spring:
  threads:
    virtual:
      enabled: true   # Switches Tomcat to virtual-thread-per-request model
```

This single flag makes Tomcat handle each request on a virtual thread, dramatically improving
throughput for I/O-heavy services without reactive programming complexity.

### When to Use Virtual Threads vs Reactive (WebFlux)

| Aspect | Virtual Threads | WebFlux (Reactive) |
|---|---|---|
| Code style | Imperative (blocking-friendly) | Reactive (Mono/Flux, non-blocking) |
| Learning curve | Low | High |
| Debugging | Easy (normal stack traces) | Hard (reactive chain debugging) |
| CPU-bound work | Better (uses OS threads for compute) | No advantage |
| Streaming | Limited | Excellent (backpressure built-in) |
| Ecosystem compatibility | Works with all blocking libraries | Needs reactive-compatible drivers |

**Recommendation**: Default to Virtual Threads for most services. Use WebFlux only for
streaming-heavy or SSE/WebSocket services, or when you need fine-grained backpressure control.

### Pitfalls with Virtual Threads
- `synchronized` blocks pin virtual threads to carrier threads — prefer `ReentrantLock`
- Avoid `ThreadLocal` for large objects — virtual threads create millions of them
- Connection pools still limit concurrency — size them to match your actual I/O capacity

---

## Microservice Patterns with Spring Cloud {#microservice-patterns}

### Service Discovery (Spring Cloud + Kubernetes)

For Kubernetes deployments, prefer Kubernetes-native service discovery over Eureka:

```yaml
spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
        all-namespaces: false
      loadbalancer:
        mode: SERVICE  # Use Kubernetes Service for load balancing
```

For non-K8s environments, use Spring Cloud Netflix Eureka or Consul.

### API Gateway (Spring Cloud Gateway)

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/v1/users/**")
                .filters(f -> f
                    .circuitBreaker(c -> c
                        .setName("userService")
                        .setFallbackUri("forward:/fallback/users"))
                    .retry(retryConfig -> retryConfig
                        .setRetries(3)
                        .setStatuses(HttpStatus.SERVICE_UNAVAILABLE))
                    .requestRateLimiter(rl -> rl
                        .setRateLimiter(redisRateLimiter())))
                .uri("lb://user-service"))
            .build();
    }

    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20); // 10 requests/sec, burst 20
    }
}
```

### Circuit Breaker (Resilience4j)

```java
@Service
public class OrderService {

    private final UserClient userClient;

    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    @Retry(name = "userService")
    @TimeLimiter(name = "userService")
    public CompletableFuture<UserDto> getUser(Long userId) {
        return CompletableFuture.supplyAsync(() -> userClient.getUser(userId));
    }

    private CompletableFuture<UserDto> getUserFallback(Long userId, Throwable t) {
        log.warn("Fallback for user {} due to: {}", userId, t.getMessage());
        return CompletableFuture.completedFuture(UserDto.unknown(userId));
    }
}
```

Resilience4j configuration:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        failure-rate-threshold: 50        # Open at 50% failure
        wait-duration-in-open-state: 10s  # Wait before half-open
        permitted-number-of-calls-in-half-open-state: 3
  retry:
    instances:
      userService:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
  timelimiter:
    instances:
      userService:
        timeout-duration: 3s
```

### Inter-Service Communication

**Declarative REST Client (Spring 6 HTTP Interface)**

```java
@HttpExchange("/api/v1/users")
public interface UserClient {

    @GetExchange("/{id}")
    UserDto getUser(@PathVariable Long id);

    @PostExchange
    UserDto createUser(@RequestBody CreateUserRequest request);
}

@Configuration
public class ClientConfig {

    @Bean
    public UserClient userClient(RestClient.Builder builder) {
        RestClient restClient = builder
            .baseUrl("http://user-service:8080")
            .requestInterceptor(new TracingClientHttpRequestInterceptor()) // propagate trace context
            .build();
        return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(restClient))
            .build()
            .createClient(UserClient.class);
    }
}
```

**Async Messaging with Spring Cloud Stream + Kafka**

```java
@Configuration
public class MessagingConfig {

    @Bean
    public Function<OrderCreatedEvent, PaymentRequest> processOrder() {
        return event -> {
            // Transform and route
            return new PaymentRequest(event.orderId(), event.amount());
        };
    }
}
```

```yaml
spring:
  cloud:
    stream:
      bindings:
        processOrder-in-0:
          destination: order-created
          group: payment-service
        processOrder-out-0:
          destination: payment-requested
      kafka:
        binder:
          brokers: ${KAFKA_BROKERS:localhost:9092}
```

### Saga Pattern (Choreography-based)

For distributed transactions across microservices, use the Saga pattern instead of 2PC:

```java
// Each service listens for events and publishes compensating events on failure
@Component
public class PaymentEventHandler {

    @Bean
    public Consumer<PaymentFailedEvent> handlePaymentFailed() {
        return event -> {
            // Compensating action: cancel the order
            orderService.cancelOrder(event.orderId());
            eventPublisher.publish(new OrderCancelledEvent(event.orderId()));
        };
    }
}
```

---

## Data Access & JPA Optimization {#data-access}

### Preventing N+1 Queries

```java
// BAD: N+1 problem
@Entity
public class Order {
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY) // LAZY is correct...
    private List<OrderItem> items; // ...but accessing items in a loop = N+1
}

// GOOD: Use JOIN FETCH or EntityGraph
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithItems(@Param("id") Long id);

    @EntityGraph(attributePaths = {"items", "items.product"})
    List<Order> findByCustomerId(Long customerId);
}
```

### Batch Operations

```java
@Modifying
@Query("UPDATE Order o SET o.status = :status WHERE o.id IN :ids")
int bulkUpdateStatus(@Param("ids") List<Long> ids, @Param("status") OrderStatus status);
```

For bulk inserts, use `JdbcTemplate` or `spring-data-jdbc` to bypass JPA overhead:

```java
@Repository
public class OrderBulkRepository {

    private final JdbcTemplate jdbc;

    public void bulkInsert(List<Order> orders) {
        jdbc.batchUpdate(
            "INSERT INTO orders (id, customer_id, amount, status) VALUES (?, ?, ?, ?)",
            orders, 1000, // batch size
            (ps, order) -> {
                ps.setLong(1, order.getId());
                ps.setLong(2, order.getCustomerId());
                ps.setBigDecimal(3, order.getAmount());
                ps.setString(4, order.getStatus().name());
            });
    }
}
```

### Read Replicas

```yaml
# application.yml - routing datasource
spring:
  datasource:
    primary:
      url: jdbc:postgresql://primary:5432/mydb
      hikari:
        maximum-pool-size: 20
    replica:
      url: jdbc:postgresql://replica:5432/mydb
      hikari:
        maximum-pool-size: 30
```

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {
        var routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                    ? "replica" : "primary";
            }
        };
        routing.setTargetDataSources(Map.of("primary", primary, "replica", replica));
        routing.setDefaultTargetDataSource(primary);
        return routing;
    }
}
```

Mark read-only operations with `@Transactional(readOnly = true)` to auto-route to replicas.

---

## Caching {#caching}

### Multi-Level Cache with Caffeine + Redis

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        // L1: Caffeine (in-process, fast, small)
        CaffeineCacheManager caffeineManager = new CaffeineCacheManager();
        caffeineManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats());

        // L2: Redis (distributed, larger)
        RedisCacheManager redisManager = RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30))
                .serializeValuesWith(
                    RedisSerializationContext.SerializationPair.fromSerializer(
                        new GenericJackson2JsonRedisSerializer())))
            .build();

        return new CompositeCacheManager(caffeineManager, redisManager);
    }
}

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public ProductDto getProduct(Long id) {
        return productRepository.findById(id)
            .map(ProductDto::from)
            .orElseThrow(() -> new NotFoundException("Product", id));
    }

    @CacheEvict(value = "products", key = "#id")
    public void updateProduct(Long id, UpdateProductRequest request) { ... }
}
```

---

## Observability {#observability}

### OpenTelemetry + Micrometer (Spring Boot 3)

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.1          # 10% sampling in production
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

### Custom Business Metrics

```java
@Component
public class OrderMetrics {

    private final Counter ordersCreated;
    private final Timer orderProcessingTime;

    public OrderMetrics(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders.created")
            .description("Total orders created")
            .tag("service", "order-service")
            .register(registry);
        this.orderProcessingTime = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }

    public void recordOrderCreated() { ordersCreated.increment(); }

    public <T> T timeOrderProcessing(Supplier<T> operation) {
        return orderProcessingTime.record(operation);
    }
}
```

---

## Testing Strategy {#testing}

### Test Pyramid for Microservices

```java
// Unit Test — fast, no Spring context
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository orderRepo;
    @InjectMocks OrderService orderService;

    @Test
    void shouldCalculateTotalCorrectly() { ... }
}

// Integration Test — with Spring context + Testcontainers
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Test
    void shouldPersistAndRetrieveOrder() { ... }
}

// Contract Test — verify API contracts between services
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
class OrderControllerContractTest {
    @Autowired MockMvc mockMvc;

    @Test
    void createOrder_shouldReturn201() throws Exception {
        mockMvc.perform(post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"customerId": 1, "items": [{"productId": 10, "quantity": 2}]}
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists());
    }
}
```

---

## Containerization & Deployment {#deployment}

### Multi-Stage Dockerfile (JDK 21)

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B
COPY src src
RUN ./mvnw package -DskipTests -B

# Runtime stage
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
USER app
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

ENV JAVA_OPTS="-XX:+UseZGC -XX:MaxRAMPercentage=75.0 -XX:+ExitOnOutOfMemoryError"
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

Key JVM flags for containers:
- `-XX:+UseZGC` — low-latency GC (or `-XX:+UseG1GC` for balanced throughput/latency)
- `-XX:MaxRAMPercentage=75.0` — use 75% of container memory limit for heap
- `-XX:+ExitOnOutOfMemoryError` — let the orchestrator restart on OOM

---

## GraalVM Native Image {#native-image}

For services where startup time and memory footprint matter (serverless, scale-to-zero):

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

```bash
./mvnw -Pnative native:compile
```

Trade-offs:
- Startup: ~50ms vs ~2-5s (JVM)
- Memory: ~50-100MB vs ~200-500MB
- Build time: much longer (5-15 min)
- Runtime peak throughput: slightly lower than warmed-up JVM (no JIT)
- Reflection/dynamic proxies need hints configuration
