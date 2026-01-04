# System Optimization & Efficiency Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2025-12-31 |
| Status | Approved |
| Owner | Platform Engineering Team |
| Review Cycle | Quarterly |

---

## 2. Overview

### 2.1 Purpose
This document specifies **performance optimization strategies, caching policies, resource efficiency guidelines, and scalability patterns** for the N9 platform to ensure optimal user experience under varying load conditions.

### 2.2 Performance Targets

| Metric | Target | Critical Threshold |
|--------|--------|-------------------|
| API Response Time (p50) | <100ms | >200ms |
| API Response Time (p95) | <300ms | >500ms |
| API Response Time (p99) | <1s | >2s |
| Page Load Time | <2s | >4s |
| Time to First Byte | <200ms | >500ms |
| Database Query Time | <50ms | >200ms |
| Cache Hit Ratio | >90% | <80% |
| Error Rate | <0.1% | >1% |

---

## 3. Caching Architecture

### 3.1 Multi-Layer Cache Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                       Client Layer                           │
│  • Browser Cache (static assets)                             │
│  • Service Worker Cache (offline support)                    │
│  • Local Storage (user preferences)                          │
└─────────────────────┬───────────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                        CDN Layer                             │
│  • CloudFront (global edge caching)                          │
│  • Static assets, images, covers                             │
│  • API response caching (selective)                          │
└─────────────────────┬───────────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│  • Redis Cluster (distributed cache)                         │
│  • Local Cache (Caffeine - hot data)                         │
│  • Query Result Cache                                        │
└─────────────────────┬───────────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                      Database Layer                          │
│  • PostgreSQL Query Cache                                    │
│  • Connection Pooling (HikariCP)                             │
│  • Prepared Statement Cache                                  │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Cache Configuration

```yaml
cache:
  redis:
    cluster:
      nodes: 6  # 3 masters, 3 replicas
      max_redirects: 3
    connection:
      pool_size: 50
      min_idle: 10
      timeout: 2000ms
    
  local:
    provider: caffeine
    spec: "maximumSize=10000,expireAfterWrite=5m"
    
  layers:
    user_session:
      store: redis
      ttl: 24h
      prefix: "session:"
      
    user_profile:
      store: redis
      ttl: 1h
      prefix: "user:profile:"
      invalidation: on_update
      
    story_details:
      store: [local, redis]
      local_ttl: 5m
      redis_ttl: 30m
      prefix: "story:"
      
    chapter_content:
      store: redis
      ttl: 1h
      prefix: "chapter:content:"
      compression: lz4
      
    trending_stories:
      store: redis
      ttl: 15m
      prefix: "trending:"
      
    search_results:
      store: redis
      ttl: 10m
      prefix: "search:"
      key_pattern: "{query}:{filters}:{page}"
```

### 3.3 Cache Service Implementation

```java
@Service
public class CacheService {
    
    private final RedisTemplate<String, Object> redis;
    private final Cache<String, Object> localCache;
    
    public <T> T getOrCompute(
        String key,
        Class<T> type,
        Supplier<T> loader,
        CacheConfig config
    ) {
        // Try local cache first
        if (config.isLocalEnabled()) {
            T local = (T) localCache.getIfPresent(key);
            if (local != null) {
                metrics.increment("cache.local.hit");
                return local;
            }
        }
        
        // Try Redis
        T cached = (T) redis.opsForValue().get(key);
        if (cached != null) {
            metrics.increment("cache.redis.hit");
            // Populate local cache
            if (config.isLocalEnabled()) {
                localCache.put(key, cached);
            }
            return cached;
        }
        
        // Load from source
        metrics.increment("cache.miss");
        T value = loader.get();
        
        // Store in caches
        if (value != null) {
            redis.opsForValue().set(key, value, config.getRedisTtl());
            if (config.isLocalEnabled()) {
                localCache.put(key, value);
            }
        }
        
        return value;
    }
    
    public void invalidate(String pattern) {
        // Invalidate local cache
        localCache.invalidateAll();
        
        // Invalidate Redis (pattern-based)
        Set<String> keys = redis.keys(pattern);
        if (!keys.isEmpty()) {
            redis.delete(keys);
        }
        
        // Publish invalidation event for other nodes
        eventPublisher.publish(new CacheInvalidationEvent(pattern));
    }
}
```

### 3.4 Cache Invalidation Strategy

| Resource | Invalidation Trigger | Strategy |
|----------|---------------------|----------|
| User Profile | Profile update | Immediate |
| Story | Story update | Immediate |
| Chapter | Chapter update/publish | Immediate |
| Trending | Time-based | TTL (15m) |
| Search | Time-based | TTL (10m) |
| Recommendations | User interaction | Delayed (5m) |

---

## 4. Database Optimization

### 4.1 Query Optimization Guidelines

```sql
-- ❌ Bad: N+1 query pattern
SELECT * FROM stories WHERE author_id = ?;
-- Then for each story:
SELECT * FROM chapters WHERE story_id = ?;

-- ✅ Good: Single query with JOIN
SELECT s.*, c.* 
FROM stories s
LEFT JOIN chapters c ON s.id = c.story_id
WHERE s.author_id = ?;

-- ❌ Bad: SELECT *
SELECT * FROM stories WHERE status = 'PUBLISHED';

-- ✅ Good: Select only needed columns
SELECT id, title, description, cover_url, view_count
FROM stories 
WHERE status = 'PUBLISHED';

-- ❌ Bad: OFFSET pagination for large datasets
SELECT * FROM stories ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- ✅ Good: Cursor-based pagination
SELECT * FROM stories 
WHERE created_at < ? 
ORDER BY created_at DESC 
LIMIT 20;
```

### 4.2 Index Strategy

```sql
-- Primary access patterns
CREATE INDEX idx_stories_author_status ON stories(author_id, status);
CREATE INDEX idx_stories_status_created ON stories(status, created_at DESC);
CREATE INDEX idx_chapters_story_order ON chapters(story_id, chapter_order);

-- Full-text search (PostgreSQL)
CREATE INDEX idx_stories_search ON stories 
USING GIN(to_tsvector('english', title || ' ' || description));

-- Partial indexes for common filters
CREATE INDEX idx_stories_published ON stories(created_at DESC) 
WHERE status = 'PUBLISHED';

-- Covering indexes for frequent queries
CREATE INDEX idx_stories_list ON stories(status, created_at DESC) 
INCLUDE (id, title, cover_url, view_count);
```

### 4.3 Connection Pool Configuration

```yaml
spring:
  datasource:
    hikari:
      pool-name: N9-HikariPool
      maximum-pool-size: 50
      minimum-idle: 10
      idle-timeout: 300000      # 5 minutes
      max-lifetime: 1800000     # 30 minutes
      connection-timeout: 20000  # 20 seconds
      leak-detection-threshold: 60000  # 1 minute
      
      # Performance settings
      auto-commit: false
      connection-test-query: SELECT 1
      
      # PostgreSQL specific
      data-source-properties:
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        cachePrepStmts: true
        useServerPrepStmts: true
```

### 4.4 Read Replica Strategy

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceType.MASTER, masterDataSource());
        targetDataSources.put(DataSourceType.REPLICA, replicaDataSource());
        
        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        
        return routingDataSource;
    }
}

// Usage with annotation
@Service
public class StoryService {
    
    @ReadOnly  // Routes to replica
    public List<Story> findPublished(Pageable pageable) {
        return storyRepository.findByStatus(StoryStatus.PUBLISHED, pageable);
    }
    
    @Transactional  // Routes to master
    public Story create(CreateStoryRequest request) {
        return storyRepository.save(mapper.toEntity(request));
    }
}
```

---

## 5. API Performance

### 5.1 Response Compression

```yaml
server:
  compression:
    enabled: true
    min-response-size: 1024  # 1KB minimum
    mime-types:
      - application/json
      - application/xml
      - text/html
      - text/plain
      - text/css
      - application/javascript
```

### 5.2 Pagination Best Practices

```java
@RestController
public class StoryController {
    
    // Cursor-based pagination for infinite scroll
    @GetMapping("/stories/feed")
    public CursorPageResponse<StoryDto> getFeed(
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") int size
    ) {
        Instant cursorTime = cursor != null 
            ? Instant.parse(cursor) 
            : Instant.now();
            
        List<Story> stories = storyService.findAfter(cursorTime, size + 1);
        
        boolean hasMore = stories.size() > size;
        if (hasMore) {
            stories = stories.subList(0, size);
        }
        
        String nextCursor = hasMore 
            ? stories.get(stories.size() - 1).getCreatedAt().toString()
            : null;
            
        return CursorPageResponse.<StoryDto>builder()
            .data(stories.stream().map(mapper::toDto).toList())
            .nextCursor(nextCursor)
            .hasMore(hasMore)
            .build();
    }
    
    // Offset pagination for admin lists
    @GetMapping("/admin/stories")
    public PageResponse<StoryDto> getAdminStories(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "50") int size
    ) {
        Page<Story> result = storyService.findAll(PageRequest.of(page, size));
        return PageResponse.from(result, mapper::toDto);
    }
}
```

### 5.3 Response Field Selection

```java
@GetMapping("/stories/{id}")
public ResponseEntity<StoryDto> getStory(
    @PathVariable UUID id,
    @RequestParam(required = false) Set<String> fields
) {
    Story story = storyService.findById(id);
    
    // Dynamic field selection
    if (fields != null && !fields.isEmpty()) {
        return ResponseEntity.ok(mapper.toDto(story, fields));
    }
    
    return ResponseEntity.ok(mapper.toDto(story));
}
```

### 5.4 Async Processing

```java
@Service
public class AsyncStoryService {
    
    @Async("storyExecutor")
    public CompletableFuture<Void> processAfterPublish(Story story) {
        // Non-blocking post-publish tasks
        return CompletableFuture.allOf(
            notificationService.notifyFollowersAsync(story),
            searchService.indexAsync(story),
            analyticsService.trackPublishAsync(story),
            summaryService.generateAsync(story)
        );
    }
}

@Configuration
public class AsyncConfig {
    
    @Bean("storyExecutor")
    public Executor storyExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("story-async-");
        executor.setRejectedExecutionHandler(new CallerRunsPolicy());
        return executor;
    }
}
```

---

## 6. Content Delivery Optimization

### 6.1 CDN Configuration

```yaml
cloudfront:
  distributions:
    static:
      origins:
        - domain: static.n9.example.com
          path: /assets
      cache_policy:
        ttl:
          min: 86400      # 1 day
          max: 31536000   # 1 year
          default: 604800 # 1 week
        headers: []
        cookies: none
        query_strings: none
        
    media:
      origins:
        - domain: media.n9.example.com
          path: /covers
      cache_policy:
        ttl:
          min: 3600
          max: 2592000
          default: 86400
        headers: [Accept]
        
    api:
      origins:
        - domain: api.n9.example.com
      cache_policy:
        ttl:
          min: 0
          max: 3600
          default: 60
        headers: [Authorization, Accept-Language]
        query_strings: all
```

### 6.2 Image Optimization

```java
@Service
public class ImageOptimizationService {
    
    public ProcessedImage optimize(MultipartFile image, ImageType type) {
        BufferedImage original = ImageIO.read(image.getInputStream());
        
        // Generate multiple sizes
        Map<String, BufferedImage> variants = new HashMap<>();
        for (ImageSize size : type.getSizes()) {
            BufferedImage resized = resize(original, size);
            BufferedImage optimized = optimize(resized, size.getQuality());
            variants.put(size.getName(), optimized);
        }
        
        // Generate WebP versions
        Map<String, byte[]> webpVariants = variants.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> convertToWebP(e.getValue())
            ));
        
        // Upload all variants
        Map<String, String> urls = uploadAll(variants, webpVariants);
        
        return ProcessedImage.builder()
            .originalUrl(urls.get("original"))
            .thumbnailUrl(urls.get("thumbnail"))
            .webpUrls(webpVariants.keySet().stream()
                .collect(Collectors.toMap(k -> k, k -> urls.get(k + "_webp"))))
            .build();
    }
}
```

### 6.3 Image Sizes by Type

| Type | Sizes | Format |
|------|-------|--------|
| Story Cover | 400x600, 200x300, 100x150 | WebP, JPEG fallback |
| User Avatar | 256x256, 128x128, 64x64 | WebP, PNG fallback |
| Chapter Image | 1200xAuto, 600xAuto | WebP, JPEG fallback |
| Banner | 1920x400, 960x200 | WebP, JPEG fallback |

---

## 7. Memory Management

### 7.1 JVM Configuration

```bash
# Production JVM settings
JAVA_OPTS="
  -Xms4g
  -Xmx4g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:+UseStringDeduplication
  -XX:+ParallelRefProcEnabled
  -XX:+DisableExplicitGC
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/n9/heap-dump.hprof
  -Xlog:gc*:file=/var/log/n9/gc.log:time,uptime:filecount=5,filesize=10M
"
```

### 7.2 Object Pooling

```java
@Configuration
public class ObjectPoolConfig {
    
    @Bean
    public ObjectPool<StringBuilder> stringBuilderPool() {
        return new GenericObjectPool<>(new StringBuilderFactory(), 
            GenericObjectPoolConfig.<StringBuilder>builder()
                .maxTotal(100)
                .maxIdle(50)
                .minIdle(10)
                .build()
        );
    }
    
    @Bean
    public ObjectPool<byte[]> bufferPool() {
        return new GenericObjectPool<>(new BufferFactory(8192),
            GenericObjectPoolConfig.<byte[]>builder()
                .maxTotal(200)
                .maxIdle(100)
                .build()
        );
    }
}
```

### 7.3 Stream Processing for Large Data

```java
@Service
public class ExportService {
    
    public void exportStoriesAsCsv(OutputStream output, ExportCriteria criteria) {
        try (Stream<Story> stories = storyRepository.streamByCriteria(criteria)) {
            CSVWriter writer = new CSVWriter(new OutputStreamWriter(output));
            
            // Write header
            writer.writeNext(new String[]{"ID", "Title", "Author", "Views", "Rating"});
            
            // Stream processing - never loads all data into memory
            stories.forEach(story -> {
                writer.writeNext(new String[]{
                    story.getId().toString(),
                    story.getTitle(),
                    story.getAuthor().getDisplayName(),
                    String.valueOf(story.getViewCount()),
                    String.valueOf(story.getRating())
                });
            });
            
            writer.flush();
        }
    }
}
```

---

## 8. Search Optimization

### 8.1 Elasticsearch Configuration

```yaml
elasticsearch:
  indices:
    stories:
      settings:
        number_of_shards: 3
        number_of_replicas: 1
        refresh_interval: "30s"
        
      mappings:
        properties:
          title:
            type: text
            analyzer: n9_analyzer
            fields:
              keyword:
                type: keyword
          description:
            type: text
            analyzer: n9_analyzer
          tags:
            type: keyword
          genres:
            type: keyword
          author_name:
            type: text
            fields:
              keyword:
                type: keyword
          status:
            type: keyword
          created_at:
            type: date
          view_count:
            type: long
          rating:
            type: float
```

### 8.2 Custom Analyzer

```yaml
analysis:
  analyzer:
    n9_analyzer:
      type: custom
      tokenizer: standard
      filter:
        - lowercase
        - asciifolding
        - n9_synonyms
        - n9_stop
        - stemmer
        
  filter:
    n9_synonyms:
      type: synonym
      synonyms:
        - "fantasy,fantasia"
        - "romance,love story"
        - "sci-fi,science fiction,scifi"
        
    n9_stop:
      type: stop
      stopwords: _english_
```

### 8.3 Search Query Optimization

```java
@Service
public class SearchService {
    
    public SearchResult search(SearchRequest request) {
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        
        // Multi-match with field boosting
        MultiMatchQueryBuilder multiMatch = QueryBuilders.multiMatchQuery(
            request.getQuery(),
            "title^3",           // Title most important
            "author_name^2",     // Then author
            "description",       // Then description
            "tags"
        ).type(MultiMatchQueryBuilder.Type.BEST_FIELDS)
         .fuzziness(Fuzziness.AUTO);
        
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery()
            .must(multiMatch);
        
        // Apply filters
        if (request.getGenres() != null) {
            boolQuery.filter(QueryBuilders.termsQuery("genres", request.getGenres()));
        }
        if (request.getStatus() != null) {
            boolQuery.filter(QueryBuilders.termQuery("status", request.getStatus()));
        }
        
        // Scoring: combine relevance with popularity
        FunctionScoreQueryBuilder functionScore = QueryBuilders.functionScoreQuery(
            boolQuery,
            ScoreFunctionBuilders.fieldValueFactorFunction("view_count")
                .modifier(FieldValueFactorFunction.Modifier.LOG1P)
                .factor(0.1f)
        ).boostMode(CombineFunction.SUM);
        
        sourceBuilder.query(functionScore);
        sourceBuilder.from(request.getPage() * request.getSize());
        sourceBuilder.size(request.getSize());
        
        // Highlighting
        sourceBuilder.highlighter(new HighlightBuilder()
            .field("title")
            .field("description")
            .preTags("<em>")
            .postTags("</em>"));
        
        return executeSearch(sourceBuilder);
    }
}
```

---

## 9. Rate Limiting & Throttling

### 9.1 Rate Limit Configuration

```yaml
rate_limiting:
  enabled: true
  storage: redis
  
  default:
    requests: 100
    window: 60s
    
  tiers:
    anonymous:
      requests: 30
      window: 60s
      
    authenticated:
      requests: 100
      window: 60s
      
    premium:
      requests: 300
      window: 60s
      
    api_partner:
      requests: 1000
      window: 60s
      
  endpoints:
    "/auth/login":
      requests: 5
      window: 60s
      
    "/search":
      requests: 30
      window: 60s
      
    "/ai/*":
      requests: 10
      window: 60s
```

### 9.2 Rate Limiter Implementation

```java
@Component
public class RateLimiter {
    
    private final RedisTemplate<String, String> redis;
    
    public RateLimitResult checkLimit(String key, RateLimitConfig config) {
        String redisKey = "ratelimit:" + key;
        long now = System.currentTimeMillis();
        long windowStart = now - config.getWindowMs();
        
        // Sliding window algorithm
        redis.execute((RedisCallback<Object>) connection -> {
            // Remove old entries
            connection.zRemRangeByScore(redisKey.getBytes(), 0, windowStart);
            
            // Count current window
            Long count = connection.zCard(redisKey.getBytes());
            
            if (count < config.getMaxRequests()) {
                // Add new request
                connection.zAdd(redisKey.getBytes(), now, 
                    String.valueOf(now).getBytes());
                connection.expire(redisKey.getBytes(), 
                    config.getWindowMs() / 1000 + 1);
                return RateLimitResult.allowed(config.getMaxRequests() - count - 1);
            }
            
            return RateLimitResult.exceeded(config.getMaxRequests(), 
                windowStart + config.getWindowMs() - now);
        });
    }
}
```

---

## 10. Monitoring & Alerting

### 10.1 Performance Metrics

```java
@Aspect
@Component
public class PerformanceMetricsAspect {
    
    private final MeterRegistry registry;
    
    @Around("@annotation(Timed)")
    public Object measureExecution(ProceedingJoinPoint pjp) throws Throwable {
        Timer.Sample sample = Timer.start(registry);
        String methodName = pjp.getSignature().toShortString();
        
        try {
            Object result = pjp.proceed();
            sample.stop(Timer.builder("method.execution")
                .tag("method", methodName)
                .tag("status", "success")
                .register(registry));
            return result;
        } catch (Exception e) {
            sample.stop(Timer.builder("method.execution")
                .tag("method", methodName)
                .tag("status", "error")
                .tag("exception", e.getClass().getSimpleName())
                .register(registry));
            throw e;
        }
    }
}
```

### 10.2 Alert Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Response Time p95 | >500ms | >1s | Scale/optimize |
| Error Rate | >1% | >5% | Investigate |
| CPU Usage | >70% | >90% | Scale out |
| Memory Usage | >75% | >90% | Scale/restart |
| Cache Hit Ratio | <85% | <70% | Review cache config |
| DB Connections | >80% pool | >95% pool | Increase pool |
| Queue Depth | >1000 | >5000 | Scale consumers |

### 10.3 Dashboard Panels

```yaml
performance_dashboard:
  panels:
    - title: "API Latency Distribution"
      type: heatmap
      query: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
      
    - title: "Request Rate"
      type: graph
      query: rate(http_requests_total[1m])
      
    - title: "Error Rate"
      type: stat
      query: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
      
    - title: "Cache Performance"
      type: graph
      queries:
        - cache_hits_total
        - cache_misses_total
        
    - title: "Database Performance"
      type: graph
      queries:
        - db_query_duration_seconds
        - db_connections_active
```

---

## 11. Scalability Patterns

### 11.1 Horizontal Scaling

```yaml
kubernetes:
  deployments:
    api:
      replicas:
        min: 3
        max: 20
      autoscaling:
        cpu_threshold: 70%
        memory_threshold: 80%
        custom_metrics:
          - name: http_requests_per_second
            target: 1000
            
    workers:
      replicas:
        min: 2
        max: 10
      autoscaling:
        queue_depth_threshold: 100
```

### 11.2 Database Sharding Strategy

```yaml
sharding:
  strategy: range
  shard_key: user_id
  
  shards:
    - name: shard_0
      range: [0, 1000000]
      primary: db-shard-0.n9.internal
      
    - name: shard_1
      range: [1000001, 2000000]
      primary: db-shard-1.n9.internal
      
  routing:
    algorithm: consistent_hash
    virtual_nodes: 150
```

---

## 12. References

### 12.1 Related Specifications
- [03_NON_FUNCTIONAL_REQUIREMENTS.md](03_NON_FUNCTIONAL_REQUIREMENTS.md) - Performance requirements
- [11_DEPLOYMENT_OPERATIONS.md](11_DEPLOYMENT_OPERATIONS.md) - Infrastructure details
- [14_AI_AUTOMATION_AND_TOOLING.md](14_AI_AUTOMATION_AND_TOOLING.md) - AI optimization

### 12.2 Design Documents
- [00_PLATFORM_PERFORMANCE_ASYNC_DESIGN.md](../Design/00_PLATFORM_PERFORMANCE_ASYNC_DESIGN.md) - Async patterns
