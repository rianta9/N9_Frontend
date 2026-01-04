# Testing Strategy Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2025-12-31 |
| Status | Approved |
| Owner | Quality Engineering Team |
| Review Cycle | Quarterly |

---

## 2. Overview

### 2.1 Purpose
This document defines the **comprehensive testing strategy** for the N9 platform, including test types, coverage requirements, tools, and quality gates.

### 2.2 Testing Pyramid

```
┌─────────────────────────────────────────────────────────────────────┐
│                      TESTING PYRAMID                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                          ┌─────┐                                     │
│                         /  E2E  \         5% - Slow, Expensive       │
│                        /─────────\                                   │
│                       /Integration\       15% - Medium Speed         │
│                      /─────────────\                                 │
│                     /    Service    \     20% - Fast                 │
│                    /─────────────────\                               │
│                   /       Unit        \   60% - Very Fast            │
│                  /─────────────────────\                             │
│                                                                      │
│  Execution Time:  ◀──────────────────────────────▶                  │
│                   Fast                        Slow                   │
│                                                                      │
│  Test Count:      ◀──────────────────────────────▶                  │
│                   Many                         Few                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Testing Goals

| Goal | Target | Measurement |
|------|--------|-------------|
| **Code Coverage** | ≥ 80% | Line + Branch coverage |
| **Mutation Score** | ≥ 70% | PIT mutation testing |
| **Test Reliability** | ≥ 99% | Flaky test rate |
| **Test Speed** | < 10 min | Full CI pipeline |
| **Bug Escape Rate** | < 5% | Bugs found in production |

---

## 3. Test Types

### 3.1 Unit Tests

| Aspect | Specification |
|--------|---------------|
| **Scope** | Single class/method |
| **Dependencies** | All mocked |
| **Database** | No |
| **Network** | No |
| **Speed** | < 100ms per test |
| **Coverage Target** | 80% |

#### 3.1.1 Unit Test Conventions

```java
@ExtendWith(MockitoExtension.class)
class StoryServiceTest {
    
    @Mock
    private StoryRepository storyRepository;
    
    @Mock
    private EventPublisher eventPublisher;
    
    @InjectMocks
    private StoryServiceImpl storyService;
    
    @Test
    @DisplayName("should create story with valid input")
    void createStory_WithValidInput_ShouldSucceed() {
        // Given
        CreateStoryRequest request = createValidRequest();
        when(storyRepository.save(any())).thenReturn(createStory());
        
        // When
        StoryResponse result = storyService.createStory(request);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getTitle()).isEqualTo(request.getTitle());
        verify(eventPublisher).publish(any(StoryCreatedEvent.class));
    }
    
    @Test
    @DisplayName("should throw exception when title is empty")
    void createStory_WithEmptyTitle_ShouldThrow() {
        // Given
        CreateStoryRequest request = createRequestWithEmptyTitle();
        
        // When & Then
        assertThatThrownBy(() -> storyService.createStory(request))
            .isInstanceOf(ValidationException.class)
            .hasMessageContaining("title");
    }
}
```

### 3.2 Integration Tests

| Aspect | Specification |
|--------|---------------|
| **Scope** | Multiple components |
| **Dependencies** | Real (containerized) |
| **Database** | TestContainers |
| **Network** | WireMock for externals |
| **Speed** | < 5s per test |
| **Coverage Target** | 70% |

#### 3.2.1 Integration Test Setup

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class StoryControllerIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("n9_test");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private StoryRepository storyRepository;
    
    @BeforeEach
    void setUp() {
        storyRepository.deleteAll();
    }
    
    @Test
    @DisplayName("should create and retrieve story")
    void createAndRetrieveStory() {
        // Create
        CreateStoryRequest request = new CreateStoryRequest("Test Story", "Description");
        ResponseEntity<StoryResponse> createResponse = restTemplate.postForEntity(
            "/v1/stories", request, StoryResponse.class);
        
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        String storyId = createResponse.getBody().getId();
        
        // Retrieve
        ResponseEntity<StoryResponse> getResponse = restTemplate.getForEntity(
            "/v1/stories/{id}", StoryResponse.class, storyId);
        
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().getTitle()).isEqualTo("Test Story");
    }
}
```

### 3.3 Service (Component) Tests

| Aspect | Specification |
|--------|---------------|
| **Scope** | Single service with real DB |
| **Dependencies** | Selective mocking |
| **Database** | In-memory or TestContainers |
| **Network** | Mocked externals |
| **Speed** | < 1s per test |

### 3.4 Contract Tests

| Aspect | Specification |
|--------|---------------|
| **Scope** | API contracts |
| **Tool** | Spring Cloud Contract |
| **Purpose** | Consumer-driven contracts |

#### 3.4.1 Contract Definition

```groovy
Contract.make {
    description "should return story by id"
    request {
        method GET()
        url "/v1/stories/550e8400-e29b-41d4-a716-446655440000"
        headers {
            header 'Authorization': 'Bearer valid-token'
        }
    }
    response {
        status OK()
        headers {
            contentType applicationJson()
        }
        body([
            data: [
                id: "550e8400-e29b-41d4-a716-446655440000",
                type: "story",
                attributes: [
                    title: $(anyNonBlankString()),
                    status: $(anyOf("DRAFT", "PUBLISHED"))
                ]
            ]
        ])
    }
}
```

### 3.5 End-to-End Tests

| Aspect | Specification |
|--------|---------------|
| **Scope** | Complete user flows |
| **Tool** | Playwright / Cypress |
| **Environment** | Staging-like |
| **Speed** | < 2 min per test |
| **Count** | Critical paths only |

#### 3.5.1 E2E Test Example

```typescript
// Playwright E2E test
test.describe('Story Publication Flow', () => {
  test('author can publish a story', async ({ page }) => {
    // Login as author
    await page.goto('/login');
    await page.fill('[data-testid="email"]', 'author@test.com');
    await page.fill('[data-testid="password"]', 'password123');
    await page.click('[data-testid="login-button"]');
    
    // Navigate to story
    await page.goto('/author/stories');
    await page.click('[data-testid="story-draft-1"]');
    
    // Publish
    await page.click('[data-testid="publish-button"]');
    await page.click('[data-testid="confirm-publish"]');
    
    // Verify
    await expect(page.locator('[data-testid="status-badge"]'))
      .toHaveText('Published');
  });
});
```

---

## 4. Test Coverage Requirements

### 4.1 Coverage by Layer

| Layer | Line Coverage | Branch Coverage | Mutation |
|-------|---------------|-----------------|----------|
| **Controllers** | 90% | 85% | 75% |
| **Services** | 85% | 80% | 70% |
| **Repositories** | 70% | 60% | N/A |
| **Domain** | 95% | 90% | 80% |
| **Utils** | 80% | 75% | 65% |

### 4.2 Coverage by Component

| Component | Minimum Coverage |
|-----------|------------------|
| Stories | 85% |
| Users | 85% |
| Payments | 90% |
| Chapters | 80% |
| Interactions | 75% |
| Search | 70% |

### 4.3 Exclusions from Coverage

| Exclusion | Reason |
|-----------|--------|
| DTOs/POJOs | No logic |
| Configuration classes | Framework-generated |
| Generated code | Lombok, MapStruct |
| Main application class | Bootstrap |

---

## 5. Test Data Management

### 5.1 Test Data Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TEST DATA STRATEGY                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │   Builders   │────▶│   Fixtures   │────▶│   Factories  │        │
│  │              │     │              │     │              │        │
│  │ Programmatic │     │  JSON/YAML   │     │   Random     │        │
│  │ construction │     │    files     │     │ generation   │        │
│  └──────────────┘     └──────────────┘     └──────────────┘        │
│                                                                      │
│  Unit Tests:        Builders (in-memory, fast)                      │
│  Integration Tests: Fixtures (consistent, repeatable)               │
│  E2E Tests:         Factories (realistic, varied)                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Test Data Builders

```java
public class StoryTestDataBuilder {
    
    private String id = UUID.randomUUID().toString();
    private String title = "Test Story";
    private String authorId = UUID.randomUUID().toString();
    private StoryStatus status = StoryStatus.DRAFT;
    
    public static StoryTestDataBuilder aStory() {
        return new StoryTestDataBuilder();
    }
    
    public StoryTestDataBuilder withTitle(String title) {
        this.title = title;
        return this;
    }
    
    public StoryTestDataBuilder published() {
        this.status = StoryStatus.PUBLISHED;
        return this;
    }
    
    public Story build() {
        return new Story(id, title, authorId, status);
    }
    
    public CreateStoryRequest buildRequest() {
        return new CreateStoryRequest(title);
    }
}

// Usage
Story story = aStory().withTitle("My Story").published().build();
```

### 5.3 Fixture Management

```yaml
# fixtures/stories.yml
stories:
  published_story:
    id: "550e8400-e29b-41d4-a716-446655440000"
    title: "Published Test Story"
    status: PUBLISHED
    author_id: "author-uuid"
    
  draft_story:
    id: "660e8400-e29b-41d4-a716-446655440001"
    title: "Draft Test Story"
    status: DRAFT
    author_id: "author-uuid"
```

### 5.4 Database State Management

| Approach | Use Case | Tool |
|----------|----------|------|
| Rollback | Fast isolation | @Transactional |
| Truncate | Clean state | @Sql scripts |
| Per-test DB | Full isolation | TestContainers |
| Shared DB | Speed | Careful ordering |

---

## 6. Mocking Strategy

### 6.1 Mock vs Real Dependencies

| Dependency | Unit Test | Integration Test | E2E |
|------------|-----------|------------------|-----|
| Repository | Mock | Real (Testcontainers) | Real |
| Cache (Redis) | Mock | Real (Testcontainers) | Real |
| Message Queue | Mock | Real (Testcontainers) | Real |
| External APIs | Mock | WireMock | Real (Sandbox) |
| Time | Mock (Clock) | Mock (Clock) | Real |
| Random | Mock | Mock | Real |

### 6.2 WireMock for External Services

```java
@WireMockTest(httpPort = 8089)
class PaymentGatewayIntegrationTest {
    
    @BeforeEach
    void setupStubs() {
        stubFor(post(urlEqualTo("/v1/payment_intents"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "id": "pi_test",
                        "status": "requires_payment_method",
                        "client_secret": "secret"
                    }
                    """)));
    }
    
    @Test
    void shouldCreatePaymentIntent() {
        // Test implementation
    }
}
```

### 6.3 Time Mocking

```java
@Test
void shouldExpireTokenAfter24Hours() {
    // Given
    Clock fixedClock = Clock.fixed(
        Instant.parse("2025-01-01T00:00:00Z"),
        ZoneOffset.UTC);
    
    TokenService tokenService = new TokenService(fixedClock);
    Token token = tokenService.createToken(user);
    
    // When - advance time 25 hours
    Clock futureTime = Clock.fixed(
        Instant.parse("2025-01-02T01:00:00Z"),
        ZoneOffset.UTC);
    tokenService.setClock(futureTime);
    
    // Then
    assertThat(tokenService.isValid(token)).isFalse();
}
```

---

## 7. Performance Testing

### 7.1 Performance Test Types

| Type | Tool | Purpose | Frequency |
|------|------|---------|-----------|
| **Load** | Gatling | Normal load behavior | Weekly |
| **Stress** | Gatling | Breaking point | Monthly |
| **Spike** | Gatling | Sudden load increase | Monthly |
| **Endurance** | Gatling | Memory leaks | Monthly |
| **Benchmark** | JMH | Micro-optimizations | On demand |

### 7.2 Load Test Scenarios

```scala
// Gatling load test
class StoryLoadSimulation extends Simulation {
  
  val httpProtocol = http
    .baseUrl("https://api.staging.n9.example.com")
    .acceptHeader("application/json")
  
  val browseStories = scenario("Browse Stories")
    .exec(http("Get Stories List")
      .get("/v1/stories")
      .queryParam("page", "1")
      .queryParam("size", "20")
      .check(status.is(200)))
    .pause(2)
    .exec(http("Get Story Detail")
      .get("/v1/stories/${storyId}")
      .check(status.is(200)))
  
  setUp(
    browseStories.inject(
      rampUsers(100).during(30.seconds),
      constantUsersPerSec(50).during(5.minutes)
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile(95).lt(500),
     global.successfulRequests.percent.gt(99)
   )
}
```

### 7.3 Performance Baselines

| Endpoint | P50 | P95 | P99 | Max RPS |
|----------|-----|-----|-----|---------|
| GET /stories | 50ms | 150ms | 300ms | 1000 |
| GET /stories/{id} | 30ms | 100ms | 200ms | 2000 |
| POST /stories | 100ms | 300ms | 500ms | 200 |
| GET /chapters/{id} | 40ms | 120ms | 250ms | 1500 |
| POST /comments | 80ms | 200ms | 400ms | 500 |

---

## 8. Security Testing

### 8.1 Security Test Categories

| Category | Tool | Frequency |
|----------|------|-----------|
| **SAST** | SonarQube, Semgrep | Every commit |
| **DAST** | OWASP ZAP | Weekly |
| **Dependency Scan** | Snyk, Dependabot | Daily |
| **Secret Detection** | GitLeaks | Every commit |
| **Penetration Testing** | Manual | Quarterly |

### 8.2 OWASP Top 10 Test Coverage

| Vulnerability | Test Type | Automated |
|---------------|-----------|-----------|
| Injection | Unit + DAST | Yes |
| Broken Auth | Integration | Yes |
| Sensitive Data | SAST + Manual | Partial |
| XXE | Unit | Yes |
| Broken Access Control | Integration | Yes |
| Misconfiguration | DAST | Yes |
| XSS | DAST | Yes |
| Insecure Deserialization | SAST | Yes |
| Vulnerable Components | Dependency scan | Yes |
| Insufficient Logging | Manual | No |

### 8.3 Security Test Examples

```java
@Test
@DisplayName("should prevent SQL injection in search")
void searchStories_WithSqlInjection_ShouldSanitize() {
    // Given
    String maliciousQuery = "'; DROP TABLE stories; --";
    
    // When
    ResponseEntity<SearchResponse> response = restTemplate.getForEntity(
        "/v1/stories/search?q={query}",
        SearchResponse.class,
        maliciousQuery);
    
    // Then
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    // Verify table still exists
    assertThat(storyRepository.count()).isGreaterThan(0);
}

@Test
@DisplayName("should prevent unauthorized access to other user's data")
void getStory_AsNonOwner_ShouldDenyAccess() {
    // Given
    Story privateStory = createPrivateStory(ownerUser);
    String otherUserToken = getTokenFor(otherUser);
    
    // When
    ResponseEntity<StoryResponse> response = restTemplate.exchange(
        "/v1/stories/{id}",
        HttpMethod.GET,
        withAuth(otherUserToken),
        StoryResponse.class,
        privateStory.getId());
    
    // Then
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.FORBIDDEN);
}
```

---

## 9. Test Automation & CI/CD

### 9.1 CI Pipeline Stages

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CI PIPELINE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐            │
│  │  Build  │──▶│  Unit   │──▶│  Integ  │──▶│  SAST   │            │
│  │         │   │  Tests  │   │  Tests  │   │  Scan   │            │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘            │
│      │                                          │                   │
│      │                                          ▼                   │
│      │                                    ┌─────────┐               │
│      │                                    │ Quality │               │
│      │                                    │  Gate   │               │
│      │                                    └─────────┘               │
│      │                                          │                   │
│      │          ┌───────────────────────────────┘                   │
│      │          │                                                    │
│      ▼          ▼                                                    │
│  ┌──────────────────┐   ┌─────────────┐   ┌─────────────┐          │
│  │  Deploy Staging  │──▶│   E2E Tests │──▶│   Deploy    │          │
│  │                  │   │             │   │   Prod      │          │
│  └──────────────────┘   └─────────────┘   └─────────────┘          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 Quality Gates

| Gate | Metric | Threshold | Blocking |
|------|--------|-----------|----------|
| **Unit Tests** | Pass rate | 100% | Yes |
| **Code Coverage** | Line coverage | ≥ 80% | Yes |
| **Integration Tests** | Pass rate | 100% | Yes |
| **SAST** | Critical issues | 0 | Yes |
| **SAST** | High issues | ≤ 5 | Yes |
| **Dependency Scan** | Critical CVEs | 0 | Yes |
| **E2E Tests** | Pass rate | ≥ 95% | No |
| **Performance** | P95 regression | < 20% | No |

### 9.3 Test Execution Parallelization

```yaml
# GitHub Actions parallel test execution
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: [stories, users, payments, chapters, interactions]
    steps:
      - name: Run unit tests for ${{ matrix.module }}
        run: ./mvnw test -pl ${{ matrix.module }}
  
  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
      redis:
        image: redis:7-alpine
    steps:
      - name: Run integration tests
        run: ./mvnw verify -Pintegration-tests
```

---

## 10. Test Environments

### 10.1 Environment Configuration

| Environment | Purpose | Data | Refresh |
|-------------|---------|------|---------|
| **Local** | Development | Fixtures | On demand |
| **CI** | Automated testing | Generated | Per run |
| **Staging** | Integration + E2E | Sanitized prod subset | Weekly |
| **UAT** | User acceptance | Production clone | Monthly |
| **Performance** | Load testing | Synthetic | Per test |

### 10.2 Environment Parity

| Aspect | Local | CI | Staging | Production |
|--------|-------|-----|---------|------------|
| PostgreSQL | 16 | 16 | 16 | 16 |
| Redis | 7 | 7 | 7 | 7 |
| Elasticsearch | 8.x | 8.x | 8.x | 8.x |
| Java | 21 | 21 | 21 | 21 |
| External APIs | Mock | Mock | Sandbox | Live |

---

## 11. Test Documentation

### 11.1 Test Naming Convention

```
methodName_StateUnderTest_ExpectedBehavior
```

Examples:
- `createStory_WithValidInput_ShouldReturnCreatedStory`
- `createStory_WithEmptyTitle_ShouldThrowValidationException`
- `getStory_WhenNotExists_ShouldReturn404`

### 11.2 Test Documentation Requirements

| Element | Required | Example |
|---------|----------|---------|
| `@DisplayName` | Yes | "should create story with valid input" |
| Arrange/Act/Assert comments | Yes | // Given // When // Then |
| Test class description | Yes | Class-level Javadoc |
| Edge case documentation | Yes | In test method |

---

## 12. Flaky Test Management

### 12.1 Flaky Test Policy

| Action | Condition |
|--------|-----------|
| **Quarantine** | 3+ failures in a week |
| **Investigate** | Within 2 business days |
| **Fix or Remove** | Within 1 week |
| **Retest** | 10 consecutive passes to un-quarantine |

### 12.2 Common Flaky Test Causes

| Cause | Prevention |
|-------|------------|
| Time dependencies | Use mocked clocks |
| Order dependencies | Independent tests |
| Resource leaks | Proper cleanup |
| Race conditions | Proper synchronization |
| External dependencies | Mock external services |
| Data pollution | Transaction rollback |

---

## 13. Test Reporting

### 13.1 Test Report Contents

| Section | Details |
|---------|---------|
| **Summary** | Pass/fail counts, duration |
| **Coverage** | Line, branch, mutation |
| **Failures** | Stack traces, screenshots |
| **Trends** | Historical comparison |
| **Flaky Tests** | Identified flaky tests |

### 13.2 Reporting Tools

| Tool | Purpose | Integration |
|------|---------|-------------|
| **JUnit Reports** | Test results | CI/CD |
| **JaCoCo** | Code coverage | SonarQube |
| **Allure** | Rich reports | CI/CD |
| **SonarQube** | Quality dashboard | PR checks |

---

## 14. References

### 14.1 Related Specifications
- [11_DEPLOYMENT_OPERATIONS.md](11_DEPLOYMENT_OPERATIONS.md)
- [05_SECURITY_AND_COMPLIANCE.md](05_SECURITY_AND_COMPLIANCE.md)

### 14.2 External References
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Testcontainers](https://www.testcontainers.org/)
- [Spring Testing](https://docs.spring.io/spring-framework/reference/testing.html)
- [Gatling Documentation](https://gatling.io/docs/)
