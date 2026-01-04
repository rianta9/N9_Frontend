# AI, Automation & Tooling Specification

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2025-12-31 |
| Status | Approved |
| Owner | AI/ML Engineering Team |
| Review Cycle | Quarterly |

---

## 2. Overview

### 2.1 Purpose
This document specifies the **AI services, automation workflows, and development tooling** integrated into the N9 platform for content moderation, recommendations, translations, and operational efficiency.

### 2.2 Scope
- AI-powered content moderation
- Recommendation engine
- Translation services
- Writing assistance tools
- Automation workflows
- Development tooling

### 2.3 AI Service Providers

| Provider | Use Cases | Fallback |
|----------|-----------|----------|
| OpenAI GPT-4 | Writing assistance, summarization | Azure OpenAI |
| Azure AI | Content moderation, sentiment | OpenAI |
| Claude (Anthropic) | Complex content review | GPT-4 |
| Custom ML Models | Recommendations | Content-based rules |

---

## 3. Content Moderation System

### 3.1 Multi-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Content Submission                        │
└─────────────────────┬───────────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 1: Rule-Based Filtering                   │
│   • Keyword blocklists    • Regex patterns                   │
│   • Link detection        • Spam signatures                  │
│   Latency: <10ms          Pass Rate: ~95%                    │
└─────────────────────┬───────────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 2: ML Classification                      │
│   • Toxicity detection    • Category classification          │
│   • Language detection    • Adult content detection          │
│   Latency: <100ms         Pass Rate: ~98%                    │
└─────────────────────┬───────────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 3: LLM Review (Flagged Only)              │
│   • Context analysis      • Nuance detection                 │
│   • Appeal processing     • Edge case handling               │
│   Latency: <2s            Usage: ~2% of content              │
└─────────────────────┬───────────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Layer 4: Human Review (Escalated)               │
│   • Final decisions       • Policy updates                   │
│   • Training data         • Appeals                          │
│   SLA: <24h               Usage: ~0.1% of content            │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Moderation Categories

| Category | Severity | Auto-Action | Appeal |
|----------|----------|-------------|--------|
| Hate Speech | Critical | Block + Review | Yes |
| Violence | Critical | Block + Review | Yes |
| Adult/Sexual | High | Age-gate or Block | Yes |
| Harassment | High | Flag + Notify | Yes |
| Spam | Medium | Block | No |
| Self-Harm | Critical | Block + Support | Yes |
| Copyright | High | DMCA Process | Yes |
| Misinformation | Medium | Label | Yes |

### 3.3 Moderation API

```java
@Service
public class ModerationService {
    
    @Value("${moderation.threshold.auto-reject}")
    private double autoRejectThreshold = 0.95;
    
    @Value("${moderation.threshold.review}")
    private double reviewThreshold = 0.70;
    
    public ModerationResult moderate(ContentSubmission content) {
        // Layer 1: Rule-based
        RuleResult ruleResult = ruleEngine.check(content);
        if (ruleResult.isBlocked()) {
            return ModerationResult.rejected(ruleResult.getReason());
        }
        
        // Layer 2: ML Classification
        MLClassification mlResult = mlClassifier.classify(content);
        if (mlResult.getMaxScore() > autoRejectThreshold) {
            return ModerationResult.rejected(mlResult.getCategory());
        }
        
        // Layer 3: LLM Review (if uncertain)
        if (mlResult.getMaxScore() > reviewThreshold) {
            LLMReview llmReview = llmReviewer.review(content, mlResult);
            if (llmReview.isRejected()) {
                return ModerationResult.rejected(llmReview.getReason());
            }
            if (llmReview.needsHumanReview()) {
                return ModerationResult.pendingReview(llmReview);
            }
        }
        
        return ModerationResult.approved();
    }
}
```

### 3.4 Moderation Metrics

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| False Positive Rate | <2% | >5% |
| False Negative Rate | <0.1% | >0.5% |
| Auto-moderation Rate | >98% | <95% |
| Average Latency | <200ms | >500ms |
| Human Review Queue | <100 | >500 |

---

## 4. Recommendation Engine

### 4.1 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Recommendation Engine                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Collaborative│  │Content-Based│  │ Hybrid Ensemble    │  │
│  │   Filtering  │  │  Filtering  │  │   (Final Rank)     │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│         │                │                     │             │
│         ▼                ▼                     ▼             │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Feature Store (Redis)                       ││
│  │  • User embeddings    • Story embeddings                 ││
│  │  • Interaction matrix • Real-time signals                ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Recommendation Types

| Type | Algorithm | Use Case |
|------|-----------|----------|
| For You | Hybrid CF + CB | Home feed |
| Similar Stories | Item-based CF | Story page |
| Popular | Trending score | Discovery |
| New Releases | Time-decay | Fresh content |
| By Author | Author similarity | Author page |
| Continue Reading | Recency + Progress | Return users |

### 4.3 Feature Signals

```yaml
user_features:
  explicit:
    - preferred_genres
    - favorite_authors
    - reading_speed
    - subscription_tier
  implicit:
    - reading_history
    - completion_rates
    - time_spent
    - scroll_depth
    - interaction_patterns

story_features:
  static:
    - genres
    - tags
    - word_count
    - chapter_count
    - author_reputation
  dynamic:
    - view_velocity
    - completion_rate
    - rating_score
    - comment_sentiment
    - share_rate
```

### 4.4 Scoring Formula

```python
def calculate_recommendation_score(user, story):
    # Collaborative filtering score (user-item similarity)
    cf_score = collaborative_model.predict(user.id, story.id)
    
    # Content-based score (feature similarity)
    cb_score = cosine_similarity(user.embedding, story.embedding)
    
    # Popularity boost (with decay)
    popularity = story.views * time_decay(story.published_at)
    
    # Freshness boost for new content
    freshness = freshness_score(story.published_at)
    
    # Author affinity
    author_score = user.author_affinities.get(story.author_id, 0)
    
    # Final hybrid score
    final_score = (
        0.4 * cf_score +
        0.3 * cb_score +
        0.15 * normalize(popularity) +
        0.1 * freshness +
        0.05 * author_score
    )
    
    # Diversity penalty (reduce similar content)
    final_score *= diversity_factor(user.recent_recommendations, story)
    
    return final_score
```

### 4.5 Real-time Updates

```java
@Component
public class RecommendationUpdater {
    
    @EventListener
    public void onUserInteraction(UserInteractionEvent event) {
        // Update user embedding incrementally
        userEmbeddingService.updateIncremental(
            event.getUserId(),
            event.getStoryId(),
            event.getInteractionType()
        );
        
        // Invalidate cached recommendations
        recommendationCache.invalidate(event.getUserId());
        
        // Update real-time signals
        signalStore.record(event);
    }
    
    @Scheduled(cron = "0 0 3 * * *") // Daily at 3 AM
    public void rebuildEmbeddings() {
        // Full model retraining
        embeddingService.rebuildAll();
    }
}
```

---

## 5. Translation Services

### 5.1 Translation Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Source    │───▶│  Language   │───▶│ Translation │───▶│   Cache     │
│   Content   │    │  Detection  │    │   Engine    │    │   Store     │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                             │
                                             ▼
                                      ┌─────────────┐
                                      │   Quality   │
                                      │    Check    │
                                      └─────────────┘
```

### 5.2 Supported Languages

| Tier | Languages | Quality | Response Time |
|------|-----------|---------|---------------|
| Tier 1 | EN, VI, ZH, JA, KO | High | <1s |
| Tier 2 | TH, ID, MS, ES, FR | Medium-High | <2s |
| Tier 3 | Others (50+) | Medium | <5s |

### 5.3 Translation API

```java
@Service
public class TranslationService {
    
    public TranslatedContent translate(
        String content,
        String sourceLanguage,
        String targetLanguage
    ) {
        // Check cache first
        String cacheKey = generateCacheKey(content, sourceLanguage, targetLanguage);
        TranslatedContent cached = cache.get(cacheKey);
        if (cached != null) return cached;
        
        // Detect language if not provided
        if (sourceLanguage == null) {
            sourceLanguage = languageDetector.detect(content);
        }
        
        // Select translation provider
        TranslationProvider provider = selectProvider(sourceLanguage, targetLanguage);
        
        // Translate with retry
        TranslatedContent result = retryTemplate.execute(ctx -> 
            provider.translate(content, sourceLanguage, targetLanguage)
        );
        
        // Quality check
        double qualityScore = qualityChecker.evaluate(content, result);
        result.setQualityScore(qualityScore);
        
        // Cache result
        cache.put(cacheKey, result, Duration.ofDays(30));
        
        return result;
    }
}
```

### 5.4 Chapter Translation

```yaml
chapter_translation:
  chunking:
    max_chunk_size: 4000  # tokens
    overlap: 100          # context preservation
  
  consistency:
    terminology_glossary: true
    character_name_mapping: true
    style_preservation: true
  
  caching:
    strategy: segment_level
    ttl: 30_days
    invalidation: on_source_update
```

---

## 6. Writing Assistance

### 6.1 Features

| Feature | Description | Model |
|---------|-------------|-------|
| Grammar Check | Real-time grammar correction | Custom + GPT |
| Style Suggestions | Writing style improvements | GPT-4 |
| Plot Assistance | Story arc suggestions | Claude |
| Character Development | Character consistency | GPT-4 |
| World Building | Setting enrichment | Claude |
| Summary Generation | Chapter/story summaries | GPT-4 |

### 6.2 Writing Assistant API

```http
POST /v1/writing/assist
Content-Type: application/json

{
  "content": "The hero walked to the castle...",
  "assistanceType": "STYLE_IMPROVEMENT",
  "context": {
    "genre": "FANTASY",
    "tone": "EPIC",
    "previousChapters": ["chapter-1-id", "chapter-2-id"]
  }
}
```

**Response:**
```json
{
  "data": {
    "suggestions": [
      {
        "type": "ENHANCEMENT",
        "original": "walked to",
        "suggested": "strode purposefully toward",
        "explanation": "More vivid verb choice for epic fantasy tone"
      }
    ],
    "overallScore": 7.5,
    "improvements": ["pacing", "descriptive language"]
  }
}
```

### 6.3 AI Summary Generation

```java
@Service
public class SummaryService {
    
    public StorySummary generateSummary(Story story) {
        // Gather chapter contents
        List<Chapter> chapters = chapterRepository.findByStoryId(story.getId());
        
        // Build context-aware prompt
        String prompt = buildSummaryPrompt(story, chapters);
        
        // Generate with appropriate model
        AIResponse response = aiService.generate(
            AIRequest.builder()
                .model("gpt-4")
                .prompt(prompt)
                .maxTokens(500)
                .temperature(0.3)
                .build()
        );
        
        return StorySummary.builder()
            .storyId(story.getId())
            .shortSummary(response.extractSection("short"))
            .longSummary(response.extractSection("long"))
            .keyCharacters(response.extractList("characters"))
            .themes(response.extractList("themes"))
            .generatedAt(Instant.now())
            .build();
    }
}
```

---

## 7. Automation Workflows

### 7.1 Event-Driven Automation

```yaml
workflows:
  new_chapter_published:
    trigger: chapter.published
    actions:
      - notify_followers
      - update_story_stats
      - generate_summary
      - index_for_search
      - check_scheduled_promotions
  
  author_milestone:
    trigger: author.milestone_reached
    conditions:
      - milestone_type: FOLLOWERS
        threshold: [100, 1000, 10000]
    actions:
      - send_congratulations
      - award_badge
      - notify_followers
  
  content_flagged:
    trigger: content.flagged
    actions:
      - run_ai_moderation
      - create_review_task
      - notify_author_if_rejected
```

### 7.2 Scheduled Jobs

| Job | Schedule | Description |
|-----|----------|-------------|
| `RebuildRecommendations` | Daily 3 AM | Full recommendation rebuild |
| `CleanupExpiredSessions` | Hourly | Remove expired sessions |
| `ProcessPayouts` | Weekly | Process author payouts |
| `GenerateTrending` | Every 15 min | Update trending lists |
| `SyncSearchIndex` | Every 5 min | Sync Elasticsearch |
| `BackupDatabase` | Daily 2 AM | Database backup |
| `PurgeDeletedContent` | Weekly | Hard delete soft-deleted |

### 7.3 Notification Automation

```java
@Component
public class NotificationAutomation {
    
    @EventListener
    public void onNewChapter(ChapterPublishedEvent event) {
        Story story = event.getStory();
        Chapter chapter = event.getChapter();
        
        // Get followers interested in notifications
        List<UUID> followers = followService.getFollowersWithNotification(
            story.getAuthorId(),
            NotificationType.NEW_CHAPTER
        );
        
        // Batch create notifications
        notificationService.batchCreate(
            followers,
            NotificationTemplate.NEW_CHAPTER,
            Map.of(
                "storyTitle", story.getTitle(),
                "chapterTitle", chapter.getTitle(),
                "authorName", story.getAuthor().getDisplayName()
            )
        );
        
        // Send push notifications
        pushService.sendBatch(followers, PushTemplate.NEW_CHAPTER, Map.of(
            "storyId", story.getId(),
            "chapterId", chapter.getId()
        ));
    }
}
```

---

## 8. Development Tooling

### 8.1 Code Quality Tools

| Tool | Purpose | Configuration |
|------|---------|---------------|
| Checkstyle | Code style | `N9-cs-formatter.xml` |
| SpotBugs | Bug detection | Default + Security |
| PMD | Code analysis | Custom ruleset |
| SonarQube | Quality gate | Quality profiles |
| JaCoCo | Coverage | 80% minimum |

### 8.2 Testing Tools

| Tool | Purpose | Usage |
|------|---------|-------|
| JUnit 5 | Unit testing | All tests |
| Mockito | Mocking | Service tests |
| Testcontainers | Integration | Database/Redis tests |
| REST Assured | API testing | Controller tests |
| Gatling | Load testing | Performance tests |

### 8.3 Development Scripts

```powershell
# Build with all checks
.\mvnw clean verify -P quality

# Run specific test suite
.\mvnw test -Dtest=*IntegrationTest

# Generate API documentation
.\mvnw springdoc:generate

# Database migration
.\mvnw flyway:migrate

# Local development stack
docker-compose -f docker-compose.dev.yml up -d
```

### 8.4 IDE Configuration

```xml
<!-- formatter.xml -->
<profiles>
  <profile>
    <name>N9 Code Style</name>
    <settings>
      <setting id="tabSize">4</setting>
      <setting id="useSpaces">true</setting>
      <setting id="lineWidth">120</setting>
      <setting id="importOrder">java,javax,jakarta,org,com,*</setting>
    </settings>
  </profile>
</profiles>
```

---

## 9. AI Service Integration

### 9.1 Provider Abstraction

```java
public interface AIProvider {
    AIResponse complete(AIRequest request);
    AIResponse chat(List<ChatMessage> messages);
    EmbeddingResponse embed(String text);
    ModerationResponse moderate(String content);
}

@Service
public class AIServiceRouter {
    private final Map<String, AIProvider> providers;
    
    public AIResponse route(AIRequest request) {
        String preferredProvider = request.getPreferredProvider();
        AIProvider provider = providers.get(preferredProvider);
        
        try {
            return provider.complete(request);
        } catch (AIProviderException e) {
            // Fallback to alternative provider
            AIProvider fallback = getFallbackProvider(preferredProvider);
            return fallback.complete(request);
        }
    }
}
```

### 9.2 Rate Limiting & Cost Control

```yaml
ai_providers:
  openai:
    rate_limit: 10000  # requests per minute
    daily_budget: 500  # USD
    priority_queue: true
    
  azure:
    rate_limit: 5000
    daily_budget: 300
    fallback_for: openai
    
  anthropic:
    rate_limit: 1000
    daily_budget: 200
    use_cases: [content_review, complex_moderation]
```

### 9.3 Caching Strategy

```java
@Service
public class AICacheService {
    
    @Cacheable(
        value = "ai-responses",
        key = "#request.hashCode()",
        unless = "#result.isCacheable() == false"
    )
    public AIResponse getCachedOrCompute(AIRequest request) {
        return aiService.complete(request);
    }
    
    // Embedding cache with similarity threshold
    public Optional<EmbeddingResponse> findSimilarEmbedding(
        String text, 
        double similarityThreshold
    ) {
        String textHash = hashText(text);
        List<CachedEmbedding> candidates = embeddingCache.findByHashPrefix(textHash);
        
        return candidates.stream()
            .filter(c -> cosineSimilarity(c.getText(), text) > similarityThreshold)
            .findFirst()
            .map(CachedEmbedding::getEmbedding);
    }
}
```

---

## 10. Monitoring & Observability

### 10.1 AI Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `ai.request.latency` | Response time | >5s |
| `ai.request.cost` | Cost per request | Budget exceeded |
| `ai.request.error_rate` | Error percentage | >5% |
| `ai.moderation.false_positive` | False positive rate | >5% |
| `recommendation.click_rate` | CTR | <5% |

### 10.2 Dashboard

```yaml
ai_dashboard:
  panels:
    - title: "AI Request Volume"
      type: time_series
      metrics: ["ai.request.count"]
      
    - title: "Cost by Provider"
      type: pie_chart
      metrics: ["ai.cost.total"]
      group_by: provider
      
    - title: "Moderation Accuracy"
      type: gauge
      metrics: ["ai.moderation.accuracy"]
      thresholds: [0.95, 0.98]
```

---

## 11. References

### 11.1 Related Specifications
- [08_API_STANDARDS.md](08_API_STANDARDS.md) - API design standards
- [09_ERROR_HANDLING.md](09_ERROR_HANDLING.md) - Error handling for AI failures
- [11_DEPLOYMENT_OPERATIONS.md](11_DEPLOYMENT_OPERATIONS.md) - AI service deployment

### 11.2 Design Documents
- [08_RECOMMENDATION_COMPONENT.md](../Design/Components/08_RECOMMENDATION_COMPONENT.md)
- [10_MODERATION_ADMIN_COMPONENT.md](../Design/Components/10_MODERATION_ADMIN_COMPONENT.md)
