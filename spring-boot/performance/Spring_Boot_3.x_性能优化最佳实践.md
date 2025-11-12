# Spring Boot 3.x 性能优化最佳实践

> 基于 Spring Boot 3.3.x + Java 21 的企业级性能与可观测性指南
> 
> 更新时间：2025-11-12

## 目录

- [1. 技术栈选型](#1-技术栈选型)
- [2. 数据库连接池优化](#2-数据库连接池优化)
- [3. 缓存策略](#3-缓存策略)
- [4. 并发编程：Virtual Threads](#4-并发编程virtual-threads)
- [5. HTTP 服务器调优](#5-http-服务器调优)
- [6. JVM 与 GC 优化](#6-jvm-与-gc-优化)
- [7. Native Image 优化](#7-native-image-优化)
- [8. 可观测性](#8-可观测性)
- [9. 限流与熔断](#9-限流与熔断)
- [10. 性能分析工具](#10-性能分析工具)
- [11. 云原生与自动扩缩容](#11-云原生与自动扩缩容)

---

## 1. 技术栈选型

### 核心依赖

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.5</version>
</parent>

<properties>
    <java.version>21</java.version>
</properties>

<dependencies>
    <!-- Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- 可观测性 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
    <!-- Micrometer + OpenTelemetry -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>
    
    <!-- 限流熔断 -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
        <version>2.2.0</version>
    </dependency>
</dependencies>
```

### 为什么选择 Java 21？

- **Virtual Threads**：简化高并发编程，无需响应式编程
- **性能提升**：相比 Java 17 提升 10-15%
- **ZGC 成熟**：亚毫秒级 GC 暂停
- **模式匹配**：代码更简洁

---

## 2. 数据库连接池优化

### HikariCP 生产配置

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/app_db
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      # 连接池大小：云数据库推荐公式 = (CPU核心数 * 2) + 1
      # 更重要的是不超过数据库的 max_connections 限制
      maximum-pool-size: 20
      minimum-idle: 10
      # 连接生命周期应小于数据库超时时间（如 MySQL wait_timeout）
      max-lifetime: 1800000  # 30分钟
      connection-timeout: 20000  # 20秒
      idle-timeout: 600000  # 10分钟
      # 连接泄漏检测（开发环境启用）
      leak-detection-threshold: 60000  # 60秒
      
  jpa:
    properties:
      hibernate:
        # 慢查询日志（生产环境建议 100-200ms）
        session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100
        # 批处理优化
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
        # 二级缓存
        cache.use_second_level_cache: true
        cache.region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
```

### 关键优化点

1. **连接池大小**：不是越大越好
   - 过大：增加数据库负载和内存消耗
   - 过小：请求排队等待
   - 推荐公式：`(CPU核心数 * 2) + 1`，通常 10-20 个连接
   - **关键**：不要超过数据库的 `max_connections` 限制

2. **批处理**：减少网络往返
   ```java
   @Transactional
   public void batchInsert(List<Entity> entities) {
       int batchSize = 50;
       for (int i = 0; i < entities.size(); i++) {
           entityManager.persist(entities.get(i));
           if (i % batchSize == 0 && i > 0) {
               entityManager.flush();
               entityManager.clear();
           }
       }
       // 处理最后一批
       entityManager.flush();
       entityManager.clear();
   }
   ```

3. **索引优化**：使用 `EXPLAIN ANALYZE` 分析查询计划

4. **连接池监控**：暴露 HikariCP 指标
   ```java
   @Configuration
   public class DataSourceConfig {
       @Bean
       public MeterBinder hikariMetrics(DataSource dataSource) {
           if (dataSource instanceof HikariDataSource hikari) {
               return (registry) -> hikari.getHikariPoolMXBean()
                   .ifPresent(pool -> new HikariCPMetrics(pool).bindTo(registry));
           }
           return registry -> {};
       }
   }
   ```

---

## 3. 缓存策略

### 多级缓存架构

```
Client → Spring Boot → L1 Cache (Caffeine) → L2 Cache (Redis) → Database
```

### Caffeine 本地缓存

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats());  // 启用统计
        return cacheManager;
    }
}
```

### Redis 分布式缓存

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 2
      timeout: 2000ms
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10分钟
      # 缓存空值防止缓存穿透（配合短 TTL）
      cache-null-values: true
      # 使用前缀避免 key 冲突
      use-key-prefix: true
      key-prefix: "app:"
```

### 缓存使用示例

```java
@Service
public class ProductService {
    
    // 本地缓存：高频读取
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("Product not found"));
    }
    
    // 缓存更新
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
    
    // 缓存失效
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
}
```

### 防止缓存问题

1. **缓存穿透**：查询不存在的数据
   - 解决方案 1：缓存空值（`cache-null-values: true` + 短 TTL）
   - 解决方案 2：布隆过滤器
   ```java
   @Configuration
   public class BloomFilterConfig {
       @Bean
       public BloomFilter<String> productBloomFilter() {
           BloomFilter<String> filter = BloomFilter.create(
               Funnels.stringFunnel(Charset.defaultCharset()),
               100000,  // 预期元素数量
               0.01     // 误判率
           );
           // 初始化时加载所有 ID
           productRepository.findAllIds().forEach(filter::put);
           return filter;
       }
   }
   ```

2. **缓存击穿**：热点数据过期，大量请求同时打到数据库
   - 解决：`@Cacheable(sync = true)` 或分布式锁
   ```java
   @Cacheable(value = "hotProducts", key = "#id", sync = true)
   public Product getHotProduct(Long id) {
       return productRepository.findById(id).orElseThrow();
   }
   ```

3. **缓存雪崩**：大量缓存同时过期
   - 解决：随机过期时间 + 限流 + 多级缓存
   ```java
   @Bean
   public RedisCacheConfiguration cacheConfiguration() {
       return RedisCacheConfiguration.defaultCacheConfig()
           .entryTtl(Duration.ofMinutes(10)
               .plusSeconds(ThreadLocalRandom.current().nextInt(60)))  // 随机 0-60 秒
           .disableCachingNullValues();
   }
   ```

---

## 4. 并发编程：Virtual Threads

### 启用 Virtual Threads

**方式 1：Tomcat 配置（推荐）**
```yaml
server:
  tomcat:
    threads:
      max: 200
    # 使用 Virtual Threads 处理请求
    use-virtual-threads: true  # Spring Boot 3.2+
```

**方式 2：@Async 配置**
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
    public AsyncTaskExecutor asyncTaskExecutor() {
        TaskExecutorBuilder builder = new TaskExecutorBuilder();
        return builder
            .taskDecorator(new ContextPropagatingTaskDecorator())
            .build()
            .virtualThreadTaskExecutor();  // 使用 Virtual Threads
    }
}
```

**方式 3：手动创建 Virtual Thread Executor**
```java
@Bean
public Executor virtualThreadExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();
}
```

### 传统线程池 vs Virtual Threads

**传统方式（不推荐）：**
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}
```

**Virtual Threads 方式（推荐）：**
```java
@Service
public class ReportService {
    
    // 自动使用 Virtual Threads
    @Async
    public CompletableFuture<Report> generateReport(Long id) {
        // 阻塞式代码也能高并发
        String data = restTemplate.getForObject(url, String.class);
        return CompletableFuture.completedFuture(processData(data));
    }
}
```

### Virtual Threads 优势

- **轻量级**：百万级线程不是问题（每个 Virtual Thread 仅占用几 KB）
- **简单**：无需学习响应式编程，保持传统阻塞式代码风格
- **兼容**：现有阻塞代码无需改动
- **性能**：I/O 密集型应用吞吐量提升 2-5 倍

### 何时不用 Virtual Threads

- **CPU 密集型任务**：使用 ForkJoinPool 或平台线程
- **频繁使用 ThreadLocal**：会增加内存开销（Virtual Threads 完全支持 ThreadLocal）
- **使用 synchronized 的热点代码**：会固定到平台线程（改用 ReentrantLock）
- **需要线程优先级**：Virtual Threads 不支持优先级

### Virtual Threads 最佳实践

```java
@RestController
public class OrderController {
    
    @Autowired
    private RestTemplate restTemplate;
    
    // 自动使用 Virtual Threads（如果启用了 use-virtual-threads）
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        // 多个阻塞调用可以并发执行
        CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(
            () -> orderService.getOrder(id)
        );
        CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(
            () -> userService.getUser(order.getUserId())
        );
        
        return orderFuture.thenCombine(userFuture, (order, user) -> {
            order.setUser(user);
            return order;
        }).join();
    }
}

---

## 5. HTTP 服务器调优

### Tomcat 配置（默认）

```yaml
server:
  port: 8080
  # 优雅关闭
  shutdown: graceful
  tomcat:
    threads:
      max: 200  # Virtual Threads 下可以更大
      min-spare: 10
    connection-timeout: 20s
    max-connections: 8192
    accept-count: 100
    max-http-form-post-size: 2MB
    # 启用 Virtual Threads
    use-virtual-threads: true

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

### 切换到 Undertow（可选）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

```yaml
server:
  shutdown: graceful
  undertow:
    threads:
      io: 4  # CPU 核心数
      worker: 200
    # 16KB buffer 性能更好
    buffer-size: 16384
    direct-buffers: true
```

### HTTP/2 支持

```yaml
server:
  http2:
    enabled: true
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_PASSWORD}
    key-store-type: PKCS12
```

---

## 6. JVM 与 GC 优化

### Java 21 推荐配置

```bash
# 生产环境 JVM 参数
JAVA_OPTS="
  # 堆内存（容器环境建议 Xms = Xmx 避免动态扩容）
  -Xms2g -Xmx2g
  
  # ZGC（推荐用于低延迟应用，Java 21 默认 Generational ZGC）
  -XX:+UseZGC
  
  # 或使用 G1GC（通用场景）
  # -XX:+UseG1GC
  # -XX:MaxGCPauseMillis=50
  # -XX:G1ReservePercent=10
  
  # 容器支持（自动识别 cgroup 限制）
  -XX:+UseContainerSupport
  -XX:MaxRAMPercentage=75.0
  
  # 诊断
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/heapdump.hprof
  -XX:+ExitOnOutOfMemoryError
  
  # GC 日志（带轮转）
  -Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags:filecount=5,filesize=100M
  
  # 性能优化
  -XX:+UseStringDeduplication
  -XX:+OptimizeStringConcat
  
  # 启用 JFR（生产环境性能分析）
  -XX:StartFlightRecording=dumponexit=true,filename=/var/log/flight.jfr
"
```

### GC 选择指南

| GC 类型 | 适用场景 | 暂停时间 | 吞吐量 |
|---------|---------|---------|--------|
| **ZGC** | 低延迟应用（<10ms） | <1ms | 中等 |
| **G1GC** | 通用场景 | 50-200ms | 高 |
| **Parallel GC** | 批处理任务 | 较长 | 最高 |

### 监控 GC 性能

```java
@Component
public class GCMetrics {
    
    @Autowired
    private MeterRegistry registry;
    
    @PostConstruct
    public void registerGCMetrics() {
        new JvmGcMetrics().bindTo(registry);
        new JvmMemoryMetrics().bindTo(registry);
    }
}
```

---

## 7. Native Image 优化

### 构建 Native Image

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

```bash
# 构建 Native Image
mvn -Pnative native:compile

# 运行
./target/myapp
```

### Native Image 优势

- **启动速度**：毫秒级（vs 秒级）
- **内存占用**：减少 50-70%
- **适用场景**：Serverless、Kubernetes、边缘计算

### 限制与注意事项

1. **反射**：需要配置 `reflect-config.json`
2. **动态代理**：需要提前声明
3. **类加载**：不支持运行时类加载
4. **构建时间**：较长（5-10 分钟）

### 配置示例

```yaml
# application.properties
spring.aot.enabled=true
```

---

## 8. 可观测性

### OpenTelemetry 集成

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
  tracing:
    sampling:
      probability: 1.0  # 生产环境建议 0.1
      
otel:
  service:
    name: my-spring-boot-app
  exporter:
    otlp:
      endpoint: http://localhost:4317
```

### 自定义指标

```java
@RestController
public class OrderController {
    
    private final Counter orderCounter;
    private final Timer orderTimer;
    
    public OrderController(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created")
            .description("Total orders created")
            .tag("type", "online")
            .register(registry);
            
        this.orderTimer = Timer.builder("orders.processing.time")
            .description("Order processing time")
            .register(registry);
    }
    
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest request) {
        return orderTimer.record(() -> {
            Order order = orderService.create(request);
            orderCounter.increment();
            return order;
        });
    }
}
```

### 分布式追踪

```java
@Service
public class PaymentService {
    
    @Autowired
    private Tracer tracer;
    
    public Payment processPayment(Order order) {
        Span span = tracer.spanBuilder("payment.process")
            .setAttribute("order.id", order.getId())
            .setAttribute("amount", order.getAmount())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 业务逻辑
            return paymentGateway.charge(order);
        } catch (Exception e) {
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### 健康检查

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            return Health.up()
                .withDetail("database", "PostgreSQL")
                .withDetail("validationQuery", "SELECT 1")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

---

## 9. 限流与熔断

### Resilience4j 集成

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        slowCallRateThreshold: 100
        slowCallDurationThreshold: 2s
        
  ratelimiter:
    instances:
      apiLimiter:
        limitForPeriod: 100
        limitRefreshPeriod: 1s
        timeoutDuration: 0s
        
  retry:
    instances:
      externalApi:
        maxAttempts: 3
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
```

### 使用示例

```java
@Service
public class PaymentService {
    
    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
    @RateLimiter(name = "apiLimiter")
    @Retry(name = "externalApi")
    public Payment processPayment(Order order) {
        return paymentGateway.charge(order);
    }
    
    // 熔断降级方法
    private Payment fallbackPayment(Order order, Exception e) {
        log.error("Payment service unavailable, using fallback", e);
        return Payment.pending(order.getId());
    }
}
```

### 监控熔断器状态

```java
@RestController
public class CircuitBreakerController {
    
    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;
    
    @GetMapping("/circuit-breakers")
    public Map<String, String> getCircuitBreakers() {
        return circuitBreakerRegistry.getAllCircuitBreakers()
            .stream()
            .collect(Collectors.toMap(
                CircuitBreaker::getName,
                cb -> cb.getState().name()
            ));
    }
}
```

---

## 10. 性能分析工具

### Async Profiler

```bash
# 下载
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz

# CPU 分析（60秒）
./profiler.sh -d 60 -f flamegraph.html <pid>

# 内存分配分析
./profiler.sh -d 60 -e alloc -f alloc.html <pid>
```

### JFR (Java Flight Recorder)

```bash
# 启动时开启
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar

# 运行时开启
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# 分析（使用 JDK Mission Control）
jmc recording.jfr
```

### Arthas

```bash
# 启动
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar

# 常用命令
dashboard          # 实时仪表板
thread -n 3        # 查看最忙的3个线程
trace com.example.Service method  # 方法调用追踪
monitor -c 5 com.example.Service method  # 方法调用监控
```

### 性能测试

```xml
<dependency>
    <groupId>io.gatling.highcharts</groupId>
    <artifactId>gatling-charts-highcharts</artifactId>
    <version>3.10.0</version>
    <scope>test</scope>
</dependency>
```

```scala
class LoadTest extends Simulation {
  val httpProtocol = http.baseUrl("http://localhost:8080")
  
  val scn = scenario("Load Test")
    .exec(http("Get Products")
      .get("/api/products")
      .check(status.is(200)))
  
  setUp(
    scn.inject(
      rampUsersPerSec(10) to 100 during (60 seconds),
      constantUsersPerSec(100) during (300 seconds)
    )
  ).protocols(httpProtocol)
}
```

---

## 11. 云原生与自动扩缩容

### Kubernetes 部署

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        env:
        - name: JAVA_OPTS
          value: "-Xms1g -Xmx1g -XX:+UseZGC"
```

### HPA（水平自动扩缩容）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: spring-boot-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: spring-boot-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
```

### KEDA（基于事件驱动的自动扩缩容）

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: spring-boot-app-scaler
spec:
  scaleTargetRef:
    name: spring-boot-app
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: http_requests_per_second
      threshold: '1000'
      query: sum(rate(http_server_requests_seconds_count[1m]))
```

### 成本优化

1. **Spot 实例**：节省 50-70% 成本
   ```yaml
   nodeSelector:
     karpenter.sh/capacity-type: spot
   ```

2. **ARM 架构**：Graviton 实例性价比更高
   ```yaml
   nodeSelector:
     kubernetes.io/arch: arm64
   ```

3. **资源配额**：避免过度分配
   ```yaml
   resources:
     requests:
       memory: "512Mi"  # 实际需要
       cpu: "500m"
     limits:
       memory: "1Gi"    # 防止 OOM
       cpu: "1000m"     # 防止 CPU 饥饿
   ```

---

## 性能优化检查清单

### 应用层
- [ ] 启用 Virtual Threads
- [ ] 配置多级缓存（Caffeine + Redis）
- [ ] 优化数据库连接池（10-20 连接）
- [ ] 启用批处理（batch_size=50）
- [ ] 添加数据库索引
- [ ] 实现慢查询日志
- [ ] 配置限流（Resilience4j RateLimiter）
- [ ] 配置熔断（CircuitBreaker）
- [ ] 实现优雅降级（Fallback）

### JVM 层
- [ ] 使用 Java 21
- [ ] 配置 ZGC 或 G1GC
- [ ] 设置 Xms = Xmx
- [ ] 启用容器支持
- [ ] 配置 GC 日志（带轮转）
- [ ] 启用 HeapDump

### 可观测性
- [ ] 集成 Micrometer + OpenTelemetry
- [ ] 配置 Prometheus 指标
- [ ] 实现分布式追踪
- [ ] 添加自定义业务指标
- [ ] 配置健康检查（Liveness/Readiness）
- [ ] 设置告警规则
- [ ] 监控连接池使用率

### 云原生
- [ ] 配置资源限制（requests/limits）
- [ ] 实现 Liveness/Readiness 探针
- [ ] 配置 HPA
- [ ] 使用 Spot 实例
- [ ] 考虑 ARM 架构
- [ ] 实现优雅关闭（shutdown: graceful）

### 性能测试
- [ ] 建立性能基线
- [ ] 集成负载测试（Gatling）
- [ ] 使用 Async Profiler 分析
- [ ] 监控 GC 暂停时间
- [ ] 分析火焰图
- [ ] 压测验证扩缩容

---

## 参考资源

- [Spring Boot 3.x 官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Java 21 新特性](https://openjdk.org/projects/jdk/21/)
- [Virtual Threads 指南](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- [OpenTelemetry Java](https://opentelemetry.io/docs/languages/java/)
- [Async Profiler](https://github.com/async-profiler/async-profiler)
- [Kubernetes 最佳实践](https://kubernetes.io/docs/concepts/configuration/overview/)

---

## 相关文章

- [Spring Boot 进阶：企业级性能与可观测性指南.pdf](./Spring%20Boot%20进阶：企业级性能与可观测性指南.pdf) - 原始参考文章

---

**标签**：`Spring Boot` `Java 21` `Virtual Threads` `性能优化` `可观测性` `云原生` `ZGC`
