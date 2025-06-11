# Observability와 로깅

## 개요

AI 애플리케이션은 복잡한 특성으로 인해 전통적인 소프트웨어보다 관측성(Observability)과 로깅이 더욱 중요합니다. Spring AI 1.0.0 GA는 Spring 생태계의 observability 기능을 기반으로 AI 작업에 대한 포괄적인 인사이트를 제공합니다. 이 장에서는 Spring AI의 내장된 observability 기능과 GenAI 호환성 표준을 활용하여 AI 애플리케이션을 효과적으로 모니터링하고 디버깅하는 방법을 살펴보겠습니다.

## Spring AI Observability 개요

Spring AI 1.0.0 GA는 다음 핵심 컴포넌트에 대한 관측성을 제공합니다:

- **ChatClient** (Advisor 포함)
- **ChatModel**, **EmbeddingModel**, **ImageModel**
- **VectorStore**
- **Tool Calling**

### GenAI 호환성 표준

Spring AI는 OpenTelemetry의 GenAI semantic conventions를 따라 일관된 관측성 데이터를 제공합니다:

```yaml
# 주요 GenAI 표준 속성
gen_ai.operation.name: "chat_completions"
gen_ai.system: "openai"
gen_ai.request.model: "gpt-4"
gen_ai.response.model: "gpt-4-0613"
gen_ai.usage.input_tokens: 150
gen_ai.usage.output_tokens: 75
gen_ai.usage.total_tokens: 225
```

### 관측성 구성 요소

1. **메트릭(Metrics)**: 토큰 사용량, 응답 시간, 오류율 등
2. **추적(Traces)**: AI 요청의 전체 실행 경로
3. **로깅(Logging)**: 프롬프트, 응답, 오류 등의 구조화된 로그
4. **메타데이터(Metadata)**: 사용량, 요금 제한 등의 AI 제공업체별 정보

## Spring AI Observability 설정

### 기본 의존성 설정

Spring AI 관측성 기능을 활용하기 위한 의존성을 추가합니다:

```xml
<!-- Maven -->
<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
</dependencies>
```

```kotlin
// Gradle
dependencies {
    implementation("org.springframework.ai:spring-ai-starter-model-openai")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
}
```

### Observability 구성

`application.yml`에서 Spring AI 관측성 기능을 구성합니다:

```yaml
spring:
  ai:
    # ChatClient 관측성 설정
    chat:
      client:
        observations:
          log-prompt: false  # 프롬프트 로깅 (민감 정보 주의)
      observations:
        log-prompt: false     # 모델 프롬프트 로깅
        log-completion: false # 응답 로깅
        include-error-logging: true
    
    # 이미지 모델 관측성
    image:
      observations:
        log-prompt: false
    
    # 벡터 스토어 관측성
    vectorstore:
      observations:
        log-query-response: false
    
    # 도구 호출 관측성
    tools:
      observations:
        include-content: false  # 도구 인수/결과 로깅

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,traces
  endpoint:
    health:
      show-details: always
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true
  tracing:
    sampling:
      probability: 1.0  # 개발 환경에서는 모든 트레이스 샘플링
```

## ChatClient Observability

### ChatClient 관측성 개요

ChatClient의 `call()` 및 `stream()` 작업에 대한 관측성 데이터가 자동으로 수집됩니다:

#### Low Cardinality Keys (메트릭용)
```yaml
gen_ai.operation.name: "framework"
gen_ai.system: "spring_ai"
spring.ai.chat.client.stream: "false"  # 스트리밍 여부
spring.ai.kind: "chat_client"
```

#### High Cardinality Keys (트레이스용)
```yaml
gen_ai.prompt: "사용자 프롬프트 내용"  # 선택적
spring.ai.chat.client.advisors: ["advisor1", "advisor2"]
spring.ai.chat.client.conversation.id: "conv-123"
spring.ai.chat.client.tool.names: ["weather_tool", "calculator_tool"]
```

### ChatClient 관측성 활용 예제

```java
@Service
public class ObservableChatService {

    private final ChatClient chatClient;
    private final MeterRegistry meterRegistry;
    
    public ObservableChatService(ChatClient.Builder builder, MeterRegistry meterRegistry) {
        this.chatClient = builder.build();
        this.meterRegistry = meterRegistry;
    }
    
    public String processQuery(String userInput) {
        // ChatClient는 자동으로 관측성 데이터를 생성
        ChatResponse response = chatClient.prompt()
            .user(userInput)
            .call()
            .chatResponse();
        
        // 사용량 메타데이터 추출
        Usage usage = response.getMetadata().getUsage();
        
        // 커스텀 메트릭 추가
        meterRegistry.counter("chat.requests.processed").increment();
        meterRegistry.summary("chat.tokens.total").record(usage.getTotalTokens());
        
        return response.getResult().getOutput().getContent();
    }
}
```

### AI 서비스 건강 지표

Spring AI 서비스의 건강 상태를 모니터링하는 커스텀 건강 지표:

```java
@Component
public class AiServiceHealthIndicator implements HealthIndicator {

    private final ChatClient chatClient;
    
    public AiServiceHealthIndicator(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    @Override
    public Health health() {
        try {
            Timer.Sample sample = Timer.start();
            
            String response = chatClient.prompt()
                .user("health check")
                .call()
                .content();
            
            long responseTime = sample.stop(Timer.builder("ai.health.check")
                .register(Metrics.globalRegistry))
                .toMillis();
            
            return Health.up()
                .withDetail("status", "AI service available")
                .withDetail("responseTime", responseTime + "ms")
                .withDetail("provider", "openai")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("status", "AI service unavailable")
                .withDetail("error", e.getMessage())
                .withDetail("errorType", e.getClass().getSimpleName())
                .build();
        }
    }
}
```

## ChatModel Observability

### ChatModel 관측성 표준

ChatModel 호출에 대한 GenAI 표준 메트릭과 트레이스가 자동으로 수집됩니다:

#### 표준 메트릭
- `gen_ai.client.token.usage`: 토큰 사용량 측정
- `gen_ai.client.operation`: 작업 실행 시간

#### 관측성 키
```yaml
# Low Cardinality (메트릭용)
gen_ai.operation.name: "chat_completions"
gen_ai.system: "openai"
gen_ai.request.model: "gpt-4"
gen_ai.response.model: "gpt-4-0613"

# High Cardinality (트레이스용)
gen_ai.request.frequency_penalty: 0.0
gen_ai.request.max_tokens: 1000
gen_ai.request.temperature: 0.7
gen_ai.response.finish_reasons: ["stop"]
gen_ai.usage.input_tokens: 150
gen_ai.usage.output_tokens: 75
gen_ai.usage.total_tokens: 225
gen_ai.prompt: "사용자 프롬프트"  # 선택적
gen_ai.completion: "AI 응답"      # 선택적
```

### ChatModel 사용량 추적

```java
@Service
public class ChatModelUsageTracker {

    private final ChatModel chatModel;
    private final MeterRegistry meterRegistry;
    
    public ChatModelUsageTracker(ChatModel chatModel, MeterRegistry meterRegistry) {
        this.chatModel = chatModel;
        this.meterRegistry = meterRegistry;
    }
    
    public String generateResponse(String prompt) {
        ChatResponse response = chatModel.call(new Prompt(prompt));
        
        // Usage 메타데이터 추출
        Usage usage = response.getMetadata().getUsage();
        
        // 상세 사용량 추적 (OpenAI 네이티브 사용량)
        if (usage.getNativeUsage() instanceof org.springframework.ai.openai.api.OpenAiApi.Usage nativeUsage) {
            // 캐시된 토큰 (비용 절감)
            int cachedTokens = nativeUsage.promptTokensDetails().cachedTokens();
            meterRegistry.counter("ai.tokens.cached").increment(cachedTokens);
            
            // 추론 토큰 (o1 모델용)
            int reasoningTokens = nativeUsage.completionTokenDetails().reasoningTokens();
            meterRegistry.counter("ai.tokens.reasoning").increment(reasoningTokens);
            
            // 오디오 토큰
            int audioTokens = nativeUsage.promptTokensDetails().audioTokens() + 
                            nativeUsage.completionTokenDetails().audioTokens();
            meterRegistry.counter("ai.tokens.audio").increment(audioTokens);
        }
        
        // 비용 계산 및 추적
        double estimatedCost = calculateCost(usage);
        meterRegistry.summary("ai.cost.estimated").record(estimatedCost);
        
        return response.getResult().getOutput().getContent();
    }
    
    private double calculateCost(Usage usage) {
        // GPT-4 가격 기준 (예시)
        double inputCost = usage.getPromptTokens() * 0.03 / 1000;
        double outputCost = usage.getCompletionTokens() * 0.06 / 1000;
        return inputCost + outputCost;
    }
}
```

## 커스텀 메트릭 수집

### AI 특화 메트릭 정의

Spring AI의 자동 관측성 기능과 함께 비즈니스 특화 메트릭을 추가할 수 있습니다:

```java
@Component
public class AIMetricsCollector {

    private final MeterRegistry meterRegistry;
    
    // 비즈니스 메트릭
    private final Counter conversationCounter;
    private final Counter qualityFeedbackCounter;
    private final Timer responseQualityTimer;
    private final DistributionSummary promptLengthSummary;
    
    public AIMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.conversationCounter = Counter.builder("ai.conversations")
                .description("Number of AI conversations")
                .register(meterRegistry);
        
        this.qualityFeedbackCounter = Counter.builder("ai.feedback")
                .description("User feedback on AI responses")
                .register(meterRegistry);
        
        this.responseQualityTimer = Timer.builder("ai.quality.evaluation")
                .description("Time spent evaluating response quality")
                .register(meterRegistry);
        
        this.promptLengthSummary = DistributionSummary.builder("ai.prompt.length")
                .description("Length of user prompts")
                .baseUnit("characters")
                .register(meterRegistry);
    }
    
    // 대화 시작 추적
    public void trackConversationStart(String userId, String conversationType) {
        conversationCounter.increment(
            Tags.of(
                Tag.of("user_id", userId),
                Tag.of("type", conversationType)
            )
        );
    }
    
    // 사용자 피드백 추적
    public void trackUserFeedback(String responseId, boolean helpful, String reason) {
        qualityFeedbackCounter.increment(
            Tags.of(
                Tag.of("response_id", responseId),
                Tag.of("helpful", String.valueOf(helpful)),
                Tag.of("reason", reason != null ? reason : "none")
            )
        );
    }
    
    // 프롬프트 길이 추적
    public void trackPromptLength(String prompt) {
        promptLengthSummary.record(prompt.length());
    }
    
    // 응답 품질 평가 시간 측정
    public Timer.Sample startQualityEvaluation() {
        return Timer.start(meterRegistry);
    }
    
    public void stopQualityEvaluation(Timer.Sample sample, double qualityScore) {
        sample.stop(responseQualityTimer);
        
        // 품질 점수도 기록
        meterRegistry.gauge("ai.response.quality.score", qualityScore);
    }
    
    // 모델별 성능 추적
    public void trackModelPerformance(String modelName, long responseTimeMs, int tokenCount) {
        Tags tags = Tags.of(
            Tag.of("model", modelName)
        );
        
        meterRegistry.timer("ai.model.response.time", tags)
                .record(responseTimeMs, TimeUnit.MILLISECONDS);
        
        meterRegistry.summary("ai.model.token.efficiency", tags)
                .record((double) tokenCount / responseTimeMs * 1000); // tokens per second
    }
}
```

### 멀티모델 관측성

여러 AI 모델을 사용하는 환경에서 모델별 성능을 비교하고 최적화할 수 있습니다:

```java
@Service
public class MultiModelObservabilityService {

    private final List<ChatModel> chatModels;
    private final MeterRegistry meterRegistry;
    
    public MultiModelObservabilityService(List<ChatModel> chatModels, MeterRegistry meterRegistry) {
        this.chatModels = chatModels;
        this.meterRegistry = meterRegistry;
    }
    
    public String processWithBestModel(String prompt) {
        String bestResponse = null;
        double bestScore = 0.0;
        
        for (ChatModel model : chatModels) {
            String modelName = getModelName(model);
            Timer.Sample sample = Timer.start(meterRegistry);
            
            try {
                ChatResponse response = model.call(new Prompt(prompt));
                String content = response.getResult().getOutput().getContent();
                
                // 응답 시간 측정
                sample.stop(meterRegistry.timer("ai.model.response.time", "model", modelName));
                
                // 토큰 효율성 계산
                Usage usage = response.getMetadata().getUsage();
                double efficiency = (double) usage.getCompletionTokens() / usage.getPromptTokens();
                meterRegistry.gauge("ai.model.efficiency", Tags.of("model", modelName), efficiency);
                
                // 간단한 품질 평가 (실제로는 더 정교한 평가 필요)
                double qualityScore = evaluateQuality(content, prompt);
                meterRegistry.gauge("ai.model.quality", Tags.of("model", modelName), qualityScore);
                
                if (qualityScore > bestScore) {
                    bestScore = qualityScore;
                    bestResponse = content;
                }
                
            } catch (Exception e) {
                meterRegistry.counter("ai.model.errors", "model", modelName, "error", e.getClass().getSimpleName()).increment();
            }
        }
        
        return bestResponse;
    }
    
    private String getModelName(ChatModel model) {
        return model.getClass().getSimpleName();
    }
    
    private double evaluateQuality(String response, String prompt) {
        // 간단한 품질 평가 로직
        if (response.length() < 10) return 0.1;
        if (response.toLowerCase().contains("error") || response.toLowerCase().contains("sorry")) return 0.3;
        return Math.min(1.0, response.length() / 100.0); // 길이 기반 간단 평가
    }
}
```

## VectorStore Observability

### VectorStore 관측성 표준

VectorStore 작업에 대한 관측성 데이터가 자동으로 수집됩니다:

#### 관측성 키
```yaml
# Low Cardinality
db.operation.name: "query"  # add, delete, query
db.system: "pg_vector"      # 벡터 DB 유형
spring.ai.kind: "vector_store"

# High Cardinality
db.collection.name: "documents"
db.namespace: "knowledge_base"
db.vector.dimension_count: 1536
db.vector.query.top_k: 5
db.vector.query.similarity_threshold: 0.7
db.vector.query.content: "사용자 쿼리 내용"  # 선택적
db.vector.query.response.documents: "검색된 문서들"  # 선택적
```

### RAG 시스템 관측성

```java
@Service
public class ObservableRAGService {

    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    private final MeterRegistry meterRegistry;
    
    public ObservableRAGService(VectorStore vectorStore, ChatClient chatClient, MeterRegistry meterRegistry) {
        this.vectorStore = vectorStore;
        this.chatClient = chatClient;
        this.meterRegistry = meterRegistry;
    }
    
    public String processRAGQuery(String query) {
        Timer.Sample ragTimer = Timer.start(meterRegistry);
        
        try {
            // 1. 벡터 검색 (자동 관측성)
            List<Document> relevantDocs = vectorStore.similaritySearch(
                SearchRequest.query(query).withTopK(5).withSimilarityThreshold(0.7)
            );
            
            // 검색 결과 메트릭
            meterRegistry.summary("rag.retrieval.documents.count").record(relevantDocs.size());
            
            if (!relevantDocs.isEmpty()) {
                double avgSimilarity = relevantDocs.stream()
                    .mapToDouble(doc -> (Double) doc.getMetadata().getOrDefault("similarity", 0.0))
                    .average().orElse(0.0);
                meterRegistry.gauge("rag.retrieval.similarity.avg", avgSimilarity);
            }
            
            // 2. 컨텍스트 구성
            String context = relevantDocs.stream()
                .map(Document::getContent)
                .collect(Collectors.joining("\n\n"));
            
            // 3. ChatClient 호출 (자동 관측성)
            String response = chatClient.prompt()
                .system("다음 컨텍스트를 바탕으로 답변하세요: " + context)
                .user(query)
                .call()
                .content();
            
            // RAG 전체 시간 측정
            ragTimer.stop(meterRegistry.timer("rag.total.time"));
            
            // 성공 카운터
            meterRegistry.counter("rag.requests.success").increment();
            
            return response;
            
        } catch (Exception e) {
            ragTimer.stop(meterRegistry.timer("rag.total.time", "status", "error"));
            meterRegistry.counter("rag.requests.error", "error", e.getClass().getSimpleName()).increment();
            throw e;
        }
    }
}
```

## Tool Calling Observability

### Tool 호출 관측성

Tool 호출에 대한 자동 관측성:

```yaml
# Low Cardinality
gen_ai.operation.name: "framework"
gen_ai.system: "spring_ai"
spring.ai.kind: "tool_call"
spring.ai.tool.definition.name: "weather_tool"

# High Cardinality
spring.ai.tool.definition.description: "Get current weather"
spring.ai.tool.definition.schema: "{...}"
spring.ai.tool.call.arguments: "{\"location\":\"Seoul\"}"  # 선택적
spring.ai.tool.call.result: "{\"temperature\":25}"        # 선택적
```

### Tool 성능 모니터링

```java
@Component
public class ObservableWeatherTool implements Function<WeatherRequest, WeatherResponse> {

    private final WeatherService weatherService;
    private final MeterRegistry meterRegistry;
    
    public ObservableWeatherTool(WeatherService weatherService, MeterRegistry meterRegistry) {
        this.weatherService = weatherService;
        this.meterRegistry = meterRegistry;
    }
    
    @Override
    public WeatherResponse apply(WeatherRequest request) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            WeatherResponse response = weatherService.getCurrentWeather(request.location());
            
            // 성공 메트릭
            sample.stop(meterRegistry.timer("tool.weather.execution.time", "status", "success"));
            meterRegistry.counter("tool.weather.calls", "status", "success").increment();
            
            return response;
            
        } catch (Exception e) {
            // 실패 메트릭
            sample.stop(meterRegistry.timer("tool.weather.execution.time", "status", "error"));
            meterRegistry.counter("tool.weather.calls", "status", "error", "error", e.getClass().getSimpleName()).increment();
            
            throw e;
        }
    }
    
    record WeatherRequest(String location) {}
    record WeatherResponse(String location, double temperature, String condition) {}
}
```

## 구조화된 로깅 전략

### Spring AI 로깅 구성

Spring AI 1.0.0은 트레이싱 인식 로깅을 지원합니다:

```yaml
# application.yml - 로깅 구성
logging:
  level:
    org.springframework.ai: INFO
    # 상세 디버깅이 필요한 경우
    org.springframework.ai.chat: DEBUG
    org.springframework.ai.vectorstore: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level [%X{traceId:-},%X{spanId:-}] %logger{36} - %msg%n"

spring:
  ai:
    # 프롬프트 및 응답 로깅 (주의: 민감 정보 포함 가능)
    chat:
      observations:
        log-prompt: false
        log-completion: false
        include-error-logging: true
```

### 구조화된 JSON 로깅

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="!local">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeContext>true</includeContext>
                <includeMdc>true</includeMdc>
                <customFields>{"service":"ai-application"}</customFields>
                <fieldNames>
                    <timestamp>@timestamp</timestamp>
                    <message>message</message>
                    <level>level</level>
                    <thread>thread</thread>
                    <logger>logger</logger>
                </fieldNames>
            </encoder>
        </appender>
    </springProfile>
    
    <springProfile name="local">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level [%X{traceId:-},%X{spanId:-}] %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </springProfile>
    
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

### 민감 정보 마스킹

```java
@Component
public class AIContentMasker {

    private static final List<Pattern> SENSITIVE_PATTERNS = List.of(
        Pattern.compile("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b"), // 이메일
        Pattern.compile("\\b\\d{3}-\\d{2}-\\d{4}\\b"),                              // SSN
        Pattern.compile("\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"),     // 신용카드
        Pattern.compile("\\b\\d{3}-\\d{3}-\\d{4}\\b")                              // 전화번호
    );
    
    public String maskSensitiveContent(String content) {
        if (content == null || content.isEmpty()) {
            return content;
        }
        
        String masked = content;
        
        for (Pattern pattern : SENSITIVE_PATTERNS) {
            masked = pattern.matcher(masked).replaceAll("[MASKED]");
        }
        
        return masked;
    }
    
    public String maskPromptForLogging(String prompt) {
        // 프롬프트 로깅시 추가 마스킹
        String masked = maskSensitiveContent(prompt);
        
        // 길이 제한 (로그 크기 제어)
        if (masked.length() > 1000) {
            masked = masked.substring(0, 1000) + "... [TRUNCATED]";
        }
        
        return masked;
    }
}
```

### AI 상호작용 추적 로깅

Spring AI의 내장 관측성과 함께 비즈니스 로직을 추가한 포괄적인 로깅 서비스:

```java
@Service
public class AIInteractionLogger {

    private static final Logger logger = LoggerFactory.getLogger(AIInteractionLogger.class);
    private final AIContentMasker contentMasker;
    private final MeterRegistry meterRegistry;
    
    public AIInteractionLogger(AIContentMasker contentMasker, MeterRegistry meterRegistry) {
        this.contentMasker = contentMasker;
        this.meterRegistry = meterRegistry;
    }
    
    public String processWithLogging(String userInput, String userId, String sessionId) {
        String requestId = UUID.randomUUID().toString();
        
        // MDC에 컨텍스트 정보 설정
        try (MDCCloseable mdcCloseable = MDC.putCloseable("requestId", requestId)) {
            MDC.put("userId", userId);
            MDC.put("sessionId", sessionId);
            
            Timer.Sample sample = Timer.start(meterRegistry);
            
            try {
                // 요청 로깅
                logger.info("AI request received", 
                    kv("prompt_length", userInput.length()),
                    kv("masked_prompt", contentMasker.maskPromptForLogging(userInput)));
                
                // ChatClient 호출 (자동 관측성 수집)
                ChatResponse response = chatClient.prompt()
                    .user(userInput)
                    .call()
                    .chatResponse();
                
                String responseContent = response.getResult().getOutput().getContent();
                Usage usage = response.getMetadata().getUsage();
                
                // 응답 로깅
                logger.info("AI response generated",
                    kv("response_length", responseContent.length()),
                    kv("prompt_tokens", usage.getPromptTokens()),
                    kv("completion_tokens", usage.getCompletionTokens()),
                    kv("total_tokens", usage.getTotalTokens()),
                    kv("masked_response", contentMasker.maskSensitiveContent(
                        responseContent.substring(0, Math.min(200, responseContent.length()))
                    )));
                
                // 시간 측정
                sample.stop(meterRegistry.timer("ai.interaction.duration"));
                
                // 성공 카운터
                meterRegistry.counter("ai.interactions", "status", "success").increment();
                
                return responseContent;
                
            } catch (Exception e) {
                // 오류 로깅
                logger.error("AI request failed",
                    kv("error_type", e.getClass().getSimpleName()),
                    kv("error_message", e.getMessage()));
                
                sample.stop(meterRegistry.timer("ai.interaction.duration", "status", "error"));
                meterRegistry.counter("ai.interactions", "status", "error", "error_type", e.getClass().getSimpleName()).increment();
                
                throw e;
            } finally {
                MDC.remove("userId");
                MDC.remove("sessionId");
            }
        }
    }
    
    // Structured logging helper
    private static KeyValue kv(String key, Object value) {
        return KeyValue.of(key, value);
    }
}
```

## 분산 추적 구현

### Spring AI 자동 추적

Spring AI 1.0.0은 자동으로 OpenTelemetry 호환 추적을 제공합니다:

```yaml
# application.yml - 추적 구성
management:
  tracing:
    sampling:
      probability: 1.0  # 모든 요청 추적
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces

spring:
  ai:
    # AI 작업에 대한 상세 추적 정보 포함
    chat:
      observations:
        log-prompt: false    # 프롬프트를 스팬에 포함
        log-completion: false # 응답을 스팬에 포함
```

### 커스텀 스팬 추가

```java
@Service
public class TracedAIWorkflowService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final Tracer tracer;
    
    public TracedAIWorkflowService(ChatClient chatClient, VectorStore vectorStore, Tracer tracer) {
        this.chatClient = chatClient;
        this.vectorStore = vectorStore;
        this.tracer = tracer;
    }
    
    public String processComplexWorkflow(String userQuery) {
        // 전체 워크플로우 스팬
        Span workflowSpan = tracer.nextSpan().name("ai-complex-workflow");
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(workflowSpan.start())) {
            workflowSpan.tag("workflow.type", "rag-with-analysis");
            workflowSpan.tag("user.query.length", String.valueOf(userQuery.length()));
            
            // 1. 질문 분석 스팬
            String intent = analyzeUserIntent(userQuery);
            
            // 2. 컨텍스트 검색 스팬 (자동 추적)
            List<Document> context = vectorStore.similaritySearch(
                SearchRequest.query(userQuery).withTopK(5)
            );
            
            // 3. 응답 생성 스팬 (자동 추적)
            String response = chatClient.prompt()
                .system("컨텍스트: " + formatContext(context))
                .user(userQuery)
                .call()
                .content();
            
            // 4. 후처리 스팬
            String finalResponse = postProcessResponse(response, intent);
            
            workflowSpan.tag("workflow.status", "success");
            workflowSpan.tag("response.length", String.valueOf(finalResponse.length()));
            
            return finalResponse;
            
        } catch (Exception e) {
            workflowSpan.tag("error", e.getClass().getSimpleName());
            throw e;
        } finally {
            workflowSpan.end();
        }
    }
    
    private String analyzeUserIntent(String query) {
        Span intentSpan = tracer.nextSpan().name("analyze-user-intent");
        
        try (Tracer.SpanInScope is = tracer.withSpanInScope(intentSpan.start())) {
            intentSpan.tag("query.language", detectLanguage(query));
            
            // 의도 분석 로직
            String intent = performIntentAnalysis(query);
            
            intentSpan.tag("detected.intent", intent);
            return intent;
            
        } finally {
            intentSpan.end();
        }
    }
    
    private String postProcessResponse(String response, String intent) {
        Span postProcessSpan = tracer.nextSpan().name("post-process-response");
        
        try (Tracer.SpanInScope ps = tracer.withSpanInScope(postProcessSpan.start())) {
            postProcessSpan.tag("intent", intent);
            postProcessSpan.tag("original.length", String.valueOf(response.length()));
            
            // 의도에 따른 후처리
            String processed = switch (intent) {
                case "question" -> addSourceReferences(response);
                case "summarize" -> formatSummary(response);
                case "analyze" -> addAnalysisStructure(response);
                default -> response;
            };
            
            postProcessSpan.tag("processed.length", String.valueOf(processed.length()));
            return processed;
            
        } finally {
            postProcessSpan.end();
        }
    }
    
    // Helper methods
    private String detectLanguage(String text) { return "ko"; }
    private String performIntentAnalysis(String query) { return "question"; }
    private String formatContext(List<Document> docs) { return ""; }
    private String addSourceReferences(String response) { return response; }
    private String formatSummary(String response) { return response; }
    private String addAnalysisStructure(String response) { return response; }
}
```

## AI 메타데이터 추적

### Usage 메타데이터 활용

Spring AI 1.0.0의 향상된 Usage API를 활용한 상세 비용 및 사용량 추적:

```java
@Service
public class AIUsageTrackingService {

    private final ChatModel chatModel;
    private final MeterRegistry meterRegistry;
    private final CostCalculator costCalculator;
    
    public AIUsageTrackingService(ChatModel chatModel, MeterRegistry meterRegistry, CostCalculator costCalculator) {
        this.chatModel = chatModel;
        this.meterRegistry = meterRegistry;
        this.costCalculator = costCalculator;
    }
    
    public String processWithDetailedTracking(String prompt) {
        ChatResponse response = chatModel.call(new Prompt(prompt));
        Usage usage = response.getMetadata().getUsage();
        
        // 기본 사용량 추적
        trackBasicUsage(usage);
        
        // 네이티브 사용량 추적 (OpenAI 예시)
        if (usage.getNativeUsage() instanceof org.springframework.ai.openai.api.OpenAiApi.Usage nativeUsage) {
            trackDetailedUsage(nativeUsage);
        }
        
        return response.getResult().getOutput().getContent();
    }
    
    private void trackBasicUsage(Usage usage) {
        // 기본 토큰 메트릭
        meterRegistry.counter("ai.tokens.prompt").increment(usage.getPromptTokens());
        meterRegistry.counter("ai.tokens.completion").increment(usage.getCompletionTokens());
        meterRegistry.counter("ai.tokens.total").increment(usage.getTotalTokens());
        
        // 기본 비용 계산
        double basicCost = costCalculator.calculateBasicCost(usage);
        meterRegistry.summary("ai.cost.basic").record(basicCost);
    }
    
    private void trackDetailedUsage(org.springframework.ai.openai.api.OpenAiApi.Usage nativeUsage) {
        // 캐시된 토큰 (비용 절감)
        int cachedTokens = nativeUsage.promptTokensDetails().cachedTokens();
        if (cachedTokens > 0) {
            meterRegistry.counter("ai.tokens.cached").increment(cachedTokens);
            double cachedSavings = costCalculator.calculateCachedSavings(cachedTokens);
            meterRegistry.summary("ai.cost.savings.cached").record(cachedSavings);
        }
        
        // 추론 토큰 (o1 모델용)
        int reasoningTokens = nativeUsage.completionTokenDetails().reasoningTokens();
        if (reasoningTokens > 0) {
            meterRegistry.counter("ai.tokens.reasoning").increment(reasoningTokens);
            double reasoningCost = costCalculator.calculateReasoningCost(reasoningTokens);
            meterRegistry.summary("ai.cost.reasoning").record(reasoningCost);
        }
        
        // 오디오 토큰
        int audioPromptTokens = nativeUsage.promptTokensDetails().audioTokens();
        int audioCompletionTokens = nativeUsage.completionTokenDetails().audioTokens();
        if (audioPromptTokens > 0 || audioCompletionTokens > 0) {
            meterRegistry.counter("ai.tokens.audio.prompt").increment(audioPromptTokens);
            meterRegistry.counter("ai.tokens.audio.completion").increment(audioCompletionTokens);
            
            double audioCost = costCalculator.calculateAudioCost(audioPromptTokens, audioCompletionTokens);
            meterRegistry.summary("ai.cost.audio").record(audioCost);
        }
        
        // 예측 토큰 (speculative decoding)
        int acceptedTokens = nativeUsage.completionTokenDetails().acceptedPredictionTokens();
        int rejectedTokens = nativeUsage.completionTokenDetails().rejectedPredictionTokens();
        if (acceptedTokens > 0 || rejectedTokens > 0) {
            meterRegistry.counter("ai.tokens.prediction.accepted").increment(acceptedTokens);
            meterRegistry.counter("ai.tokens.prediction.rejected").increment(rejectedTokens);
            
            double predictionEfficiency = (double) acceptedTokens / (acceptedTokens + rejectedTokens);
            meterRegistry.gauge("ai.prediction.efficiency", predictionEfficiency);
        }
        
        // 전체 상세 비용
        double totalDetailedCost = costCalculator.calculateDetailedCost(nativeUsage);
        meterRegistry.summary("ai.cost.detailed.total").record(totalDetailedCost);
    }
}

@Component
public class CostCalculator {
    
    // 2024년 OpenAI 가격 기준
    private static final double GPT4_INPUT_COST_PER_1K = 0.03;
    private static final double GPT4_OUTPUT_COST_PER_1K = 0.06;
    private static final double REASONING_COST_PER_1K = 0.15;  // o1 모델
    private static final double AUDIO_COST_PER_1K = 0.006;
    
    public double calculateBasicCost(Usage usage) {
        return (usage.getPromptTokens() * GPT4_INPUT_COST_PER_1K + 
                usage.getCompletionTokens() * GPT4_OUTPUT_COST_PER_1K) / 1000;
    }
    
    public double calculateDetailedCost(org.springframework.ai.openai.api.OpenAiApi.Usage nativeUsage) {
        double inputCost = (nativeUsage.promptTokens() - nativeUsage.promptTokensDetails().cachedTokens()) 
                          * GPT4_INPUT_COST_PER_1K / 1000;
        double outputCost = nativeUsage.completionTokens() * GPT4_OUTPUT_COST_PER_1K / 1000;
        double reasoningCost = nativeUsage.completionTokenDetails().reasoningTokens() * REASONING_COST_PER_1K / 1000;
        double audioCost = (nativeUsage.promptTokensDetails().audioTokens() + 
                           nativeUsage.completionTokenDetails().audioTokens()) * AUDIO_COST_PER_1K / 1000;
        
        return inputCost + outputCost + reasoningCost + audioCost;
    }
    
    public double calculateCachedSavings(int cachedTokens) {
        return cachedTokens * GPT4_INPUT_COST_PER_1K / 1000;
    }
    
    public double calculateReasoningCost(int reasoningTokens) {
        return reasoningTokens * REASONING_COST_PER_1K / 1000;
    }
    
    public double calculateAudioCost(int promptAudioTokens, int completionAudioTokens) {
        return (promptAudioTokens + completionAudioTokens) * AUDIO_COST_PER_1K / 1000;
    }
}
```

### Rate Limit 모니터링

AI 서비스 호출을 추적하는 방법을 살펴보겠습니다:

```java
import brave.Span;
import brave.Tracer;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

@Service
public class TracedAiService {

    private final ChatClient chatClient;
    private final Tracer tracer;
    
    public TracedAiService(ChatClient chatClient, Tracer tracer) {
        this.chatClient = chatClient;
        this.tracer = tracer;
    }
    
    public String generateResponse(Prompt prompt) {
        // 'ai-request' 이름으로 새 스팬 생성
        Span aiRequestSpan = tracer.nextSpan().name("ai-request");
        
        try (Tracer.SpanInScope spanInScope = tracer.withSpanInScope(aiRequestSpan.start())) {
            // 프롬프트 정보 추가
            aiRequestSpan.tag("prompt.messages.count", String.valueOf(prompt.getMessages().size()));
            aiRequestSpan.tag("prompt.type", prompt.getClass().getSimpleName());
            
            // AI 호출
            long startTime = System.currentTimeMillis();
            var response = chatClient.call(prompt);
            long duration = System.currentTimeMillis() - startTime;
            
            // 응답 정보 추가
            aiRequestSpan.tag("response.time", String.valueOf(duration));
            
            // 토큰 사용량 정보 추가 (가능한 경우)
            var usage = response.getMetadata().get("usage");
            if (usage instanceof Map) {
                Map<String, Object> usageMap = (Map<String, Object>) usage;
                aiRequestSpan.tag("tokens.total", String.valueOf(usageMap.getOrDefault("total_tokens", 0)));
                aiRequestSpan.tag("tokens.prompt", String.valueOf(usageMap.getOrDefault("prompt_tokens", 0)));
                aiRequestSpan.tag("tokens.completion", String.valueOf(usageMap.getOrDefault("completion_tokens", 0)));
            }
            
            return response.getResult().getOutput().getContent();
        } catch (Exception e) {
            // 오류 정보 추가
            aiRequestSpan.tag("error", e.getClass().getSimpleName());
            aiRequestSpan.tag("error.message", e.getMessage());
            throw e;
        } finally {
            // 스팬 종료
            aiRequestSpan.finish();
        }
    }
}
```

### 전체 워크플로우 추적

AI 서비스를 포함한 전체 애플리케이션 워크플로우를 추적하는 방법을 살펴보겠습니다:

```java
import brave.Span;
import brave.Tracer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AiController {

    private final TracedAiService aiService;
    private final DocumentService documentService;
    private final Tracer tracer;
    
    @Autowired
    public AiController(TracedAiService aiService, DocumentService documentService, Tracer tracer) {
        this.aiService = aiService;
        this.documentService = documentService;
        this.tracer = tracer;
    }
    
    @PostMapping("/api/summarize")
    public SummaryResponse summarizeDocument(@RequestBody SummaryRequest request) {
        // 전체 요청을 위한 스팬 이미 활성화됨 (Spring Cloud Sleuth가 자동으로 생성)
        Span currentSpan = tracer.currentSpan();
        currentSpan.tag("request.type", "document_summarization");
        
        // 문서 조회
        Span documentSpan = tracer.nextSpan().name("document-retrieval");
        String document;
        try (Tracer.SpanInScope documentScope = tracer.withSpanInScope(documentSpan.start())) {
            documentSpan.tag("document.id", request.getDocumentId());
            document = documentService.getDocumentContent(request.getDocumentId());
            documentSpan.tag("document.length", String.valueOf(document.length()));
        } finally {
            documentSpan.finish();
        }
        
        // 요약 프롬프트 생성
        Prompt summaryPrompt = createSummaryPrompt(document, request.getMaxLength());
        
        // AI 서비스 호출 (내부적으로 자체 스팬 생성)
        String summary = aiService.generateResponse(summaryPrompt);
        
        // 응답 반환
        return new SummaryResponse(summary);
    }
    
    // 요청 및 응답 클래스
    static class SummaryRequest {
        private String documentId;
        private int maxLength;
        
        // getter, setter
        public String getDocumentId() {
            return documentId;
        }
        
        public int getMaxLength() {
            return maxLength;
        }
    }
    
    static class SummaryResponse {
        private String summary;
        
        public SummaryResponse(String summary) {
            this.summary = summary;
        }
        
        public String getSummary() {
            return summary;
        }
    }
    
    // 헬퍼 메서드
    private Prompt createSummaryPrompt(String document, int maxLength) {
        // 프롬프트 생성 로직
        return new Prompt(/* ... */);
    }
}
```

### Zipkin UI로 추적 시각화

Zipkin을 사용하여 트레이스를 시각화하는 방법:

1. Zipkin 서버를 실행합니다 (Docker 사용):
   ```bash
   docker run -d -p 9411:9411 openzipkin/zipkin
   ```

2. Spring Boot 애플리케이션을 실행합니다.

3. 웹 브라우저에서 Zipkin UI에 접속합니다: `http://localhost:9411`

4. 애플리케이션 이름, 서비스 이름, 또는 특정 태그로 트레이스를 검색하고 시각화합니다.

## 대시보드 및 알림 설정

수집된 메트릭과 로그를 시각화하고 모니터링하기 위한 대시보드와 알림 시스템을 구성하는 방법을 알아보겠습니다.

### Prometheus 및 Grafana 설정

1. Docker Compose를 사용하여 Prometheus와 Grafana를 설정합니다:

```yaml
# docker-compose.yml
version: '3'
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    
  grafana:
    image: grafana/grafana
    depends_on:
      - prometheus
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
```

2. Prometheus 설정 파일을 작성합니다:

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-ai-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']
```

3. Docker Compose를 시작합니다:

```bash
docker-compose up -d
```

4. Grafana(http://localhost:3000)에 접속하여 다음 단계를 수행합니다:
   - Prometheus 데이터 소스 추가: http://prometheus:9090
   - AI 관련 대시보드 구성

### AI 모니터링 대시보드

Grafana에서 다음 메트릭을 시각화하는 대시보드를 구성할 수 있습니다:

1. **기본 메트릭**:
   - 요청 횟수 및 오류율
   - 응답 시간 분포
   - 시스템 리소스 사용량 (CPU, 메모리)

2. **AI 특화 메트릭**:
   - 토큰 사용량
   - 모델별 응답 시간
   - 비용 추적
   - 응답 품질 점수

다음은 Grafana 대시보드에 사용할 수 있는 PromQL 쿼리 예시입니다:

```
# 분당 AI 요청 수
rate(ai_requests_total[1m])

# 평균 응답 시간
rate(ai_response_time_seconds_sum[5m]) / rate(ai_response_time_seconds_count[5m])

# 95번째 백분위 응답 시간
histogram_quantile(0.95, sum(rate(ai_response_time_seconds_bucket[5m])) by (le))

# 모델별 요청 수
sum(ai_requests_total) by (model)

# 오류율
rate(ai_errors_total[5m]) / rate(ai_requests_total[5m])

# 총 토큰 사용량
sum(increase(ai_token_usage_total[24h]))

# 예상 일일 비용
sum(increase(ai_cost_total[24h]))
```

### 알림 규칙 설정

중요한 메트릭에 대한 알림 규칙을 Prometheus에 설정할 수 있습니다:

```yaml
# prometheus.yml에 추가
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - "alert_rules.yml"
```

```yaml
# alert_rules.yml
groups:
- name: ai-alerts
  rules:
  - alert: HighErrorRate
    expr: rate(ai_errors_total[5m]) / rate(ai_requests_total[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High AI API error rate"
      description: "AI API error rate is {{ $value | humanizePercentage }} for the past 5 minutes"

  - alert: SlowResponseTime
    expr: histogram_quantile(0.95, sum(rate(ai_response_time_seconds_bucket[5m])) by (le)) > 5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Slow AI API response time"
      description: "95th percentile of AI API response time is above 5 seconds: {{ $value | humanizeDuration }}"

  - alert: HighCostRate
    expr: sum(increase(ai_cost_total[1h])) > 10
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "High AI API cost"
      description: "AI API cost is {{ $value | humanizePercentage }} USD in the past hour"

  - alert: TokenQuotaNearingLimit
    expr: sum(increase(ai_token_usage_total[24h])) > 0.8 * 1000000
    for: 30m
    labels:
      severity: warning
    annotations:
      summary: "Token usage nearing daily limit"
      description: "Token usage is at {{ $value | humanize }} (80% of daily quota)"
```

## 응답 품질 모니터링

AI 응답의 품질을 모니터링하는 방법을 살펴보겠습니다.

### 자동 품질 평가 시스템

AI 응답의 품질을 자동으로 평가하는 시스템을 구현할 수 있습니다:

```java
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

@Service
public class ResponseQualityEvaluator {

    private final ChatClient evaluatorChatClient;
    private final MeterRegistry meterRegistry;
    
    // 평가 기준
    private static final List<String> QUALITY_CRITERIA = List.of(
            "relevance", "accuracy", "completeness", "clarity", "helpfulness"
    );
    
    // 점수 추출을 위한 패턴
    private static final Pattern SCORE_PATTERN = 
            Pattern.compile("\"(\\w+)\":\\s*(\\d+)");
    
    public ResponseQualityEvaluator(ChatClient evaluatorChatClient, MeterRegistry meterRegistry) {
        this.evaluatorChatClient = evaluatorChatClient;
        this.meterRegistry = meterRegistry;
    }
    
    public Map<String, Integer> evaluateResponseQuality(String query, String response) {
        // 평가 프롬프트 생성
        SystemMessage systemMessage = new SystemMessage(
                "You are an AI response quality evaluator. " +
                "Your task is to evaluate the quality of an AI assistant's response based on the following criteria:\n" +
                "1. Relevance (1-10): How well the response addresses the user's query\n" +
                "2. Accuracy (1-10): How factually correct the information is\n" +
                "3. Completeness (1-10): How thoroughly the response answers the query\n" +
                "4. Clarity (1-10): How clear and easy to understand the response is\n" +
                "5. Helpfulness (1-10): How helpful the response is for the user's needs\n\n" +
                "Provide your evaluation as a JSON object with numerical scores only, like this:\n" +
                "{\"relevance\": 8, \"accuracy\": 7, \"completeness\": 9, \"clarity\": 8, \"helpfulness\": 7}"
        );
        
        UserMessage userMessage = new UserMessage(
                "User Query: " + query + "\n\n" +
                "AI Response: " + response
        );
        
        Prompt evaluationPrompt = new Prompt(List.of(systemMessage, userMessage));
        
        // 평가 실행
        String evaluationResult = evaluatorChatClient.call(evaluationPrompt)
                .getResult().getOutput().getContent();
        
        // 점수 추출
        Map<String, Integer> scores = extractScores(evaluationResult);
        
        // 메트릭 기록
        recordQualityMetrics(scores);
        
        return scores;
    }
    
    private Map<String, Integer> extractScores(String evaluationResult) {
        Map<String, Integer> scores = new HashMap<>();
        
        Matcher matcher = SCORE_PATTERN.matcher(evaluationResult);
        while (matcher.find()) {
            String criterion = matcher.group(1).toLowerCase();
            int score = Integer.parseInt(matcher.group(2));
            scores.put(criterion, score);
        }
        
        return scores;
    }
    
    private void recordQualityMetrics(Map<String, Integer> scores) {
        // 각 품질 기준에 대한 메트릭 기록
        for (String criterion : QUALITY_CRITERIA) {
            if (scores.containsKey(criterion)) {
                meterRegistry.summary("ai.response.quality", "criterion", criterion)
                        .record(scores.get(criterion));
            }
        }
        
        // 종합 점수 계산 및 기록
        double averageScore = scores.values().stream()
                .mapToInt(Integer::intValue)
                .average()
                .orElse(0.0);
        
        meterRegistry.summary("ai.response.quality.overall").record(averageScore);
    }
}
```

### 사용자 피드백 수집

사용자로부터 AI 응답에 대한 피드백을 수집하는 시스템을 구현할 수 있습니다:

```java
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Service;

@Service
public class UserFeedbackService {

    private final MeterRegistry meterRegistry;
    private final ResponseFeedbackRepository feedbackRepository;
    
    public UserFeedbackService(MeterRegistry meterRegistry, ResponseFeedbackRepository feedbackRepository) {
        this.meterRegistry = meterRegistry;
        this.feedbackRepository = feedbackRepository;
    }
    
    public void recordFeedback(String responseId, String userId, boolean helpful, String comment) {
        // 피드백 저장
        ResponseFeedback feedback = new ResponseFeedback(
                responseId, userId, helpful, comment, Instant.now());
        feedbackRepository.save(feedback);
        
        // 메트릭 기록
        meterRegistry.counter("ai.user.feedback", "helpful", String.valueOf(helpful)).increment();
        
        // 특정 유형의 피드백이 많이 발생하는지 감지하는 로직을 추가할 수 있음
        if (!helpful) {
            checkNegativeFeedbackPattern(responseId);
        }
    }
    
    private void checkNegativeFeedbackPattern(String responseId) {
        // 특정 응답 또는 특정 유형의 쿼리에 대한 부정적인 피드백 패턴 감지
        long recentNegativeFeedbackCount = feedbackRepository.countRecentNegativeFeedback(
                responseId, Instant.now().minus(Duration.ofHours(1)));
        
        if (recentNegativeFeedbackCount > 5) {
            // 알림 발생 또는 로그 기록
            // alertService.sendAlert("High negative feedback rate detected for response type");
        }
    }
    
    // 엔티티 및 레포지토리
    static class ResponseFeedback {
        private String responseId;
        private String userId;
        private boolean helpful;
        private String comment;
        private Instant timestamp;
        
        // 생성자, getter, setter
    }
    
    interface ResponseFeedbackRepository {
        void save(ResponseFeedback feedback);
        long countRecentNegativeFeedback(String responseId, Instant since);
    }
}
```

## 멀티테넌트 환경에서의 관측성

여러 테넌트(고객 또는 조직)를 서비스하는 AI 애플리케이션에서 테넌트별로 관측성 데이터를 분리하는 방법을 살펴보겠습니다.

### 테넌트 컨텍스트 관리

```java
import org.slf4j.MDC;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

public class TenantContextInterceptor implements HandlerInterceptor {

    private static final String TENANT_HEADER = "X-Tenant-ID";
    private static final String TENANT_MDC_KEY = "tenantId";
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 요청 헤더에서 테넌트 ID 추출
        String tenantId = request.getHeader(TENANT_HEADER);
        
        // 테넌트 ID가 없으면 기본값 사용
        if (tenantId == null || tenantId.isEmpty()) {
            tenantId = "default";
        }
        
        // MDC에 테넌트 ID 설정
        MDC.put(TENANT_MDC_KEY, tenantId);
        
        // 요청 처리 계속
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, 
                                Object handler, Exception ex) {
        // MDC에서 테넌트 ID 제거
        MDC.remove(TENANT_MDC_KEY);
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new TenantContextInterceptor());
    }
}
```

### 테넌트별 메트릭 수집

테넌트 ID를 메트릭 태그로 사용하여 테넌트별 메트릭을 수집합니다:

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Tag;
import io.micrometer.core.instrument.Tags;
import org.slf4j.MDC;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

@Service
public class MultitenantAiService {

    private final ChatClient chatClient;
    private final MeterRegistry meterRegistry;
    
    public MultitenantAiService(ChatClient chatClient, MeterRegistry meterRegistry) {
        this.chatClient = chatClient;
        this.meterRegistry = meterRegistry;
    }
    
    public String generateResponse(Prompt prompt) {
        // MDC에서 현재 테넌트 ID 가져오기
        String tenantId = MDC.get("tenantId");
        if (tenantId == null || tenantId.isEmpty()) {
            tenantId = "unknown";
        }
        
        // 테넌트 태그 생성
        Tags tenantTags = Tags.of(Tag.of("tenant", tenantId));
        
        // 요청 카운터 증가 (테넌트별)
        meterRegistry.counter("ai.requests", tenantTags).increment();
        
        try {
            long startTime = System.currentTimeMillis();
            var response = chatClient.call(prompt);
            long duration = System.currentTimeMillis() - startTime;
            
            // 응답 시간 기록 (테넌트별)
            meterRegistry.timer("ai.response.time", tenantTags).record(duration, java.util.concurrent.TimeUnit.MILLISECONDS);
            
            // 토큰 사용량 기록 (테넌트별)
            var usage = response.getMetadata().get("usage");
            if (usage instanceof Map) {
                Map<String, Object> usageMap = (Map<String, Object>) usage;
                int totalTokens = (int) usageMap.getOrDefault("total_tokens", 0);
                meterRegistry.summary("ai.token.usage", tenantTags).record(totalTokens);
            }
            
            return response.getResult().getOutput().getContent();
        } catch (Exception e) {
            // 오류 카운터 증가 (테넌트별)
            meterRegistry.counter("ai.errors", tenantTags).increment();
            throw e;
        }
    }
}
```

### 테넌트별 로깅

각 테넌트의 로그를 분리하여 저장하거나 필터링할 수 있습니다:

```xml
<!-- logback-spring.xml -->
<configuration>
    <!-- 모든 로그를 포함하는 기본 파일 어펜더 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
    </appender>
    
    <!-- 테넌트별 파일 어펜더 -->
    <appender name="TENANT-FILE" class="ch.qos.logback.classic.sift.SiftingAppender">
        <discriminator>
            <key>tenantId</key>
            <defaultValue>unknown</defaultValue>
        </discriminator>
        <sift>
            <appender name="FILE-${tenantId}" class="ch.qos.logback.core.rolling.RollingFileAppender">
                <file>logs/${tenantId}/application.log</file>
                <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                    <fileNamePattern>logs/${tenantId}/application.%d{yyyy-MM-dd}.log</fileNamePattern>
                    <maxHistory>30</maxHistory>
                </rollingPolicy>
                <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
            </appender>
        </sift>
    </appender>
    
    <!-- 루트 로거 설정 -->
    <root level="INFO">
        <appender-ref ref="FILE" />
        <appender-ref ref="TENANT-FILE" />
    </root>
</configuration>
```

## 실제 사용 사례: 대화형 AI 에이전트 모니터링

마지막으로, 지금까지 살펴본 관측성 기법을 활용하여 대화형 AI 에이전트를 종합적으로 모니터링하는 실제 사례를 살펴보겠습니다.

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Consumer;

@Service
public class ConversationalAgentService {

    private static final Logger logger = LoggerFactory.getLogger(ConversationalAgentService.class);
    
    private final ChatClient chatClient;
    private final MeterRegistry meterRegistry;
    private final ConversationRepository conversationRepository;
    private final ResponseQualityEvaluator qualityEvaluator;
    
    // 활성 대화 세션 캐시
    private final Map<String, ConversationSession> activeSessions = new ConcurrentHashMap<>();
    
    public ConversationalAgentService(
            ChatClient chatClient,
            MeterRegistry meterRegistry,
            ConversationRepository conversationRepository,
            ResponseQualityEvaluator qualityEvaluator) {
        this.chatClient = chatClient;
        this.meterRegistry = meterRegistry;
        this.conversationRepository = conversationRepository;
        this.qualityEvaluator = qualityEvaluator;
    }
    
    public String processUserMessage(String sessionId, String userId, String message) {
        // 메트릭 태그 설정
        String tenantId = MDC.get("tenantId");
        Tags metricTags = Tags.of(
                Tag.of("tenant", tenantId != null ? tenantId : "unknown"),
                Tag.of("user", userId)
        );
        
        // 응답 ID 생성
        String responseId = UUID.randomUUID().toString();
        
        // 로깅 컨텍스트 설정
        MDC.put("sessionId", sessionId);
        MDC.put("userId", userId);
        MDC.put("responseId", responseId);
        
        try {
            logger.info("User message received: {}", maskSensitiveData(message));
            
            // 사용자 메시지 추적
            meterRegistry.counter("agent.messages.received", metricTags).increment();
            
            // 대화 세션 관리
            ConversationSession session = getOrCreateSession(sessionId, userId);
            session.addUserMessage(message);
            
            // 대화 유형 분류 및 추적
            String conversationType = classifyConversation(session);
            meterRegistry.counter("agent.conversation.type", 
                    Tags.concat(metricTags, "type", conversationType)).increment();
            
            // 응답 생성 시간 측정
            Timer.Sample timer = Timer.start(meterRegistry);
            
            // AI 응답 생성
            Prompt prompt = session.createPrompt();
            String response = chatClient.call(prompt).getResult().getOutput().getContent();
            
            // 응답 시간 기록
            timer.stop(meterRegistry.timer("agent.response.time", metricTags));
            
            // 세션 업데이트
            session.addAgentResponse(response);
            
            // 대화 저장
            conversationRepository.saveConversation(session.toConversationEntity());
            
            // 응답 품질 평가 (비동기)
            evaluateResponseQualityAsync(message, response, metricTags);
            
            // 응답 로깅
            logger.info("Agent response: {}", maskSensitiveData(response));
            
            // 응답 메트릭 증가
            meterRegistry.counter("agent.messages.sent", metricTags).increment();
            
            return response;
        } catch (Exception e) {
            // 오류 로깅
            logger.error("Error processing message: {}", e.getMessage(), e);
            
            // 오류 메트릭 증가
            meterRegistry.counter("agent.errors", 
                    Tags.concat(metricTags, "error", e.getClass().getSimpleName())).increment();
            
            throw e;
        } finally {
            // MDC 정리
            MDC.remove("sessionId");
            MDC.remove("userId");
            MDC.remove("responseId");
        }
    }
    
    private ConversationSession getOrCreateSession(String sessionId, String userId) {
        return activeSessions.computeIfAbsent(sessionId, id -> {
            // 기존 세션이 있으면 로드, 없으면 새로 생성
            ConversationSession session = conversationRepository
                    .findBySessionId(id)
                    .map(ConversationSession::fromEntity)
                    .orElse(new ConversationSession(id, userId));
            
            // 세션 생성 메트릭
            if (session.isNew()) {
                meterRegistry.counter("agent.sessions.created").increment();
            }
            
            return session;
        });
    }
    
    private String classifyConversation(ConversationSession session) {
        // 간단한 규칙 기반 분류 또는 ML 기반 분류 구현
        String conversationText = session.getAllMessages();
        
        if (conversationText.toLowerCase().contains("help") || 
                conversationText.toLowerCase().contains("support")) {
            return "support";
        } else if (conversationText.toLowerCase().contains("buy") || 
                conversationText.toLowerCase().contains("purchase")) {
            return "sales";
        } else if (conversationText.toLowerCase().contains("price") || 
                conversationText.toLowerCase().contains("cost")) {
            return "pricing";
        }
        
        return "general";
    }
    
    private void evaluateResponseQualityAsync(String query, String response, Tags metricTags) {
        CompletableFuture.runAsync(() -> {
            try {
                Map<String, Integer> qualityScores = qualityEvaluator.evaluateResponseQuality(query, response);
                
                // 품질 점수 기록
                for (Map.Entry<String, Integer> entry : qualityScores.entrySet()) {
                    meterRegistry.summary("agent.response.quality", 
                            Tags.concat(metricTags, "criterion", entry.getKey()))
                            .record(entry.getValue());
                }
                
                // 종합 점수 계산 및 기록
                double averageScore = qualityScores.values().stream()
                        .mapToInt(Integer::intValue)
                        .average()
                        .orElse(0.0);
                
                meterRegistry.summary("agent.response.quality.overall", metricTags).record(averageScore);
                
                // 낮은 품질 응답에 대한 경고
                if (averageScore < 5.0) {
                    logger.warn("Low quality response detected: {}", averageScore);
                    
                    // 낮은 품질 응답 카운터 증가
                    meterRegistry.counter("agent.response.low_quality", metricTags).increment();
                }
            } catch (Exception e) {
                logger.error("Error evaluating response quality: {}", e.getMessage(), e);
            }
        });
    }
    
    private String maskSensitiveData(String text) {
        // 민감한 데이터 마스킹 로직 구현
        return SensitiveDataMasker.maskSensitiveData(text);
    }
}
```

## 결론

이 장에서는 Spring AI 애플리케이션의 관측성과 로깅을 구현하는 방법을 살펴보았습니다. Spring Boot Actuator와 Micrometer를 활용한 메트릭 수집, 구조화된 로깅, 분산 추적, 대시보드 구성, 알림 설정, 응답 품질 모니터링, 멀티테넌트 환경에서의 관측성 등 다양한 기법을 알아보았습니다.

이러한 관측성 기법을 구현하면 AI 애플리케이션의 동작을 더 잘 이해하고, 문제를 신속하게 식별하고 해결할 수 있으며, 사용자 경험을 지속적으로 개선할 수 있습니다. 특히 프로덕션 환경에서 AI 애플리케이션을 운영할 때 이러한 관측성 기능은 필수적입니다.

다음 장에서는 AI 에이전트 설계와 구현에 대해 알아보겠습니다. 이를 통해 Spring AI를 사용하여 더 복잡하고 지능적인 애플리케이션을 구축하는 방법을 배울 수 있습니다.