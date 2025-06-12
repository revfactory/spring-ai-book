# 데이터 보안·비용 관리

## 개요

AI 애플리케이션을 개발하고 운영하는 데 있어 데이터 보안과 비용 관리는 핵심적인 고려사항입니다. 특히 Spring AI와 같은 프레임워크를 사용해 외부 AI 서비스를 연동할 때는 데이터 유출 위험과 예측 불가능한 비용 발생에 주의해야 합니다. 이 장에서는 Spring AI 1.0.0 GA의 최신 보안 기능과 비용 관리 도구를 활용하여 안전하고 경제적인 AI 애플리케이션을 구축하는 방법을 살펴보겠습니다.

## 데이터 보안 고려사항

### AI 애플리케이션의 보안 위험

AI 애플리케이션, 특히 외부 API에 의존하는 애플리케이션은 다음과 같은 보안 위험에 노출될 수 있습니다:

1. **데이터 전송 중 유출**: API 호출 과정에서 민감한 데이터가 유출될 위험
2. **프롬프트 주입 공격**: 사용자 입력을 통한 악의적인 프롬프트 주입
3. **개인정보 노출**: 프롬프트나 응답에 포함된 개인식별정보(PII) 유출
4. **API 키 노출**: 인증 자격 증명의 부적절한 관리로 인한 무단 접근
5. **콘텐츠 안전성**: AI 모델이 유해하거나 부적절한 콘텐츠를 생성할 위험
6. **데이터 거버넌스**: 서로 다른 AI 제공업체의 데이터 사용 정책

## Spring AI의 보안 기능

### API 키 관리와 Cloud Bindings

Spring AI 1.0.0 GA는 Cloud Bindings를 통한 향상된 API 키 관리를 제공합니다:

```xml
<!-- Cloud Bindings 의존성 추가 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-spring-cloud-bindings</artifactId>
</dependency>
```

Cloud Bindings를 사용한 안전한 구성:

```yaml
# application.yml
spring:
  ai:
    cloud:
      bindings:
        openai:
          enabled: true
          # 환경별 바인딩 구성
```

환경 변수를 통한 안전한 API 키 관리:

```bash
# 개발 환경
export SPRING_AI_OPENAI_API_KEY=sk-...
export SPRING_AI_ANTHROPIC_API_KEY=sk-ant-...

# 프로덕션 환경 (Kubernetes Secrets)
kubectl create secret generic ai-api-keys \
  --from-literal=openai-key=sk-... \
  --from-literal=anthropic-key=sk-ant-...
```

### 콘텐츠 모더레이션

Spring AI는 OpenAI의 Moderation API를 지원하여 유해한 콘텐츠를 탐지할 수 있습니다:

```java
@Service
public class ContentModerationService {

    private final ModerationModel moderationModel;
    private final ChatClient chatClient;

    public ContentModerationService(ModerationModel moderationModel, ChatClient chatClient) {
        this.moderationModel = moderationModel;
        this.chatClient = chatClient;
    }

    public String processWithModeration(String userInput) {
        // 1. 입력 내용 모더레이션
        ModerationPrompt moderationPrompt = new ModerationPrompt(userInput);
        ModerationResponse moderationResponse = moderationModel.call(moderationPrompt);
        
        Moderation moderation = moderationResponse.getResult().getOutput();
        
        // 2. 유해 콘텐츠 검사
        for (ModerationResult result : moderation.getResults()) {
            if (result.isFlagged()) {
                Categories categories = result.getCategories();
                if (categories.isHate() || categories.isHarassment() || categories.isViolence()) {
                    return "요청하신 내용에 부적절한 내용이 포함되어 있어 처리할 수 없습니다.";
                }
                
                // 세부 카테고리 점수 확인
                CategoryScores scores = result.getCategoryScores();
                if (scores.getHate() > 0.8 || scores.getViolence() > 0.8) {
                    logSecurityIncident(userInput, categories);
                    return "보안 정책에 따라 요청을 처리할 수 없습니다.";
                }
            }
        }
        
        // 3. 안전한 콘텐츠인 경우 AI 처리
        return chatClient.prompt()
            .user(userInput)
            .call()
            .content();
    }
    
    private void logSecurityIncident(String input, Categories categories) {
        // 보안 인시던트 로깅
        log.warn("Security incident detected: categories={}, input_length={}", 
                categories, input.length());
    }
}
```

### SafeGuard Advisor

Spring AI의 SafeGuard Advisor를 사용하여 프롬프트와 응답을 모니터링할 수 있습니다:

```java
@Configuration
public class SecurityConfig {

    @Bean
    public ChatClient secureChatClient(ChatClient.Builder builder) {
        return builder
            .defaultAdvisors(
                new SafeGuardAdvisor(), // 콘텐츠 안전성 검사
                new SimpleLoggerAdvisor() // 보안 감사를 위한 로깅
            )
            .build();
    }
}
```

### 개인정보 보호 및 데이터 마스킹

개인 식별 정보(PII)를 자동으로 탐지하고 마스킹하는 서비스:

```java
@Component
public class PIIProtectionService {

    private final Map<Pattern, String> piiPatterns;
    
    public PIIProtectionService() {
        this.piiPatterns = Map.of(
            Pattern.compile("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b"), "[EMAIL]",
            Pattern.compile("\\b\\d{3}-\\d{2}-\\d{4}\\b"), "[SSN]",
            Pattern.compile("\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"), "[CREDIT_CARD]",
            Pattern.compile("\\b\\d{3}-\\d{3}-\\d{4}\\b"), "[PHONE]"
        );
    }
    
    public String maskPII(String text) {
        String maskedText = text;
        
        for (Map.Entry<Pattern, String> entry : piiPatterns.entrySet()) {
            maskedText = entry.getKey().matcher(maskedText)
                .replaceAll(entry.getValue());
        }
        
        return maskedText;
    }
    
    public boolean containsPII(String text) {
        return piiPatterns.keySet().stream()
            .anyMatch(pattern -> pattern.matcher(text).find());
    }
    
    public PIIAnalysis analyzePII(String text) {
        Map<String, Integer> detectedTypes = new HashMap<>();
        
        for (Map.Entry<Pattern, String> entry : piiPatterns.entrySet()) {
            Matcher matcher = entry.getKey().matcher(text);
            int count = 0;
            while (matcher.find()) {
                count++;
            }
            if (count > 0) {
                detectedTypes.put(entry.getValue(), count);
            }
        }
        
        return new PIIAnalysis(detectedTypes, !detectedTypes.isEmpty());
    }
    
    record PIIAnalysis(Map<String, Integer> detectedTypes, boolean hasPII) {}
}
```

### 데이터 주권과 지역별 배포

Spring AI는 다양한 AI 제공업체의 지역별 엔드포인트를 지원합니다:

```java
@Configuration
public class RegionalAIConfig {

    @Value("${app.data-region:US}")
    private String dataRegion;

    @Bean
    @ConditionalOnProperty(name = "app.data-region", havingValue = "EU")
    public ChatClient europeanChatClient() {
        return ChatClient.builder(createEuropeanChatModel())
            .build();
    }
    
    @Bean
    @ConditionalOnProperty(name = "app.data-region", havingValue = "US")
    public ChatClient usChatClient() {
        return ChatClient.builder(createUSChatModel())
            .build();
    }
    
    private ChatModel createEuropeanChatModel() {
        // EU 데이터 센터 사용
        return OpenAiChatModel.builder()
            .openAiApi(OpenAiApi.builder()
                .baseUrl("https://api.openai.com/eu")
                .apiKey(System.getenv("OPENAI_EU_API_KEY"))
                .build())
            .build();
    }
    
    private ChatModel createUSChatModel() {
        // US 데이터 센터 사용
        return OpenAiChatModel.builder()
            .openAiApi(OpenAiApi.builder()
                .baseUrl("https://api.openai.com")
                .apiKey(System.getenv("OPENAI_US_API_KEY"))
                .build())
            .build();
    }
}
```

## 비용 관리 및 사용량 추적

### 향상된 Usage Tracking

Spring AI 1.0.0 GA는 상세한 사용량 추적을 위한 `Usage` 인터페이스를 제공합니다:

```java
@Service
public class CostTrackingService {

    private final ChatClient chatClient;
    private final CostRepository costRepository;
    private final MeterRegistry meterRegistry;

    public CostTrackingService(ChatClient chatClient, 
                              CostRepository costRepository,
                              MeterRegistry meterRegistry) {
        this.chatClient = chatClient;
        this.costRepository = costRepository;
        this.meterRegistry = meterRegistry;
    }

    public String processWithCostTracking(String userInput, String userId) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            // ChatClient 호출
            ChatResponse response = chatClient.prompt()
                .user(userInput)
                .call()
                .chatResponse();
            
            // 사용량 정보 추출
            Usage usage = response.getMetadata().getUsage();
            
            // 표준 사용량 메트릭
            int promptTokens = usage.getPromptTokens();
            int completionTokens = usage.getCompletionTokens();
            int totalTokens = usage.getTotalTokens();
            
            // 네이티브 사용량 정보 (OpenAI 전용)
            if (usage.getNativeUsage() instanceof org.springframework.ai.openai.api.OpenAiApi.Usage nativeUsage) {
                // 상세한 토큰 정보
                var promptDetails = nativeUsage.promptTokensDetails();
                var completionDetails = nativeUsage.completionTokenDetails();
                
                // 캐시된 토큰 수 (비용 절감)
                int cachedTokens = promptDetails.cachedTokens();
                
                // 추론 토큰 (o1 모델용)
                int reasoningTokens = completionDetails.reasoningTokens();
                
                // 오디오 토큰
                int audioPromptTokens = promptDetails.audioTokens();
                int audioCompletionTokens = completionDetails.audioTokens();
                
                // 상세 비용 계산
                double cost = calculateDetailedCost(
                    promptTokens - cachedTokens, // 실제 처리된 프롬프트 토큰
                    completionTokens,
                    reasoningTokens,
                    audioPromptTokens,
                    audioCompletionTokens
                );
                
                // 사용량 기록
                costRepository.recordUsage(CostRecord.builder()
                    .userId(userId)
                    .promptTokens(promptTokens)
                    .completionTokens(completionTokens)
                    .cachedTokens(cachedTokens)
                    .reasoningTokens(reasoningTokens)
                    .audioTokens(audioPromptTokens + audioCompletionTokens)
                    .totalCost(cost)
                    .timestamp(Instant.now())
                    .build());
            } else {
                // 기본 비용 계산
                double cost = calculateBasicCost(promptTokens, completionTokens);
                costRepository.recordUsage(CostRecord.builder()
                    .userId(userId)
                    .promptTokens(promptTokens)
                    .completionTokens(completionTokens)
                    .totalCost(cost)
                    .timestamp(Instant.now())
                    .build());
            }
            
            // 메트릭 업데이트
            meterRegistry.counter("ai.tokens.prompt").increment(promptTokens);
            meterRegistry.counter("ai.tokens.completion").increment(completionTokens);
            meterRegistry.counter("ai.cost.total").increment(cost);
            
            return response.getResult().getOutput().getContent();
            
        } finally {
            sample.stop(Timer.builder("ai.request.duration")
                .description("AI request duration")
                .register(meterRegistry));
        }
    }
    
    private double calculateDetailedCost(int promptTokens, int completionTokens, 
                                       int reasoningTokens, int audioPromptTokens, 
                                       int audioCompletionTokens) {
        // 2024년 OpenAI 가격 기준 (예시)
        double promptCost = promptTokens * 0.0015 / 1000; // $0.0015 per 1K tokens
        double completionCost = completionTokens * 0.002 / 1000; // $0.002 per 1K tokens
        double reasoningCost = reasoningTokens * 0.06 / 1000; // o1 reasoning tokens
        double audioCost = (audioPromptTokens + audioCompletionTokens) * 0.006 / 1000;
        
        return promptCost + completionCost + reasoningCost + audioCost;
    }
    
    private double calculateBasicCost(int promptTokens, int completionTokens) {
        return (promptTokens * 0.0015 + completionTokens * 0.002) / 1000;
    }
}
```

### 실시간 비용 모니터링 및 제한

사용자별 또는 테넌트별 비용 제한을 구현하는 서비스:

```java
@Service
public class CostControlService {

    private final ChatClient chatClient;
    private final RedisTemplate<String, String> redisTemplate;
    private final CostLimitConfig costLimitConfig;

    public CostControlService(ChatClient chatClient, 
                             RedisTemplate<String, String> redisTemplate,
                             CostLimitConfig costLimitConfig) {
        this.chatClient = chatClient;
        this.redisTemplate = redisTemplate;
        this.costLimitConfig = costLimitConfig;
    }

    public String processWithCostControl(String userInput, String userId) {
        // 1. 현재 사용량 확인
        double currentMonthlyCost = getCurrentMonthlyCost(userId);
        double userLimit = getUserCostLimit(userId);
        
        // 2. 예상 비용 계산
        int estimatedTokens = estimateTokenCount(userInput);
        double estimatedCost = estimateRequestCost(estimatedTokens);
        
        // 3. 비용 제한 확인
        if (currentMonthlyCost + estimatedCost > userLimit) {
            return String.format(
                "월간 AI 사용 한도($%.2f)를 초과할 수 있습니다. " +
                "현재 사용량: $%.2f, 예상 추가 비용: $%.2f",
                userLimit, currentMonthlyCost, estimatedCost
            );
        }
        
        // 4. 요청 처리 및 실제 비용 추적
        ChatResponse response = chatClient.prompt()
            .user(userInput)
            .call()
            .chatResponse();
        
        // 5. 실제 비용 계산 및 업데이트
        Usage usage = response.getMetadata().getUsage();
        double actualCost = calculateActualCost(usage);
        updateMonthlyCost(userId, actualCost);
        
        // 6. 사용량 경고
        double newTotalCost = currentMonthlyCost + actualCost;
        if (newTotalCost > userLimit * 0.8) { // 80% 도달시 경고
            sendCostWarning(userId, newTotalCost, userLimit);
        }
        
        return response.getResult().getOutput().getContent();
    }
    
    private double getCurrentMonthlyCost(String userId) {
        String key = "monthly_cost:" + userId + ":" + getCurrentMonth();
        String cost = redisTemplate.opsForValue().get(key);
        return cost != null ? Double.parseDouble(cost) : 0.0;
    }
    
    private void updateMonthlyCost(String userId, double additionalCost) {
        String key = "monthly_cost:" + userId + ":" + getCurrentMonth();
        redisTemplate.opsForValue().increment(key, additionalCost);
        redisTemplate.expire(key, Duration.ofDays(32)); // 월말 이후 삭제
    }
    
    private double getUserCostLimit(String userId) {
        // 사용자별 또는 플랜별 한도 반환
        return costLimitConfig.getDefaultLimit(); // 예: $100
    }
    
    private int estimateTokenCount(String text) {
        // 간단한 토큰 수 추정 (정확한 계산을 위해서는 tiktoken 등 사용)
        return text.length() / 3; // 평균적으로 3-4 문자당 1 토큰
    }
    
    private double estimateRequestCost(int estimatedTokens) {
        // 보수적인 비용 추정 (프롬프트 + 예상 응답)
        int estimatedResponseTokens = Math.min(estimatedTokens * 2, 1000);
        return (estimatedTokens * 0.0015 + estimatedResponseTokens * 0.002) / 1000;
    }
    
    private double calculateActualCost(Usage usage) {
        return (usage.getPromptTokens() * 0.0015 + usage.getCompletionTokens() * 0.002) / 1000;
    }
    
    private String getCurrentMonth() {
        return LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM"));
    }
    
    private void sendCostWarning(String userId, double currentCost, double limit) {
        // 사용자에게 비용 경고 알림 발송
        log.warn("Cost warning for user {}: ${:.2f} / ${:.2f}", userId, currentCost, limit);
    }
}
```

### 비용 최적화 전략

#### 지능형 캐싱

의미적 유사성을 기반으로 한 고급 캐싱:

```java
@Service
public class IntelligentCachingService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;
    private final double similarityThreshold = 0.85;

    public IntelligentCachingService(ChatClient chatClient, 
                                   VectorStore vectorStore,
                                   EmbeddingModel embeddingModel) {
        this.chatClient = chatClient;
        this.vectorStore = vectorStore;
        this.embeddingModel = embeddingModel;
    }

    public String getResponseWithSemanticCache(String userQuery) {
        // 1. 쿼리 임베딩 생성
        EmbeddingRequest embeddingRequest = new EmbeddingRequest(
            List.of(userQuery), OpenAiEmbeddingOptions.builder().build()
        );
        EmbeddingResponse embeddingResponse = embeddingModel.call(embeddingRequest);
        float[] queryEmbedding = embeddingResponse.getResults().get(0).getOutput();
        
        // 2. 유사한 쿼리 검색
        SearchRequest searchRequest = SearchRequest.query(userQuery)
            .withTopK(1)
            .withSimilarityThreshold(similarityThreshold);
        
        List<Document> similarQueries = vectorStore.similaritySearch(searchRequest);
        
        // 3. 캐시된 응답이 있으면 반환
        if (!similarQueries.isEmpty()) {
            Document cachedQuery = similarQueries.get(0);
            String cachedResponse = cachedQuery.getMetadata().get("response").toString();
            String originalQuery = cachedQuery.getMetadata().get("original_query").toString();
            
            log.info("Cache hit: '{}' similar to '{}'", userQuery, originalQuery);
            return cachedResponse + "\n\n[캐시된 응답]";
        }
        
        // 4. 새 응답 생성 및 캐싱
        String newResponse = chatClient.prompt()
            .user(userQuery)
            .call()
            .content();
        
        // 5. 응답을 벡터 스토어에 캐싱
        Document cacheDocument = new Document(userQuery, Map.of(
            "response", newResponse,
            "original_query", userQuery,
            "timestamp", Instant.now().toString(),
            "type", "cached_query"
        ));
        
        vectorStore.add(List.of(cacheDocument));
        
        return newResponse;
    }
}
```

#### 모델 티어링 및 폴백

비용 효율성을 위한 계층적 모델 사용:

```java
@Service
public class TieredModelService {

    private final ChatClient fastModel;    // GPT-3.5-turbo (저비용)
    private final ChatClient smartModel;   // GPT-4 (고비용)
    private final QueryComplexityAnalyzer complexityAnalyzer;

    public TieredModelService(@Qualifier("fastChatClient") ChatClient fastModel,
                             @Qualifier("smartChatClient") ChatClient smartModel,
                             QueryComplexityAnalyzer complexityAnalyzer) {
        this.fastModel = fastModel;
        this.smartModel = smartModel;
        this.complexityAnalyzer = complexityAnalyzer;
    }

    public String processWithOptimalModel(String userQuery) {
        QueryComplexity complexity = complexityAnalyzer.analyze(userQuery);
        
        try {
            switch (complexity) {
                case SIMPLE -> {
                    // 간단한 쿼리는 빠른 모델 사용
                    return fastModel.prompt()
                        .user(userQuery)
                        .call()
                        .content();
                }
                case COMPLEX -> {
                    // 복잡한 쿼리는 스마트 모델 사용
                    return smartModel.prompt()
                        .user(userQuery)
                        .call()
                        .content();
                }
                case MEDIUM -> {
                    // 중간 복잡도는 빠른 모델 먼저 시도
                    try {
                        String response = fastModel.prompt()
                            .user(userQuery)
                            .call()
                            .content();
                        
                        // 응답 품질 검증
                        if (isResponseAdequate(response, userQuery)) {
                            return response;
                        } else {
                            // 품질이 부족하면 스마트 모델로 재시도
                            return smartModel.prompt()
                                .user(userQuery)
                                .call()
                                .content();
                        }
                    } catch (Exception e) {
                        // 빠른 모델 실패시 스마트 모델로 폴백
                        return smartModel.prompt()
                            .user(userQuery)
                            .call()
                            .content();
                    }
                }
            }
        } catch (Exception e) {
            // 모든 모델 실패시 기본 응답
            return "죄송합니다. 현재 AI 서비스를 이용할 수 없습니다.";
        }
        
        return "처리할 수 없는 쿼리입니다.";
    }
    
    private boolean isResponseAdequate(String response, String query) {
        // 응답 품질 평가 로직
        return response.length() > 50 && 
               !response.toLowerCase().contains("i don't know") &&
               !response.toLowerCase().contains("cannot help");
    }
    
    enum QueryComplexity {
        SIMPLE,   // FAQ, 간단한 정보 요청
        MEDIUM,   // 일반적인 질문
        COMPLEX   // 추론, 분석, 복잡한 문제 해결
    }
    
    @Component
    static class QueryComplexityAnalyzer {
        
        public QueryComplexity analyze(String query) {
            // 간단한 휴리스틱 기반 복잡도 분석
            String lowerQuery = query.toLowerCase();
            
            // 복잡한 키워드 확인
            String[] complexKeywords = {"analyze", "compare", "explain why", "how does", 
                                      "what are the implications", "strategy", "algorithm"};
            if (Arrays.stream(complexKeywords).anyMatch(lowerQuery::contains)) {
                return QueryComplexity.COMPLEX;
            }
            
            // 간단한 키워드 확인
            String[] simpleKeywords = {"what is", "who is", "when", "where", "define"};
            if (Arrays.stream(simpleKeywords).anyMatch(lowerQuery::contains) && 
                query.length() < 100) {
                return QueryComplexity.SIMPLE;
            }
            
            // 기본적으로 중간 복잡도
            return QueryComplexity.MEDIUM;
        }
    }
}
```

## 보안 모니터링 및 감사

### 보안 이벤트 추적

```java
@Component
public class SecurityAuditService {

    private final ApplicationEventPublisher eventPublisher;
    private final SecurityMetricsCollector metricsCollector;

    public SecurityAuditService(ApplicationEventPublisher eventPublisher,
                               SecurityMetricsCollector metricsCollector) {
        this.eventPublisher = eventPublisher;
        this.metricsCollector = metricsCollector;
    }

    @EventListener
    public void handleModerationEvent(ModerationEvent event) {
        if (event.isFlagged()) {
            SecurityIncident incident = SecurityIncident.builder()
                .type(SecurityIncidentType.CONTENT_MODERATION)
                .userId(event.getUserId())
                .content(maskSensitiveContent(event.getContent()))
                .categories(event.getCategories())
                .severity(calculateSeverity(event.getScores()))
                .timestamp(Instant.now())
                .build();
            
            // 메트릭 업데이트
            metricsCollector.recordSecurityIncident(incident);
            
            // 심각한 경우 즉시 알림
            if (incident.getSeverity() == Severity.HIGH) {
                eventPublisher.publishEvent(new SecurityAlertEvent(incident));
            }
        }
    }
    
    @EventListener
    public void handlePIIDetectionEvent(PIIDetectionEvent event) {
        SecurityIncident incident = SecurityIncident.builder()
            .type(SecurityIncidentType.PII_DETECTION)
            .userId(event.getUserId())
            .piiTypes(event.getDetectedTypes())
            .severity(Severity.MEDIUM)
            .timestamp(Instant.now())
            .build();
        
        metricsCollector.recordSecurityIncident(incident);
        eventPublisher.publishEvent(new SecurityAlertEvent(incident));
    }
    
    private String maskSensitiveContent(String content) {
        // 민감한 내용 마스킹
        return content.length() > 100 ? 
            content.substring(0, 100) + "..." : content;
    }
    
    private Severity calculateSeverity(Map<String, Double> scores) {
        double maxScore = scores.values().stream()
            .mapToDouble(Double::doubleValue)
            .max()
            .orElse(0.0);
        
        if (maxScore > 0.9) return Severity.HIGH;
        if (maxScore > 0.7) return Severity.MEDIUM;
        return Severity.LOW;
    }
}
```

## 규정 준수 및 데이터 거버넌스

### GDPR 및 데이터 프라이버시 준수

```java
@Service
public class DataPrivacyService {

    private final DataRetentionPolicy retentionPolicy;
    private final ConsentManagementService consentService;

    public DataPrivacyService(DataRetentionPolicy retentionPolicy,
                             ConsentManagementService consentService) {
        this.retentionPolicy = retentionPolicy;
        this.consentService = consentService;
    }

    public String processWithPrivacyCompliance(String userInput, String userId, 
                                             String region) {
        // 1. 사용자 동의 확인
        if (!consentService.hasValidConsent(userId, ConsentType.AI_PROCESSING)) {
            return "AI 처리에 대한 동의가 필요합니다.";
        }
        
        // 2. 지역별 규정 확인
        DataProcessingRules rules = getDataProcessingRules(region);
        if (!rules.allowsAIProcessing()) {
            return "현재 지역에서는 AI 처리가 제한됩니다.";
        }
        
        // 3. 데이터 최소화 원칙 적용
        String minimizedInput = applyDataMinimization(userInput);
        
        // 4. AI 처리
        String response = processAIRequest(minimizedInput);
        
        // 5. 데이터 보존 정책 적용
        scheduleDataDeletion(userId, minimizedInput, response);
        
        return response;
    }
    
    private String applyDataMinimization(String input) {
        // 불필요한 개인정보 제거
        return input.replaceAll("\\b\\d{3}-\\d{2}-\\d{4}\\b", "[REDACTED]")
                   .replaceAll("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b", "[EMAIL]");
    }
    
    private void scheduleDataDeletion(String userId, String input, String response) {
        // 데이터 보존 기간에 따른 삭제 스케줄링
        Duration retentionPeriod = retentionPolicy.getRetentionPeriod(userId);
        // 스케줄링 로직 구현
    }
}
```

## 실제 사용 사례: 금융 서비스 AI 어시스턴트

```java
@Service
public class FinancialAIAssistantService {

    private final ChatClient chatClient;
    private final ModerationModel moderationModel;
    private final PIIProtectionService piiProtection;
    private final CostControlService costControl;
    private final AuditService auditService;

    public FinancialAIAssistantService(ChatClient chatClient,
                                     ModerationModel moderationModel,
                                     PIIProtectionService piiProtection,
                                     CostControlService costControl,
                                     AuditService auditService) {
        this.chatClient = chatClient;
        this.moderationModel = moderationModel;
        this.piiProtection = piiProtection;
        this.costControl = costControl;
        this.auditService = auditService;
    }

    @PreAuthorize("hasRole('CUSTOMER')")
    public String processFinancialQuery(String query, String customerId) {
        try {
            // 1. 입력 검증 및 보안 검사
            if (piiProtection.containsPII(query)) {
                auditService.logSecurityEvent(SecurityEventType.PII_DETECTED, customerId);
                return "개인정보가 포함된 질문은 처리할 수 없습니다. 일반적인 금융 질문을 해주세요.";
            }
            
            // 2. 콘텐츠 모더레이션
            ModerationResponse moderation = moderationModel.call(new ModerationPrompt(query));
            if (moderation.getResult().getOutput().getResults().get(0).isFlagged()) {
                auditService.logSecurityEvent(SecurityEventType.INAPPROPRIATE_CONTENT, customerId);
                return "부적절한 내용이 감지되었습니다.";
            }
            
            // 3. 비용 제한 확인
            String costCheckResult = costControl.checkCostLimit(customerId, query);
            if (costCheckResult != null) {
                return costCheckResult;
            }
            
            // 4. 금융 전문 AI 처리
            String response = chatClient.prompt()
                .system("""
                    당신은 금융 정보 전문가입니다. 다음 규칙을 준수하세요:
                    1. 개인적인 투자 조언은 제공하지 마세요
                    2. 일반적인 금융 정보만 제공하세요
                    3. 법적 조언은 제공하지 마세요
                    4. 불확실한 정보에 대해서는 명확히 언급하세요
                    5. 항상 전문가와 상담을 권유하세요
                    """)
                .user(query)
                .call()
                .content();
            
            // 5. 감사 로그 기록
            auditService.logAIInteraction(AuditLog.builder()
                .customerId(customerId)
                .query(piiProtection.maskPII(query))
                .responseLength(response.length())
                .timestamp(Instant.now())
                .compliant(true)
                .build());
            
            return response + "\n\n⚠️ 이 정보는 일반적인 참고용이며, 개인적인 금융 결정은 전문가와 상담하시기 바랍니다.";
            
        } catch (Exception e) {
            auditService.logError(customerId, e);
            return "죄송합니다. 현재 서비스를 이용할 수 없습니다.";
        }
    }
}
```

## 결론

이 장에서는 Spring AI 1.0.0 GA의 최신 보안 기능과 비용 관리 도구를 활용하여 안전하고 경제적인 AI 애플리케이션을 구축하는 방법을 살펴보았습니다. 

주요 내용 요약:

### 보안 기능
1. **Cloud Bindings를 통한 안전한 API 키 관리**
2. **OpenAI Moderation API를 활용한 콘텐츠 안전성 검사**
3. **SafeGuard Advisor를 통한 프롬프트 및 응답 모니터링**
4. **PII 탐지 및 자동 마스킹**
5. **지역별 데이터 주권 지원**

### 비용 관리
1. **향상된 Usage API를 통한 상세한 토큰 추적**
2. **실시간 비용 모니터링 및 제한**
3. **의미적 유사성 기반 지능형 캐싱**
4. **모델 티어링을 통한 비용 최적화**
5. **예측적 비용 분석**

### 규정 준수
1. **GDPR 및 데이터 프라이버시 준수**
2. **감사 로깅 및 보안 이벤트 추적**
3. **데이터 최소화 및 보존 정책**

이러한 기능들을 통해 기업급 AI 애플리케이션에서 요구되는 보안, 비용 효율성, 규정 준수를 모두 달성할 수 있습니다. 다음 장에서는 AI 애플리케이션의 관측성과 로깅에 대해 알아보겠습니다.