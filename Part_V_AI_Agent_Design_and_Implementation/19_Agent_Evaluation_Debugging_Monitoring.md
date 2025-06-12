# 19장: 에이전트 평가·디버깅·모니터링

## 19.1 개요

Spring AI는 AI 애플리케이션의 신뢰성과 성능을 보장하기 위한 포괄적인 관찰가능성(Observability) 기능을 제공합니다. 이 장에서는 Spring AI의 평가(Evaluation), 디버깅(Debugging), 모니터링(Monitoring) 기능을 활용하여 프로덕션 환경에서 AI 에이전트를 효과적으로 관리하는 방법을 알아봅니다.

### 19.1.1 Spring AI의 관찰가능성 특징

Spring AI는 Spring 생태계의 관찰가능성 기능을 기반으로 AI 관련 작업에 대한 통찰력을 제공합니다:

- **메트릭과 추적**: ChatClient, ChatModel, EmbeddingModel, ImageModel, VectorStore 등 핵심 컴포넌트에 대한 메트릭과 추적 기능
- **낮은/높은 카디널리티 키**: 메트릭과 추적에는 낮은 카디널리티 키가 추가되고, 추적에만 높은 카디널리티 키가 추가
- **어드바이저 관찰가능성**: 어드바이저 체인의 타이밍과 추적 정보
- **도구 호출 관찰가능성**: 도구 실행, 인수, 결과 추적

### 19.1.2 평가 프레임워크

Spring AI는 AI 응답을 평가하기 위한 `Evaluator` 인터페이스를 제공합니다:

```java
@FunctionalInterface
public interface Evaluator {
    EvaluationResponse evaluate(EvaluationRequest evaluationRequest);
}
```

평가 요청은 다음 정보를 포함합니다:

```java
public class EvaluationRequest {
    private final String userText;        // 사용자 입력
    private final List<Content> dataList; // RAG 등의 컨텍스트 데이터
    private final String responseContent; // AI 모델의 응답
}
```

## 19.2 관찰가능성 구성

### 19.2.1 ChatClient 관찰가능성

ChatClient의 `call()` 또는 `stream()` 작업이 호출될 때 관찰 정보가 기록됩니다.

**낮은 카디널리티 키:**
- `gen_ai.operation.name`: 항상 `framework`
- `gen_ai.system`: 항상 `spring_ai`
- `spring.ai.chat.client.stream`: 스트림 응답 여부
- `spring.ai.kind`: `chat_client`

**높은 카디널리티 키:**
- `gen_ai.prompt`: 프롬프트 내용 (선택적)
- `spring.ai.chat.client.advisors`: 구성된 어드바이저 목록
- `spring.ai.chat.client.conversation.id`: 대화 ID
- `spring.ai.chat.client.tool.names`: 도구 이름 목록

### 19.2.2 프롬프트 로깅 구성

프롬프트 내용은 민감한 정보를 포함할 수 있어 기본적으로 내보내지지 않습니다:

```yaml
spring:
  ai:
    chat:
      client:
        observations:
          log-prompt: false  # 프롬프트 로깅 활성화/비활성화
```

⚠️ **경고**: 프롬프트 로깅을 활성화하면 민감한 정보가 노출될 위험이 있습니다.

### 19.2.3 ChatModel 관찰가능성

ChatModel의 `call()` 또는 `stream()` 메서드 호출 시 관찰 정보가 기록됩니다.

**메트릭:**
- `gen_ai.client.token.usage`: 단일 모델 호출에 사용된 입력/출력 토큰 수

**추적 정보:**
```java
// 낮은 카디널리티 키
- gen_ai.operation.name     // 수행 중인 작업 이름
- gen_ai.system            // 모델 제공자
- gen_ai.request.model     // 요청 모델 이름
- gen_ai.response.model    // 응답 모델 이름

// 높은 카디널리티 키
- gen_ai.request.temperature      // 온도 설정
- gen_ai.request.max_tokens      // 최대 토큰 수
- gen_ai.usage.input_tokens      // 입력 토큰 수
- gen_ai.usage.output_tokens     // 출력 토큰 수
- gen_ai.usage.total_tokens      // 총 토큰 수
- gen_ai.prompt                  // 전체 프롬프트 (선택적)
- gen_ai.completion              // 전체 응답 (선택적)
```

### 19.2.4 프롬프트 및 완성 데이터 로깅

```yaml
spring:
  ai:
    chat:
      observations:
        log-prompt: false      # 프롬프트 로깅
        log-completion: false  # 완성 로깅
        include-error-logging: false  # 오류 로깅 포함
```

### 19.2.5 도구 호출 관찰가능성

Spring AI는 도구 호출에 대한 상세한 관찰 정보를 제공합니다:

**낮은 카디널리티 키:**
- `gen_ai.operation.name`: 항상 `framework`
- `gen_ai.system`: 항상 `spring_ai`
- `spring.ai.kind`: 항상 `tool_call`
- `spring.ai.tool.definition.name`: 도구 이름

**도구 호출 인수 및 결과:**
```yaml
spring:
  ai:
    vectorstore:
      observations:
        include-tool-call-arguments: false  # 기본값: false (보안)
        include-tool-call-result: false     # 기본값: false (보안)
```

### 19.2.6 어드바이저 관찰가능성

어드바이저 실행 시 생성되는 관찰 정보:

```java
// 낮은 카디널리티 키
- gen_ai.operation.name: "framework"
- gen_ai.system: "spring_ai"
- spring.ai.kind: "advisor"

// 높은 카디널리티 키
- spring.ai.advisor.name   // 어드바이저 이름
- spring.ai.advisor.order  // 어드바이저 체인에서의 순서
```

## 19.3 평가 프레임워크

### 19.3.1 Evaluator 인터페이스

Spring AI는 AI 응답을 평가하기 위한 표준 인터페이스를 제공합니다:

```java
@FunctionalInterface
public interface Evaluator {
    EvaluationResponse evaluate(EvaluationRequest evaluationRequest);
}
```

### 19.3.2 RelevancyEvaluator

`RelevancyEvaluator`는 RAG 플로우에서 AI 응답의 관련성을 평가합니다:

```java
@Test
void evaluateRelevancy() {
    String question = "Where does the adventure take place?";
    
    RetrievalAugmentationAdvisor ragAdvisor = RetrievalAugmentationAdvisor.builder()
        .documentRetriever(VectorStoreDocumentRetriever.builder()
            .vectorStore(vectorStore)
            .build())
        .build();
    
    ChatResponse chatResponse = ChatClient.builder(chatModel)
        .build()
        .prompt(question)
        .advisors(ragAdvisor)
        .call()
        .chatResponse();
    
    EvaluationRequest evaluationRequest = new EvaluationRequest(
        question,  // 원본 질문
        chatResponse.getMetadata().get(RetrievalAugmentationAdvisor.DOCUMENT_CONTEXT),
        chatResponse.getResult().getOutput().getText()
    );
    
    RelevancyEvaluator evaluator = new RelevancyEvaluator(ChatClient.builder(chatModel));
    EvaluationResponse evaluationResponse = evaluator.evaluate(evaluationRequest);
    
    assertThat(evaluationResponse.isPass()).isTrue();
}
```

### 19.3.3 FactCheckingEvaluator

`FactCheckingEvaluator`는 AI 응답의 사실성을 평가하여 환각을 감지합니다:

```java
public class FactCheckingEvaluator {
    private final ChatClient.Builder chatClientBuilder;
    
    public FactCheckingEvaluator(ChatClient.Builder chatClientBuilder) {
        this.chatClientBuilder = chatClientBuilder;
    }
    
    // 평가 프롬프트 템플릿
    private static final String FACT_CHECK_TEMPLATE = """
        Document: {document}
        Claim: {claim}
        """;
}
```

### 19.3.4 커스텀 평가 템플릿

평가자는 커스텀 `PromptTemplate`을 제공할 수 있습니다:

```java
RelevancyEvaluator evaluator = RelevancyEvaluator.builder()
    .chatClientBuilder(ChatClient.builder(chatModel))
    .promptTemplate(customPromptTemplate)  // 커스텀 템플릿
    .build();
```

필수 플레이스홀더:
- `query`: 사용자 질문
- `response`: AI 모델 응답
- `context`: 컨텍스트 정보

## 19.4 디버깅 도구

### 19.4.1 로깅 어드바이저

Spring AI는 디버깅을 위한 로깅 어드바이저 예제를 제공합니다:

```java
public class SimpleLoggerAdvisor implements CallAroundAdvisor, StreamAroundAdvisor {
    
    private static final Logger logger = LoggerFactory.getLogger(SimpleLoggerAdvisor.class);
    
    @Override
    public String getName() {
        return this.getClass().getSimpleName();
    }
    
    @Override
    public int getOrder() {
        return 0;
    }
    
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest advisedRequest, CallAroundAdvisorChain chain) {
        logger.debug("BEFORE: {}", advisedRequest);
        
        AdvisedResponse advisedResponse = chain.nextAroundCall(advisedRequest);
        
        logger.debug("AFTER: {}", advisedResponse);
        
        return advisedResponse;
    }
    
    @Override
    public Flux<AdvisedResponse> aroundStream(AdvisedRequest advisedRequest, StreamAroundAdvisorChain chain) {
        logger.debug("BEFORE: {}", advisedRequest);
        
        Flux<AdvisedResponse> advisedResponses = chain.nextAroundStream(advisedRequest);
        
        return new MessageAggregator().aggregateAdvisedResponse(advisedResponses, 
            advisedResponse -> logger.debug("AFTER: {}", advisedResponse));
    }
}
```

### 19.4.2 프롬프트 및 응답 추적

디버깅을 위해 프롬프트와 응답을 추적할 수 있습니다:

```java
@Component
public class PromptResponseTracker {
    
    private final Logger logger = LoggerFactory.getLogger(PromptResponseTracker.class);
    
    @EventListener
    public void handleChatClientObservation(ChatClientObservationEvent event) {
        // 프롬프트 로깅
        if (event.getPrompt() != null) {
            logger.debug("Prompt: {}", event.getPrompt());
        }
        
        // 응답 로깅
        if (event.getResponse() != null) {
            logger.debug("Response: {}", event.getResponse());
        }
        
        // 토큰 사용량 로깅
        if (event.getTokenUsage() != null) {
            logger.debug("Tokens - Input: {}, Output: {}, Total: {}",
                event.getTokenUsage().getInputTokens(),
                event.getTokenUsage().getOutputTokens(),
                event.getTokenUsage().getTotalTokens()
            );
        }
    }
}
```

### 19.4.3 어드바이저 체인 디버깅

어드바이저 체인의 실행 흐름을 디버깅하는 방법:

```java
@Component
public class AdvisorChainDebugger {
    
    public void debugAdvisorChain(ChatClient chatClient) {
        chatClient.prompt("Test prompt")
            .advisors(advisor -> {
                // 어드바이저 체인 확인
                advisor.getAdvisors().forEach(adv -> {
                    System.out.println("Advisor: " + adv.getName() + 
                                     " Order: " + adv.getOrder());
                });
            })
            .call()
            .content();
    }
}
```

## 19.5 안전성 및 콘텐츠 조절

### 19.5.1 SafeGuardAdvisor

Spring AI는 유해한 콘텐츠 생성을 방지하는 `SafeGuardAdvisor`를 제공합니다:

```java
@Configuration
public class SafetyConfiguration {
    
    @Bean
    public ChatClient safeChatClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel)
            .defaultAdvisors(new SafeGuardAdvisor())
            .build();
    }
}
```

### 19.5.2 콘텐츠 조절 API

Spring AI는 OpenAI와 Mistral AI의 조절 서비스를 통합합니다:

```java
// OpenAI Moderation
ModerationApi moderationApi = new OpenAiModerationApi(apiKey);
ModerationResponse response = moderationApi.moderate("Content to check");

// 카테고리별 점수 확인
if (response.getResults().get(0).isFlagged()) {
    // 위험한 콘텐츠 감지됨
    response.getResults().get(0).getCategories()
        .forEach((category, flagged) -> {
            if (flagged) {
                System.out.println("Flagged category: " + category);
            }
        });
}
```

### 19.5.3 비용 및 사용량 추적

AI 자원 사용량을 추적하고 비용을 관리합니다:

```java
@Component
public class UsageTracker {
    
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void trackTokenUsage(TokenUsageEvent event) {
        // 토큰 사용량 메트릭 기록
        meterRegistry.counter("ai.tokens.used",
            "model", event.getModel(),
            "type", "input"
        ).increment(event.getInputTokens());
        
        meterRegistry.counter("ai.tokens.used",
            "model", event.getModel(),
            "type", "output"
        ).increment(event.getOutputTokens());
        
        // 비용 추적
        double estimatedCost = calculateCost(event);
        meterRegistry.gauge("ai.estimated.cost",
            Tags.of("model", event.getModel()),
            estimatedCost
        );
    }
    
    private double calculateCost(TokenUsageEvent event) {
        // 모델별 비용 계산 로직
        return CostCalculator.calculate(
            event.getModel(),
            event.getInputTokens(),
            event.getOutputTokens()
        );
    }
}
```

## 19.5 효율성 및 성능 평가

에이전트의 효율성과 성능을 평가하는 방법을 알아봅니다.

### 19.5.1 응답 시간 측정

```java
@Component
public class ResponseTimeMetric implements EvaluationMetric {
    
    @Override
    public String getName() {
        return "response_time";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 응답 시간을 메타데이터에서 추출
        Map<String, Object> metadata = response.getMetadata();
        if (!metadata.containsKey("responseTimeMs")) {
            return 0.0; // 응답 시간 정보가 없음
        }
        
        long responseTimeMs = (long) metadata.get("responseTimeMs");
        
        // 응답 시간 점수 계산 (낮을수록 좋음)
        // 예: 1초 이내는 1.0, 10초 이상은 0.1
        if (responseTimeMs <= 1000) {
            return 1.0;
        } else if (responseTimeMs >= 10000) {
            return 0.1;
        } else {
            return 1.0 - ((responseTimeMs - 1000) / 9000.0 * 0.9);
        }
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures the response time of the agent",
            "range", "0.0 to 1.0 (higher is faster)"
        );
    }
}
```

### 19.5.2 토큰 사용량 측정

```java
@Component
public class TokenUsageMetric implements EvaluationMetric {
    
    @Override
    public String getName() {
        return "token_efficiency";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 토큰 사용량을 메타데이터에서 추출
        Map<String, Object> metadata = response.getMetadata();
        if (!metadata.containsKey("totalTokens")) {
            return 0.0; // 토큰 정보가 없음
        }
        
        int totalTokens = (int) metadata.get("totalTokens");
        int expectedMaxTokens = getExpectedMaxTokens(sample);
        
        // 토큰 효율성 점수 계산
        if (totalTokens <= expectedMaxTokens) {
            return 1.0;
        } else {
            double overageRatio = (double) totalTokens / expectedMaxTokens;
            return Math.max(0.0, 1.0 - ((overageRatio - 1.0) * 0.5));
        }
    }
    
    private int getExpectedMaxTokens(EvaluationSample sample) {
        // 샘플 메타데이터에서 예상 최대 토큰 수 추출
        Map<String, Object> metadata = sample.getMetadata();
        return metadata.containsKey("expectedMaxTokens")
            ? (int) metadata.get("expectedMaxTokens")
            : 1000; // 기본값
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures the token usage efficiency of the agent",
            "range", "0.0 to 1.0 (higher is more efficient)"
        );
    }
}
```

### 19.5.3 리소스 사용량 측정

```java
@Component
public class ResourceUsageMetric implements EvaluationMetric {
    
    @Override
    public String getName() {
        return "resource_usage";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 리소스 사용량을 메타데이터에서 추출
        Map<String, Object> metadata = response.getMetadata();
        
        double cpuScore = calculateCpuScore(metadata);
        double memoryScore = calculateMemoryScore(metadata);
        double apiCallScore = calculateApiCallScore(metadata);
        
        // 종합 리소스 점수 계산
        return (cpuScore + memoryScore + apiCallScore) / 3.0;
    }
    
    private double calculateCpuScore(Map<String, Object> metadata) {
        // CPU 사용량 점수 계산
        if (!metadata.containsKey("cpuTimeMs")) {
            return 1.0; // 정보가 없으면 최대 점수
        }
        
        long cpuTimeMs = (long) metadata.get("cpuTimeMs");
        return cpuTimeMs < 500 ? 1.0 : Math.max(0.0, 1.0 - ((cpuTimeMs - 500) / 9500.0));
    }
    
    private double calculateMemoryScore(Map<String, Object> metadata) {
        // 메모리 사용량 점수 계산
    }
    
    private double calculateApiCallScore(Map<String, Object> metadata) {
        // API 호출 횟수 점수 계산
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Evaluates the computational resource efficiency of the agent",
            "range", "0.0 to 1.0 (higher is more efficient)"
        );
    }
}
```

## 19.6 실시간 메트릭 수집

Spring AI는 Micrometer를 통해 실시간 메트릭을 수집하고 모니터링할 수 있습니다.

### 19.6.1 토큰 사용량 메트릭

Spring AI는 `gen_ai.client.token.usage` 메트릭을 통해 토큰 사용량을 추적합니다:

```java
@Component
public class TokenUsageMonitor {
    
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void handleChatModelObservation(ChatModelObservationEvent event) {
        // 토큰 사용량 메트릭은 자동으로 수집됨
        // gen_ai.client.token.usage 메트릭으로 접근 가능
        
        // 커스텀 메트릭 추가
        if (event.getUsage() != null) {
            // 모델별 비용 추적
            String model = event.getModel();
            double cost = calculateCost(model, 
                event.getUsage().getInputTokens(),
                event.getUsage().getOutputTokens());
            
            meterRegistry.gauge("ai.model.cost",
                Tags.of("model", model),
                cost
            );
        }
    }
    
    private double calculateCost(String model, int inputTokens, int outputTokens) {
        // 모델별 토큰 가격 계산
        return switch(model) {
            case "gpt-4" -> inputTokens * 0.03 / 1000 + outputTokens * 0.06 / 1000;
            case "gpt-3.5-turbo" -> inputTokens * 0.0005 / 1000 + outputTokens * 0.0015 / 1000;
            default -> 0.0;
        };
    }
}
```

### 19.6.2 VectorStore 메트릭

VectorStore 작업에 대한 관찰가능성:

```java
@Configuration
public class VectorStoreObservabilityConfig {
    
    @Bean
    public VectorStoreObservationConvention vectorStoreObservationConvention() {
        return new DefaultVectorStoreObservationConvention() {
            @Override
            public KeyValues getLowCardinalityKeyValues(
                    VectorStoreObservationContext context) {
                return super.getLowCardinalityKeyValues(context)
                    .and("db.system", context.getDatabaseSystem())
                    .and("db.operation.name", context.getOperationName())
                    .and("spring.ai.kind", "vector_store");
            }
        };
    }
    
    @Component
    public static class VectorStoreMetricsCollector {
        
        private final MeterRegistry meterRegistry;
        
        @EventListener
        public void handleVectorStoreOperation(VectorStoreObservationEvent event) {
            // 벡터 스토어 작업 메트릭
            String operation = event.getOperationName(); // add, delete, query
            String dbSystem = event.getDatabaseSystem();
            
            // 쿼리 성능 추적
            if ("query".equals(operation)) {
                int topK = event.getTopK();
                double similarityThreshold = event.getSimilarityThreshold();
                int documentsReturned = event.getDocumentsReturned();
                
                meterRegistry.gauge("vectorstore.query.documents",
                    Tags.of("db.system", dbSystem),
                    documentsReturned
                );
                
                // 쿼리 효율성 메트릭
                double efficiency = (double) documentsReturned / topK;
                meterRegistry.gauge("vectorstore.query.efficiency",
                    Tags.of("db.system", dbSystem),
                    efficiency
                );
            }
        }
    }
}
```

### 19.6.3 프롬프트 및 완성 로깅

민감한 정보를 포함할 수 있는 프롬프트와 완성 데이터의 로깅 구성:

```yaml
spring:
  ai:
    chat:
      client:
        observations:
          log-prompt: true  # ChatClient 프롬프트 로깅
      observations:
        log-prompt: true      # ChatModel 프롬프트 로깅
        log-completion: true  # ChatModel 완성 로깅
    image:
      observations:
        log-prompt: true      # ImageModel 프롬프트 로깅
    vectorstore:
      observations:
        log-query-response: true  # VectorStore 쿼리 응답 로깅
```

```java
@Component
@ConditionalOnProperty(name = "spring.ai.chat.observations.log-prompt", 
                      havingValue = "true")
public class PromptLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(PromptLogger.class);
    
    @EventListener
    public void logChatModelPrompt(ChatModelPromptEvent event) {
        // 추적 정보와 함께 프롬프트 로깅
        MDC.put("traceId", event.getTraceId());
        MDC.put("spanId", event.getSpanId());
        
        logger.debug("Chat Model Prompt: {}", event.getPrompt());
        
        // 민감한 정보 마스킹
        String maskedPrompt = maskSensitiveData(event.getPrompt());
        logger.info("Masked Prompt: {}", maskedPrompt);
        
        MDC.clear();
    }
    
    private String maskSensitiveData(String prompt) {
        // 이메일, 전화번호, 신용카드 번호 등 마스킹
        return prompt
            .replaceAll("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b", "[EMAIL]")
            .replaceAll("\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b", "[CARD]")
            .replaceAll("\\b\\d{3}[-.]?\\d{3}[-.]?\\d{4}\\b", "[PHONE]");
    }
}
```

## 19.7 모니터링 대시보드 구성

Spring AI의 관찰가능성 데이터를 활용한 모니터링 대시보드 구성 방법을 알아봅니다.

### 19.7.1 Grafana 대시보드 구성

```java
@Configuration
public class GrafanaDashboardConfig {
    
    @Bean
    public GrafanaDashboardExporter grafanaDashboardExporter() {
        return GrafanaDashboardExporter.builder()
            .withDashboardTitle("Spring AI Observability")
            .withDefaultPanels()
            .build();
    }
    
    // Grafana 대시보드 JSON 템플릿
    public String getAIDashboardTemplate() {
        return """
        {
          "dashboard": {
            "title": "Spring AI Metrics",
            "panels": [
              {
                "title": "Token Usage by Model",
                "targets": [{
                  "expr": "sum(rate(gen_ai_client_token_usage_total[5m])) by (gen_ai_request_model)"
                }]
              },
              {
                "title": "Chat Model Response Time",
                "targets": [{
                  "expr": "histogram_quantile(0.95, gen_ai_client_operation_duration_seconds_bucket)"
                }]
              },
              {
                "title": "Vector Store Query Performance",
                "targets": [{
                  "expr": "rate(db_vector_client_operation_duration_seconds_sum[5m])"
                }]
              },
              {
                "title": "Error Rate by Operation",
                "targets": [{
                  "expr": "sum(rate(gen_ai_client_operation_errors_total[5m])) by (gen_ai_operation_name)"
                }]
              }
            ]
          }
        }
        """;
    }
}

// Prometheus 메트릭 수집 구성
@Configuration
public class PrometheusConfig {
    
    @Bean
    public PrometheusMeterRegistry prometheusMeterRegistry() {
        PrometheusMeterRegistry registry = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
        
        // AI 관련 메트릭 필터링
        registry.config().meterFilter(
            MeterFilter.maximumAllowableMetrics("gen_ai", 1000)
        );
        
        return registry;
    }
}
```

### 19.7.2 알림 규칙 구성

```java
@Component
public class AIAlertingRules {
    
    private final MeterRegistry meterRegistry;
    private final NotificationService notificationService;
    
    @Scheduled(fixedRate = 60000) // 1분마다 체크
    public void checkAlertConditions() {
        // 토큰 사용량 임계값 체크
        checkTokenUsageThreshold();
        
        // 에러율 체크
        checkErrorRate();
        
        // 응답 시간 체크
        checkResponseTime();
        
        // 비용 임계값 체크
        checkCostThreshold();
    }
    
    private void checkTokenUsageThreshold() {
        // 시간당 토큰 사용량 체크
        double tokensPerHour = meterRegistry.counter("gen_ai.client.token.usage")
            .getId()
            .getTag("gen_ai.usage.type")
            .equals("total") ? getRate("gen_ai.client.token.usage") : 0;
        
        if (tokensPerHour > 1_000_000) {
            notificationService.sendAlert(Alert.builder()
                .severity(AlertSeverity.WARNING)
                .title("High Token Usage")
                .message(String.format("Token usage rate: %.0f tokens/hour", tokensPerHour))
                .build());
        }
    }
    
    private void checkErrorRate() {
        // 5분간 에러율 계산
        double errorRate = calculateErrorRate("gen_ai.client.operation", 5);
        
        if (errorRate > 0.05) { // 5% 이상
            notificationService.sendAlert(Alert.builder()
                .severity(AlertSeverity.CRITICAL)
                .title("High Error Rate")
                .message(String.format("Error rate: %.2f%%", errorRate * 100))
                .build());
        }
    }
    
    private void checkResponseTime() {
        // P95 응답 시간 체크
        double p95ResponseTime = meterRegistry.timer("gen_ai.client.operation.duration")
            .takeSnapshot()
            .percentileValue(0.95);
        
        if (p95ResponseTime > 10_000) { // 10초 이상
            notificationService.sendAlert(Alert.builder()
                .severity(AlertSeverity.WARNING)
                .title("Slow Response Time")
                .message(String.format("P95 response time: %.2fs", p95ResponseTime / 1000))
                .build());
        }
    }
    
    private void checkCostThreshold() {
        // 일일 비용 임계값 체크
        double dailyCost = meterRegistry.gauge("ai.model.cost.daily", 0.0);
        
        if (dailyCost > 1000) { // $1000 이상
            notificationService.sendAlert(Alert.builder()
                .severity(AlertSeverity.CRITICAL)
                .title("Cost Threshold Exceeded")
                .message(String.format("Daily cost: $%.2f", dailyCost))
                .metadata(Map.of(
                    "action", "Review AI usage patterns",
                    "dashboard", "/grafana/ai-cost-analysis"
                ))
                .build());
        }
    }
    
    private double getRate(String metricName) {
        // 메트릭의 변화율 계산
        return meterRegistry.counter(metricName).count() / 3600.0; // 시간당 비율
    }
    
    private double calculateErrorRate(String metricName, int minutes) {
        // 특정 기간 동안의 에러율 계산
        double totalRequests = meterRegistry.counter(metricName + ".total").count();
        double errors = meterRegistry.counter(metricName + ".errors").count();
        return totalRequests > 0 ? errors / totalRequests : 0;
    }
}
```

### 19.7.3 분산 추적 구성

```java
@Configuration
@EnableConfigurationProperties(TracingProperties.class)
public class DistributedTracingConfig {
    
    @Bean
    public Tracer tracer() {
        return OTel.getGlobalTracer("spring-ai-application");
    }
    
    @Bean
    public ObservationRegistry observationRegistry(Tracer tracer) {
        ObservationRegistry registry = ObservationRegistry.create();
        
        // OpenTelemetry 브릿지 구성
        registry.observationConfig()
            .observationHandler(new TracingObservationHandler(tracer));
        
        return registry;
    }
    
    // AI 작업에 대한 커스텀 스팬 생성
    @Component
    public static class AIOperationTracer {
        
        private final Tracer tracer;
        
        public <T> T traceAIOperation(String operationName, 
                                     Supplier<T> operation,
                                     Map<String, String> attributes) {
            Span span = tracer.spanBuilder(operationName)
                .setSpanKind(SpanKind.CLIENT)
                .setAttribute("gen_ai.system", "spring_ai")
                .setAllAttributes(Attributes.of(
                    AttributeKey.stringKey("gen_ai.operation.name"), operationName
                ))
                .startSpan();
            
            try (Scope scope = span.makeCurrent()) {
                // 추가 속성 설정
                attributes.forEach((key, value) -> 
                    span.setAttribute(key, value));
                
                T result = operation.get();
                
                // 결과 메타데이터 추가
                if (result instanceof ChatResponse) {
                    ChatResponse response = (ChatResponse) result;
                    span.setAttribute("gen_ai.usage.total_tokens", 
                        response.getMetadata().getUsage().getTotalTokens());
                }
                
                span.setStatus(StatusCode.OK);
                return result;
                
            } catch (Exception e) {
                span.recordException(e);
                span.setStatus(StatusCode.ERROR, e.getMessage());
                throw e;
            } finally {
                span.end();
            }
        }
    }
    
    // Jaeger 구성
    @Bean
    @ConditionalOnProperty(name = "spring.ai.tracing.backend", havingValue = "jaeger")
    public JaegerTracer jaegerTracer(TracingProperties properties) {
        return JaegerTracer.builder()
            .serviceName(properties.getServiceName())
            .endpoint(properties.getJaeger().getEndpoint())
            .build();
    }
}

@ConfigurationProperties(prefix = "spring.ai.tracing")
public class TracingProperties {
    private String serviceName = "spring-ai-app";
    private String backend = "jaeger";
    private JaegerProperties jaeger = new JaegerProperties();
    
    // getters and setters...
    
    public static class JaegerProperties {
        private String endpoint = "http://localhost:14250";
        // getters and setters...
    }
}
```

## 19.8 통합 테스팅 패턴

Spring AI의 평가 프레임워크를 활용한 통합 테스트 패턴을 알아봅니다.

### 19.8.1 RAG 평가 테스트

```java
@SpringBootTest
@AutoConfigureMockMvc
public class RAGEvaluationTest {
    
    @Autowired
    private ChatModel chatModel;
    
    @Autowired
    private VectorStore vectorStore;
    
    @Test
    void testRAGRelevancy() {
        // 문서 준비 및 저장
        List<Document> documents = List.of(
            new Document("The capital of France is Paris.", Map.of("country", "France")),
            new Document("The capital of Germany is Berlin.", Map.of("country", "Germany")),
            new Document("The capital of Italy is Rome.", Map.of("country", "Italy"))
        );
        
        vectorStore.add(documents);
        
        // RAG 어드바이저 구성
        String question = "What is the capital of France?";
        
        RetrievalAugmentationAdvisor ragAdvisor = RetrievalAugmentationAdvisor.builder()
            .documentRetriever(VectorStoreDocumentRetriever.builder()
                .vectorStore(vectorStore)
                .similarityThreshold(0.7)
                .topK(3)
                .build())
            .build();
        
        // ChatClient로 질문
        ChatResponse chatResponse = ChatClient.builder(chatModel)
            .build()
            .prompt(question)
            .advisors(ragAdvisor)
            .call()
            .chatResponse();
        
        // 평가 요청 생성
        EvaluationRequest evaluationRequest = new EvaluationRequest(
            question,
            chatResponse.getMetadata().get(RetrievalAugmentationAdvisor.DOCUMENT_CONTEXT),
            chatResponse.getResult().getOutput().getText()
        );
        
        // RelevancyEvaluator로 평가
        RelevancyEvaluator evaluator = RelevancyEvaluator.builder()
            .chatClientBuilder(ChatClient.builder(chatModel))
            .build();
        
        EvaluationResponse evaluationResponse = evaluator.evaluate(evaluationRequest);
        
        // 검증
        assertThat(evaluationResponse.isPass()).isTrue();
        assertThat(chatResponse.getResult().getOutput().getText())
            .containsIgnoringCase("Paris");
    }
    
    @ParameterizedTest
    @CsvSource({
        "What is the capital of France?, France, Paris",
        "What is the capital of Germany?, Germany, Berlin",
        "What is the capital of Italy?, Italy, Rome"
    })
    void testMultipleRAGQueries(String question, String expectedCountry, String expectedCapital) {
        // RAG 플로우 실행 및 평가
        ChatResponse response = executeRAGQuery(question);
        
        // 응답 검증
        assertThat(response.getResult().getOutput().getText())
            .containsIgnoringCase(expectedCapital);
        
        // 메타데이터 검증
        List<Content> retrievedDocs = response.getMetadata()
            .get(RetrievalAugmentationAdvisor.DOCUMENT_CONTEXT);
        
        assertThat(retrievedDocs)
            .isNotEmpty()
            .anyMatch(doc -> doc.getText().contains(expectedCapital));
    }
}
```

### 19.8.2 알림 시스템 구현

```java
@Service
public class AgentAlertingService {
    
    private final MeterRegistry meterRegistry;
    private final NotificationService notificationService;
    private final List<AlertRule> alertRules;
    
    @Scheduled(fixedRate = 60000) // 1분마다 실행
    public void checkAlerts() {
        for (AlertRule rule : alertRules) {
            // 규칙에 따른 메트릭 검사
            boolean isTriggered = checkMetricAgainstRule(rule);
            
            if (isTriggered) {
                // 알림 생성 및 전송
                Alert alert = createAlert(rule);
                notificationService.sendAlert(alert);
            }
        }
    }
    
    private boolean checkMetricAgainstRule(AlertRule rule) {
        // 규칙에 따른 메트릭 평가
        String metricName = rule.getMetricName();
        String agentName = rule.getAgentName();
        
        double currentValue = fetchMetricValue(metricName, agentName);
        
        switch (rule.getOperator()) {
            case GREATER_THAN:
                return currentValue > rule.getThreshold();
            case LESS_THAN:
                return currentValue < rule.getThreshold();
            case EQUALS:
                return Math.abs(currentValue - rule.getThreshold()) < 0.0001;
            default:
                return false;
        }
    }
    
    private double fetchMetricValue(String metricName, String agentName) {
        // 메트릭 값 조회
        Search search = Search.in(meterRegistry).name(metricName);
        
        if (agentName != null) {
            search = search.tag("agent", agentName);
        }
        
        return search.meter()
            .map(meter -> {
                if (meter instanceof Counter) {
                    return ((Counter) meter).count();
                } else if (meter instanceof Gauge) {
                    return ((Gauge) meter).value();
                } else if (meter instanceof Timer) {
                    return ((Timer) meter).mean(TimeUnit.MILLISECONDS);
                } else {
                    return 0.0;
                }
            })
            .orElse(0.0);
    }
    
    private Alert createAlert(AlertRule rule) {
        // 알림 객체 생성
        String metricName = rule.getMetricName();
        String agentName = rule.getAgentName();
        double currentValue = fetchMetricValue(metricName, agentName);
        
        return Alert.builder()
            .type("METRIC_ALERT")
            .severity(rule.getSeverity())
            .message(String.format(
                "Alert for %s on agent %s: value %.2f %s threshold %.2f",
                metricName,
                agentName != null ? agentName : "all",
                currentValue,
                rule.getOperator().getSymbol(),
                rule.getThreshold()
            ))
            .details(Map.of(
                "metricName", metricName,
                "agentName", agentName != null ? agentName : "all",
                "currentValue", currentValue,
                "threshold", rule.getThreshold(),
                "operator", rule.getOperator().toString()
            ))
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

### 19.8.3 대시보드 및 시각화

```java
@RestController
@RequestMapping("/api/monitoring")
public class AgentMonitoringController {
    
    private final AgentMetricsService metricsService;
    private final AgentEvaluationService evaluationService;
    private final AlertHistoryService alertService;
    
    @GetMapping("/metrics/summary")
    public ResponseEntity<MetricsSummary> getMetricsSummary(
            @RequestParam(required = false) String agentName,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end) {
        
        // 메트릭 요약 조회
        MetricsSummary summary = metricsService.getMetricsSummary(agentName, start, end);
        return ResponseEntity.ok(summary);
    }
    
    @GetMapping("/metrics/timeseries")
    public ResponseEntity<MetricsTimeSeries> getMetricsTimeSeries(
            @RequestParam String metricName,
            @RequestParam(required = false) String agentName,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end,
            @RequestParam(defaultValue = "PT1M") Duration interval) {
        
        // 시계열 메트릭 조회
        MetricsTimeSeries timeSeries = metricsService.getMetricsTimeSeries(
            metricName, agentName, start, end, interval);
        return ResponseEntity.ok(timeSeries);
    }
    
    @GetMapping("/evaluation/scores")
    public ResponseEntity<List<EvaluationScore>> getEvaluationScores(
            @RequestParam(required = false) String agentName,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end) {
        
        // 평가 점수 조회
        List<EvaluationScore> scores = evaluationService.getEvaluationScores(agentName, start, end);
        return ResponseEntity.ok(scores);
    }
    
    @GetMapping("/alerts")
    public ResponseEntity<List<Alert>> getAlerts(
            @RequestParam(required = false) String agentName,
            @RequestParam(required = false) AlertSeverity severity,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end) {
        
        // 알림 이력 조회
        List<Alert> alerts = alertService.getAlerts(agentName, severity, start, end);
        return ResponseEntity.ok(alerts);
    }
}
```

## 19.9 에이전트 개선 및 최적화 프로세스

에이전트 평가 결과를 바탕으로 지속적인 개선과 최적화를 수행하는 방법을 알아봅니다.

### 19.9.1 A/B 테스팅 프레임워크

```java
@Service
public class AgentABTestingService {
    
    private final Random random = new Random();
    private final Map<String, ABTest> activeTests = new ConcurrentHashMap<>();
    private final ABTestResultRepository resultRepository;
    private final EvaluationService evaluationService;
    
    public String createABTest(ABTestConfiguration config) {
        // A/B 테스트 생성
        String testId = UUID.randomUUID().toString();
        
        ABTest test = new ABTest();
        test.setId(testId);
        test.setName(config.getName());
        test.setDescription(config.getDescription());
        test.setControlAgent(config.getControlAgent());
        test.setVariantAgent(config.getVariantAgent());
        test.setEvaluationDataset(config.getEvaluationDataset());
        test.setMetrics(config.getMetrics());
        test.setTrafficSplit(config.getTrafficSplit());
        test.setStartTime(LocalDateTime.now());
        test.setStatus("RUNNING");
        
        activeTests.put(testId, test);
        
        return testId;
    }
    
    public Agent selectAgentForRequest(String testId, AgentRequest request) {
        // 요청에 대한 에이전트 선택 (A 또는 B)
        ABTest test = activeTests.get(testId);
        if (test == null) {
            throw new IllegalArgumentException("A/B test not found: " + testId);
        }
        
        // 트래픽 분할에 따른 에이전트 선택
        double randomValue = random.nextDouble();
        boolean useVariant = randomValue < test.getTrafficSplit();
        
        Agent selectedAgent = useVariant ? test.getVariantAgent() : test.getControlAgent();
        
        // 선택 이력 기록
        recordAgentSelection(testId, request, selectedAgent, useVariant);
        
        return selectedAgent;
    }
    
    public void recordTestResult(String testId, AgentRequest request, AgentResponse response, boolean isVariant) {
        // 테스트 결과 기록
        ABTestResult result = new ABTestResult();
        result.setTestId(testId);
        result.setRequestId(UUID.randomUUID().toString());
        result.setRequest(request);
        result.setResponse(response);
        result.setVariant(isVariant);
        result.setTimestamp(LocalDateTime.now());
        
        // 평가 지표에 따른 점수 계산
        ABTest test = activeTests.get(testId);
        Map<String, Double> metricScores = new HashMap<>();
        
        for (String metricName : test.getMetrics()) {
            // 각 지표별 점수 계산
            EvaluationMetric metric = evaluationService.getMetric(metricName);
            EvaluationSample sample = findMatchingSample(test.getEvaluationDataset(), request);
            
            if (sample != null && metric != null) {
                double score = metric.evaluate(sample, response);
                metricScores.put(metricName, score);
            }
        }
        
        result.setMetricScores(metricScores);
        resultRepository.save(result);
    }
    
    public ABTestReport generateTestReport(String testId) {
        // 테스트 보고서 생성
        ABTest test = activeTests.get(testId);
        if (test == null) {
            throw new IllegalArgumentException("A/B test not found: " + testId);
        }
        
        List<ABTestResult> results = resultRepository.findByTestId(testId);
        
        // 컨트롤과 변형 그룹 분리
        List<ABTestResult> controlResults = results.stream()
            .filter(r -> !r.isVariant())
            .collect(Collectors.toList());
        
        List<ABTestResult> variantResults = results.stream()
            .filter(ABTestResult::isVariant)
            .collect(Collectors.toList());
        
        // 각 지표별 평균 점수 계산
        Map<String, ABTestMetricComparison> metricComparisons = new HashMap<>();
        
        for (String metricName : test.getMetrics()) {
            double controlAverage = calculateAverageScore(controlResults, metricName);
            double variantAverage = calculateAverageScore(variantResults, metricName);
            double improvement = variantAverage - controlAverage;
            double relativeChange = controlAverage != 0 ? improvement / controlAverage * 100 : 0;
            
            metricComparisons.put(metricName, new ABTestMetricComparison(
                metricName, controlAverage, variantAverage, improvement, relativeChange));
        }
        
        return new ABTestReport(test, controlResults.size(), variantResults.size(), metricComparisons);
    }
    
    private void recordAgentSelection(String testId, AgentRequest request, Agent selectedAgent, boolean isVariant) {
        // 에이전트 선택 이력 기록
    }
    
    private EvaluationSample findMatchingSample(EvaluationDataset dataset, AgentRequest request) {
        // 요청과 일치하는 평가 샘플 찾기
    }
    
    private double calculateAverageScore(List<ABTestResult> results, String metricName) {
        // 평균 점수 계산
        return results.stream()
            .map(r -> r.getMetricScores().getOrDefault(metricName, 0.0))
            .mapToDouble(Double::doubleValue)
            .average()
            .orElse(0.0);
    }
}
```

### 19.9.2 성능 최적화 도구

```java
@Service
public class AgentOptimizationService {
    
    private final ModelSelectionService modelSelectionService;
    private final PromptOptimizationService promptOptimizationService;
    private final TokenUsageAnalyzer tokenUsageAnalyzer;
    private final ParameterTuningService parameterTuningService;
    
    public OptimizationReport analyzeAndOptimize(Agent agent, EvaluationDataset dataset) {
        OptimizationReport report = new OptimizationReport();
        report.setAgentType(agent.getClass().getName());
        report.setTimestamp(LocalDateTime.now());
        
        // 1. 토큰 사용량 분석
        TokenUsageAnalysis tokenAnalysis = tokenUsageAnalyzer.analyzeTokenUsage(agent, dataset);
        report.setTokenUsageAnalysis(tokenAnalysis);
        
        // 2. 프롬프트 최적화 제안
        PromptOptimizationSuggestions promptSuggestions = 
            promptOptimizationService.generateSuggestions(agent, dataset);
        report.setPromptSuggestions(promptSuggestions);
        
        // 3. 모델 선택 분석
        ModelSelectionAnalysis modelAnalysis = modelSelectionService.analyzeModelFit(agent, dataset);
        report.setModelAnalysis(modelAnalysis);
        
        // 4. 파라미터 튜닝 제안
        ParameterTuningSuggestions parameterSuggestions = 
            parameterTuningService.suggestParameterAdjustments(agent, dataset);
        report.setParameterSuggestions(parameterSuggestions);
        
        // 5. 비용-성능 분석
        CostPerformanceAnalysis costAnalysis = analyzeCostPerformance(agent, dataset);
        report.setCostAnalysis(costAnalysis);
        
        return report;
    }
    
    private CostPerformanceAnalysis analyzeCostPerformance(Agent agent, EvaluationDataset dataset) {
        // 비용-성능 분석 로직
        CostPerformanceAnalysis analysis = new CostPerformanceAnalysis();
        
        // 샘플당 평균 비용 계산
        double totalCost = 0.0;
        for (EvaluationSample sample : dataset.getSamples()) {
            AgentResponse response = agent.execute(sample.getInput());
            double cost = calculateResponseCost(response);
            totalCost += cost;
        }
        
        double averageCost = totalCost / dataset.size();
        analysis.setAverageCostPerQuery(averageCost);
        
        // 성능당 비용 계산
        double performanceScore = evaluatePerformance(agent, dataset);
        analysis.setPerformanceScore(performanceScore);
        analysis.setCostPerformanceRatio(averageCost / performanceScore);
        
        // 최적화 제안 생성
        List<OptimizationSuggestion> suggestions = generateCostOptimizationSuggestions(analysis);
        analysis.setOptimizationSuggestions(suggestions);
        
        return analysis;
    }
    
    private double calculateResponseCost(AgentResponse response) {
        // 응답 비용 계산 로직
    }
    
    private double evaluatePerformance(Agent agent, EvaluationDataset dataset) {
        // 에이전트 성능 평가 로직
    }
    
    private List<OptimizationSuggestion> generateCostOptimizationSuggestions(CostPerformanceAnalysis analysis) {
        // 비용 최적화 제안 생성
    }
}
```

### 19.9.3 지속적 개선 파이프라인

```java
@Service
public class ContinuousImprovementPipeline {
    
    private final AgentRepository agentRepository;
    private final EvaluationDatasetRepository datasetRepository;
    private final EvaluationService evaluationService;
    private final AgentOptimizationService optimizationService;
    private final AgentVersioningService versioningService;
    
    @Scheduled(cron = "0 0 0 * * ?") // 매일 자정에 실행
    public void runImprovementPipeline() {
        // 개선 대상 에이전트 목록 조회
        List<Agent> agents = agentRepository.findByImprovementEnabled(true);
        
        for (Agent agent : agents) {
            try {
                // 에이전트별 개선 프로세스 실행
                improveAgent(agent);
            } catch (Exception e) {
                // 오류 처리
                log.error("Error improving agent: {}", agent.getClass().getName(), e);
            }
        }
    }
    
    private void improveAgent(Agent agent) {
        // 1. 평가 데이터셋 로드
        EvaluationDataset dataset = loadEvaluationDataset(agent);
        
        // 2. 현재 성능 평가
        EvaluationResult currentPerformance = evaluationService.evaluateAgent(agent, dataset);
        
        // 3. 성능 최적화 분석
        OptimizationReport optimizationReport = optimizationService.analyzeAndOptimize(agent, dataset);
        
        // 4. 개선 항목 필터링 (임계값 이상의 개선 효과가 예상되는 항목)
        List<ImprovementAction> improvementActions = filterImprovementActions(
            optimizationReport, currentPerformance);
        
        if (improvementActions.isEmpty()) {
            log.info("No significant improvements identified for agent: {}", agent.getClass().getName());
            return;
        }
        
        // 5. 개선 항목 적용
        Agent improvedAgent = applyImprovementActions(agent, improvementActions);
        
        // 6. 개선된 에이전트 평가
        EvaluationResult improvedPerformance = evaluationService.evaluateAgent(improvedAgent, dataset);
        
        // 7. 개선 효과 검증
        boolean isImproved = validateImprovement(currentPerformance, improvedPerformance);
        
        if (isImproved) {
            // 8. 개선된 버전 저장
            String newVersion = versioningService.createNewVersion(
                agent, improvedAgent, improvementActions, improvedPerformance);
            
            log.info("Created improved version {} for agent: {}", 
                newVersion, agent.getClass().getName());
        } else {
            log.info("Improvements did not yield significant results for agent: {}", 
                agent.getClass().getName());
        }
    }
    
    private EvaluationDataset loadEvaluationDataset(Agent agent) {
        // 에이전트에 적합한 평가 데이터셋 로드
    }
    
    private List<ImprovementAction> filterImprovementActions(
            OptimizationReport report, 
            EvaluationResult currentPerformance) {
        // 개선 효과가 기대되는 항목 필터링
    }
    
    private Agent applyImprovementActions(Agent agent, List<ImprovementAction> actions) {
        // 개선 항목 적용
    }
    
    private boolean validateImprovement(
            EvaluationResult currentPerformance, 
            EvaluationResult improvedPerformance) {
        // 개선 효과 검증
    }
}
```

## 19.10 실전 에이전트 평가 사례

실제 애플리케이션에서 에이전트를 평가하고 모니터링하는 구체적인 사례를 살펴봅니다.

### 19.10.1 고객 지원 에이전트 평가

```java
@Service
public class CustomerSupportAgentEvaluator {
    
    private final CustomerSupportAgent agent;
    private final EvaluationDatasetService datasetService;
    private final EvaluationService evaluationService;
    private final UserFeedbackRepository feedbackRepository;
    
    public CustomerSupportEvaluationReport evaluateAgent() {
        // 1. 기술적 평가
        EvaluationDataset technicalDataset = datasetService.getDataset("customer_support_technical");
        EvaluationResult technicalResult = evaluationService.evaluateAgent(agent, technicalDataset);
        
        // 2. 사용자 만족도 평가
        UserSatisfactionMetrics satisfactionMetrics = evaluateUserSatisfaction();
        
        // 3. 비즈니스 지표 평가
        BusinessMetrics businessMetrics = evaluateBusinessMetrics();
        
        // 4. 결합 보고서 생성
        return new CustomerSupportEvaluationReport(
            technicalResult, satisfactionMetrics, businessMetrics);
    }
    
    private UserSatisfactionMetrics evaluateUserSatisfaction() {
        // 사용자 만족도 평가 로직
        LocalDateTime oneMonthAgo = LocalDateTime.now().minusMonths(1);
        List<UserFeedback> recentFeedback = feedbackRepository.findAfter(oneMonthAgo);
        
        // 평균 만족도 점수
        double averageRating = recentFeedback.stream()
            .mapToDouble(UserFeedback::getRating)
            .average()
            .orElse(0.0);
        
        // 긍정적 피드백 비율
        long positiveCount = recentFeedback.stream()
            .filter(feedback -> feedback.getRating() >= 4.0)
            .count();
        
        double positiveRatio = (double) positiveCount / recentFeedback.size();
        
        // 부정적 피드백 분석
        List<UserFeedback> negativeFeedback = recentFeedback.stream()
            .filter(feedback -> feedback.getRating() < 3.0)
            .collect(Collectors.toList());
        
        Map<String, Integer> issueCategories = analyzeNegativeFeedback(negativeFeedback);
        
        return new UserSatisfactionMetrics(
            averageRating, positiveRatio, issueCategories);
    }
    
    private Map<String, Integer> analyzeNegativeFeedback(List<UserFeedback> negativeFeedback) {
        // 부정적 피드백 카테고리화 및 분석
    }
    
    private BusinessMetrics evaluateBusinessMetrics() {
        // 비즈니스 지표 평가 로직
        
        // 티켓 처리 속도
        double averageResolutionTime = calculateAverageResolutionTime();
        
        // 에스컬레이션 비율
        double escalationRate = calculateEscalationRate();
        
        // 첫 번째 응답에서 해결된 비율
        double firstContactResolutionRate = calculateFirstContactResolutionRate();
        
        // 비용 절감 추정치
        double costSavingsEstimate = estimateCostSavings();
        
        return new BusinessMetrics(
            averageResolutionTime,
            escalationRate,
            firstContactResolutionRate,
            costSavingsEstimate
        );
    }
    
    // 비즈니스 지표 계산 메서드들...
}
```

### 19.10.2 RAG 기반 지식 검색 에이전트 평가

```java
@Service
public class KnowledgeSearchAgentEvaluator {
    
    private final KnowledgeSearchAgent agent;
    private final DocumentRepository documentRepository;
    private final EvaluationService evaluationService;
    private final TextEmbedding embeddingService;
    
    public KnowledgeSearchEvaluationReport evaluateAgent() {
        // 1. 정보 검색 정확성 평가
        RetrievalAccuracyResult retrievalAccuracy = evaluateRetrievalAccuracy();
        
        // 2. 응답 품질 평가
        ResponseQualityResult responseQuality = evaluateResponseQuality();
        
        // 3. 지식 커버리지 평가
        KnowledgeCoverageResult knowledgeCoverage = evaluateKnowledgeCoverage();
        
        return new KnowledgeSearchEvaluationReport(
            retrievalAccuracy, responseQuality, knowledgeCoverage);
    }
    
    private RetrievalAccuracyResult evaluateRetrievalAccuracy() {
        // 정보 검색 정확성 평가
        EvaluationDataset retrievalDataset = createRetrievalEvaluationDataset();
        
        List<RetrievalEvaluation> evaluations = new ArrayList<>();
        
        for (EvaluationSample sample : retrievalDataset.getSamples()) {
            // 에이전트 실행
            AgentRequest request = sample.getInput();
            AgentResponse response = agent.execute(request);
            
            // 검색된 문서 추출
            List<Document> retrievedDocuments = extractRetrievedDocuments(response);
            
            // 관련성 점수 계산
            List<String> expectedDocumentIds = getExpectedDocumentIds(sample);
            
            double precision = calculatePrecision(retrievedDocuments, expectedDocumentIds);
            double recall = calculateRecall(retrievedDocuments, expectedDocumentIds);
            double f1Score = calculateF1Score(precision, recall);
            
            evaluations.add(new RetrievalEvaluation(
                sample.getId(), precision, recall, f1Score, retrievedDocuments.size()));
        }
        
        return new RetrievalAccuracyResult(evaluations);
    }
    
    private ResponseQualityResult evaluateResponseQuality() {
        // 응답 품질 평가
        EvaluationDataset qualityDataset = createQualityEvaluationDataset();
        EvaluationResult result = evaluationService.evaluateAgent(agent, qualityDataset);
        
        List<String> evaluatedMetrics = Arrays.asList(
            "accuracy", "completeness", "relevance", "conciseness", "coherence");
        
        Map<String, Double> metricScores = new HashMap<>();
        for (String metric : evaluatedMetrics) {
            metricScores.put(metric, result.getAverageScore(metric));
        }
        
        return new ResponseQualityResult(metricScores);
    }
    
    private KnowledgeCoverageResult evaluateKnowledgeCoverage() {
        // 지식 커버리지 평가
        
        // 1. 전체 문서 코퍼스 로드
        List<Document> allDocuments = documentRepository.findAll();
        
        // 2. 모든 문서의 내용 추출 및 임베딩
        List<float[]> documentEmbeddings = new ArrayList<>();
        for (Document doc : allDocuments) {
            float[] embedding = embeddingService.embed(doc.getContent()).getVector();
            documentEmbeddings.add(embedding);
        }
        
        // 3. 임베딩 공간 분석
        KnowledgeCoverageAnalysis coverageAnalysis = analyzeEmbeddingCoverage(documentEmbeddings);
        
        // 4. 랜덤 쿼리 생성 및 유사 문서 검색
        List<CoverageTestResult> testResults = performCoverageTests(allDocuments);
        
        return new KnowledgeCoverageResult(coverageAnalysis, testResults);
    }
    
    // 정보 검색 정확성 평가 관련 메서드들...
    
    // 응답 품질 평가 관련 메서드들...
    
    // 지식 커버리지 평가 관련 메서드들...
}
```

### 19.10.3 다중 에이전트 시스템 평가

```java
@Service
public class MultiAgentSystemEvaluator {
    
    private final List<Agent> agents;
    private final AgentCoordinator coordinator;
    private final EvaluationService evaluationService;
    private final SystemPerformanceMonitor performanceMonitor;
    
    public MultiAgentEvaluationReport evaluateSystem() {
        // 1. 개별 에이전트 평가
        Map<String, EvaluationResult> agentResults = new HashMap<>();
        for (Agent agent : agents) {
            String agentName = agent.getClass().getSimpleName();
            EvaluationDataset dataset = getDatasetForAgent(agent);
            EvaluationResult result = evaluationService.evaluateAgent(agent, dataset);
            agentResults.put(agentName, result);
        }
        
        // 2. 통합 시스템 평가
        SystemEvaluationResult systemResult = evaluateIntegratedSystem();
        
        // 3. 에이전트 간 상호작용 평가
        InteractionEvaluationResult interactionResult = evaluateAgentInteractions();
        
        // 4. 시스템 성능 평가
        PerformanceEvaluationResult performanceResult = evaluateSystemPerformance();
        
        return new MultiAgentEvaluationReport(
            agentResults, systemResult, interactionResult, performanceResult);
    }
    
    private EvaluationDataset getDatasetForAgent(Agent agent) {
        // 에이전트별 평가 데이터셋 결정
    }
    
    private SystemEvaluationResult evaluateIntegratedSystem() {
        // 통합 시스템 평가 로직
        EvaluationDataset systemDataset = createSystemEvaluationDataset();
        
        List<SystemTestResult> testResults = new ArrayList<>();
        
        for (EvaluationSample sample : systemDataset.getSamples()) {
            // 시스템 실행
            AgentRequest request = sample.getInput();
            AgentResponse response = coordinator.processRequest(request);
            
            // 결과 평가
            boolean taskCompleted = isTaskCompleted(response, sample);
            int steps = countExecutionSteps(response);
            long executionTime = extractExecutionTime(response);
            
            testResults.add(new SystemTestResult(
                sample.getId(), 
                taskCompleted, 
                steps, 
                executionTime
            ));
        }
        
        // 종합 결과 계산
        double taskCompletionRate = calculateTaskCompletionRate(testResults);
        double averageSteps = calculateAverageSteps(testResults);
        double averageExecutionTime = calculateAverageExecutionTime(testResults);
        
        return new SystemEvaluationResult(
            taskCompletionRate,
            averageSteps,
            averageExecutionTime,
            testResults
        );
    }
    
    private InteractionEvaluationResult evaluateAgentInteractions() {
        // 에이전트 간 상호작용 평가 로직
        List<InteractionTestResult> results = new ArrayList<>();
        
        // 에이전트 쌍 간 메시지 전달 정확성 테스트
        for (int i = 0; i < agents.size(); i++) {
            for (int j = 0; j < agents.size(); j++) {
                if (i != j) {
                    Agent sender = agents.get(i);
                    Agent receiver = agents.get(j);
                    
                    InteractionTestResult result = testAgentInteraction(sender, receiver);
                    results.add(result);
                }
            }
        }
        
        // 대화 연속성 테스트
        ConversationContinuityResult continuityResult = testConversationContinuity();
        
        // 작업 위임 테스트
        TaskDelegationResult delegationResult = testTaskDelegation();
        
        return new InteractionEvaluationResult(
            results, continuityResult, delegationResult);
    }
    
    private PerformanceEvaluationResult evaluateSystemPerformance() {
        // 시스템 성능 평가 로직
        return performanceMonitor.collectPerformanceMetrics();
    }
    
    // 기타 평가 관련 메서드들...
}
```

## 19.11 결론

이 장에서는 Spring AI 에이전트를 효과적으로 평가하고, 디버깅하며, 모니터링하는 다양한 방법을 살펴보았습니다. 평가 프레임워크, 디버깅 도구, 모니터링 시스템, 그리고 지속적 개선 파이프라인을 구현하는 방법을 통해 에이전트의 품질과 성능을 지속적으로 향상시킬 수 있습니다.

에이전트를 체계적으로 평가하고 모니터링함으로써 AI 시스템의 안정성과 신뢰성을 확보할 수 있으며, 사용자에게 더 나은 경험을 제공할 수 있습니다. 또한, 지속적인 개선 프로세스를 통해 시간이 지남에 따라 에이전트의 성능이 향상되도록 할 수 있습니다.

이로써 "Part V: AI 에이전트 설계와 구현"을 마무리합니다. 다음 파트에서는 지금까지 배운 내용을 바탕으로 실전 프로젝트와 케이스 스터디를 통해 Spring AI의 활용 방법을 더 깊이 살펴보겠습니다.

## 19.12 프로덕션 디버깅 기법

프로덕션 환경에서 에이전트를 효과적으로 디버깅하기 위한 고급 기법을 살펴봅니다.

### 19.12.1 분산 추적 기반 디버깅

```java
@Component
public class DistributedAgentDebugger {
    
    private final Tracer tracer;
    private final SpanProcessor spanProcessor;
    private final TraceSampler sampler;
    
    public void debugAgentExecution(String agentId, String sessionId) {
        // 특정 세션에 대한 추적 활성화
        sampler.enableFullSampling(sessionId);
        
        // 스팬 프로세서에 디버그 핸들러 추가
        spanProcessor.addHandler(new DebugSpanHandler() {
            @Override
            public void onSpanStart(ReadableSpan span) {
                if (isAgentRelated(span, agentId)) {
                    // 디버그 정보 수집
                    collectDebugInfo(span);
                }
            }
            
            @Override
            public void onSpanEnd(ReadableSpan span) {
                if (isAgentRelated(span, agentId)) {
                    // 실행 경로 분석
                    analyzeExecutionPath(span);
                }
            }
        });
    }
    
    private void collectDebugInfo(ReadableSpan span) {
        // 스팬 속성 추출
        Map<String, Object> attributes = span.getAttributes().asMap();
        
        // 프롬프트와 응답 캡처
        if (attributes.containsKey("gen_ai.prompt")) {
            String prompt = (String) attributes.get("gen_ai.prompt");
            debugLogger.logPrompt(span.getSpanContext().getTraceId(), prompt);
        }
        
        // 토큰 사용량 추적
        if (attributes.containsKey("gen_ai.usage.total_tokens")) {
            int tokens = (int) attributes.get("gen_ai.usage.total_tokens");
            debugLogger.logTokenUsage(span.getSpanContext().getTraceId(), tokens);
        }
    }
    
    private void analyzeExecutionPath(ReadableSpan span) {
        // 실행 경로 재구성
        List<SpanData> path = reconstructExecutionPath(span);
        
        // 병목 지점 식별
        BottleneckAnalysis bottlenecks = identifyBottlenecks(path);
        
        // 오류 패턴 분석
        ErrorPatternAnalysis errors = analyzeErrorPatterns(path);
        
        // 디버그 보고서 생성
        DebugReport report = new DebugReport(
            span.getSpanContext().getTraceId(),
            path,
            bottlenecks,
            errors
        );
        
        debugLogger.logReport(report);
    }
}
```

### 19.12.2 실시간 에이전트 상태 검사

```java
@RestController
@RequestMapping("/api/debug/agent")
public class AgentDebugController {
    
    private final AgentRegistry agentRegistry;
    private final AgentStateInspector stateInspector;
    private final DebugSessionManager debugSessionManager;
    
    @PostMapping("/{agentId}/debug/start")
    public ResponseEntity<DebugSession> startDebugSession(
            @PathVariable String agentId,
            @RequestBody DebugConfiguration config) {
        
        Agent agent = agentRegistry.getAgent(agentId);
        if (agent == null) {
            return ResponseEntity.notFound().build();
        }
        
        // 디버그 세션 시작
        DebugSession session = debugSessionManager.createSession(agent, config);
        
        // 에이전트에 디버그 인터셉터 추가
        agent.addInterceptor(new DebugInterceptor(session));
        
        return ResponseEntity.ok(session);
    }
    
    @GetMapping("/{agentId}/state")
    public ResponseEntity<AgentState> inspectAgentState(
            @PathVariable String agentId,
            @RequestParam(required = false) String sessionId) {
        
        Agent agent = agentRegistry.getAgent(agentId);
        if (agent == null) {
            return ResponseEntity.notFound().build();
        }
        
        // 에이전트 상태 검사
        AgentState state = stateInspector.inspect(agent, sessionId);
        
        return ResponseEntity.ok(state);
    }
    
    @PostMapping("/{agentId}/debug/breakpoint")
    public ResponseEntity<Void> setBreakpoint(
            @PathVariable String agentId,
            @RequestBody BreakpointConfiguration breakpoint) {
        
        DebugSession session = debugSessionManager.getActiveSession(agentId);
        if (session == null) {
            return ResponseEntity.badRequest().build();
        }
        
        // 브레이크포인트 설정
        session.addBreakpoint(breakpoint);
        
        return ResponseEntity.ok().build();
    }
    
    @GetMapping("/{agentId}/debug/trace/{traceId}")
    public ResponseEntity<ExecutionTrace> getExecutionTrace(
            @PathVariable String agentId,
            @PathVariable String traceId) {
        
        // 실행 추적 정보 조회
        ExecutionTrace trace = stateInspector.getExecutionTrace(agentId, traceId);
        
        if (trace == null) {
            return ResponseEntity.notFound().build();
        }
        
        return ResponseEntity.ok(trace);
    }
}

@Component
public class DebugInterceptor implements AgentInterceptor {
    
    private final DebugSession session;
    private final AtomicBoolean paused = new AtomicBoolean(false);
    
    @Override
    public void beforeExecution(AgentContext context) {
        // 브레이크포인트 체크
        if (session.hasBreakpoint(context)) {
            paused.set(true);
            notifyDebugger(context);
            
            // 디버거 명령 대기
            waitForDebuggerCommand();
        }
        
        // 실행 컨텍스트 기록
        session.recordContext(context);
    }
    
    @Override
    public void afterExecution(AgentContext context, AgentResult result) {
        // 실행 결과 기록
        session.recordResult(context, result);
    }
    
    @Override
    public void onError(AgentContext context, Throwable error) {
        // 오류 정보 상세 기록
        session.recordError(context, error);
        
        // 오류 발생 시 자동 중단
        if (session.isPauseOnError()) {
            paused.set(true);
            notifyDebugger(context, error);
            waitForDebuggerCommand();
        }
    }
}
```

### 19.12.3 메모리 및 리소스 프로파일링

```java
@Service
public class AgentResourceProfiler {
    
    private final MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
    private final ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
    
    public ResourceProfile profileAgentExecution(Agent agent, AgentRequest request) {
        // 프로파일링 시작
        ResourceSnapshot startSnapshot = captureResourceSnapshot();
        
        // 에이전트 실행
        long startTime = System.nanoTime();
        AgentResponse response = agent.execute(request);
        long endTime = System.nanoTime();
        
        // 프로파일링 종료
        ResourceSnapshot endSnapshot = captureResourceSnapshot();
        
        // 리소스 사용량 계산
        return calculateResourceUsage(startSnapshot, endSnapshot, endTime - startTime);
    }
    
    private ResourceSnapshot captureResourceSnapshot() {
        ResourceSnapshot snapshot = new ResourceSnapshot();
        
        // 메모리 사용량
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        snapshot.setHeapUsed(heapUsage.getUsed());
        snapshot.setHeapMax(heapUsage.getMax());
        
        // 스레드 정보
        snapshot.setThreadCount(threadBean.getThreadCount());
        snapshot.setPeakThreadCount(threadBean.getPeakThreadCount());
        
        // CPU 시간
        snapshot.setCpuTime(threadBean.getCurrentThreadCpuTime());
        
        // GC 정보
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        long totalGcCount = 0;
        long totalGcTime = 0;
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            totalGcCount += gcBean.getCollectionCount();
            totalGcTime += gcBean.getCollectionTime();
        }
        snapshot.setGcCount(totalGcCount);
        snapshot.setGcTime(totalGcTime);
        
        return snapshot;
    }
    
    private ResourceProfile calculateResourceUsage(
            ResourceSnapshot start, 
            ResourceSnapshot end, 
            long executionTimeNanos) {
        
        ResourceProfile profile = new ResourceProfile();
        
        // 실행 시간
        profile.setExecutionTimeMs(executionTimeNanos / 1_000_000);
        
        // 메모리 사용량 변화
        profile.setMemoryDelta(end.getHeapUsed() - start.getHeapUsed());
        profile.setPeakMemory(Math.max(start.getHeapUsed(), end.getHeapUsed()));
        
        // CPU 사용량
        profile.setCpuTimeMs((end.getCpuTime() - start.getCpuTime()) / 1_000_000);
        
        // GC 영향
        profile.setGcCount(end.getGcCount() - start.getGcCount());
        profile.setGcTimeMs(end.getGcTime() - start.getGcTime());
        
        // 스레드 사용량
        profile.setPeakThreads(end.getPeakThreadCount());
        
        return profile;
    }
}
```

## 19.13 프로덕션 오류 추적 및 진단

### 19.13.1 지능형 오류 분석 시스템

```java
@Service
public class IntelligentErrorAnalyzer {
    
    private final ErrorPatternRepository patternRepository;
    private final ChatModel chatModel;
    private final ErrorClassifier errorClassifier;
    
    public ErrorAnalysisReport analyzeError(AgentError error) {
        // 1. 오류 분류
        ErrorClassification classification = errorClassifier.classify(error);
        
        // 2. 유사 오류 패턴 검색
        List<ErrorPattern> similarPatterns = findSimilarPatterns(error);
        
        // 3. 근본 원인 분석
        RootCauseAnalysis rootCause = analyzeRootCause(error, similarPatterns);
        
        // 4. AI 기반 해결책 제안
        List<Solution> solutions = generateSolutions(error, rootCause);
        
        // 5. 영향도 평가
        ImpactAssessment impact = assessImpact(error);
        
        return new ErrorAnalysisReport(
            classification,
            rootCause,
            solutions,
            impact,
            similarPatterns
        );
    }
    
    private RootCauseAnalysis analyzeRootCause(AgentError error, List<ErrorPattern> patterns) {
        // 스택 트레이스 분석
        StackTraceAnalysis stackAnalysis = analyzeStackTrace(error.getStackTrace());
        
        // 컨텍스트 분석
        ContextAnalysis contextAnalysis = analyzeErrorContext(error.getContext());
        
        // 타이밍 분석
        TimingAnalysis timingAnalysis = analyzeErrorTiming(error);
        
        // AI를 활용한 종합 분석
        String prompt = buildRootCausePrompt(stackAnalysis, contextAnalysis, timingAnalysis);
        
        ChatResponse response = ChatClient.builder(chatModel)
            .build()
            .prompt(prompt)
            .call()
            .chatResponse();
        
        return parseRootCauseAnalysis(response.getResult().getOutput().getText());
    }
    
    private List<Solution> generateSolutions(AgentError error, RootCauseAnalysis rootCause) {
        List<Solution> solutions = new ArrayList<>();
        
        // 1. 알려진 해결책 검색
        solutions.addAll(findKnownSolutions(error, rootCause));
        
        // 2. AI 기반 해결책 생성
        String solutionPrompt = buildSolutionPrompt(error, rootCause);
        
        ChatResponse response = ChatClient.builder(chatModel)
            .build()
            .prompt(solutionPrompt)
            .call()
            .chatResponse();
        
        solutions.addAll(parseSolutions(response.getResult().getOutput().getText()));
        
        // 3. 해결책 우선순위 지정
        prioritizeSolutions(solutions, error, rootCause);
        
        return solutions;
    }
}
```

### 19.13.2 오류 재현 및 테스트 시스템

```java
@Service
public class ErrorReproductionService {
    
    private final AgentRegistry agentRegistry;
    private final RequestRecorder requestRecorder;
    private final EnvironmentSimulator environmentSimulator;
    
    public ReproductionResult reproduceError(String errorId) {
        // 1. 오류 정보 로드
        AgentError error = loadError(errorId);
        
        // 2. 원본 요청 복원
        AgentRequest originalRequest = requestRecorder.getRequest(error.getRequestId());
        
        // 3. 환경 상태 복원
        EnvironmentState originalState = error.getEnvironmentSnapshot();
        environmentSimulator.restoreState(originalState);
        
        // 4. 에이전트 상태 복원
        Agent agent = agentRegistry.getAgent(error.getAgentId());
        restoreAgentState(agent, error.getAgentState());
        
        // 5. 오류 재현 시도
        List<ReproductionAttempt> attempts = new ArrayList<>();
        
        for (int i = 0; i < 5; i++) {
            ReproductionAttempt attempt = attemptReproduction(agent, originalRequest, error);
            attempts.add(attempt);
            
            if (attempt.isSuccessful()) {
                break;
            }
            
            // 재현 실패 시 조건 조정
            adjustReproductionConditions(attempt);
        }
        
        return new ReproductionResult(error, attempts);
    }
    
    private ReproductionAttempt attemptReproduction(
            Agent agent, 
            AgentRequest request, 
            AgentError originalError) {
        
        ReproductionAttempt attempt = new ReproductionAttempt();
        attempt.setStartTime(LocalDateTime.now());
        
        try {
            // 디버그 모드로 실행
            AgentResponse response = agent.executeInDebugMode(request);
            attempt.setResponse(response);
            attempt.setSuccessful(false);
            attempt.setReason("Error not reproduced");
            
        } catch (Exception e) {
            attempt.setError(e);
            
            // 오류 비교
            boolean isSameError = compareErrors(originalError, e);
            attempt.setSuccessful(isSameError);
            
            if (isSameError) {
                // 상세 진단 정보 수집
                collectDiagnosticInfo(attempt, agent, request);
            }
        }
        
        attempt.setEndTime(LocalDateTime.now());
        return attempt;
    }
    
    private void collectDiagnosticInfo(
            ReproductionAttempt attempt, 
            Agent agent, 
            AgentRequest request) {
        
        DiagnosticInfo diagnostics = new DiagnosticInfo();
        
        // 메모리 덤프
        diagnostics.setMemoryDump(captureMemoryDump());
        
        // 스레드 덤프
        diagnostics.setThreadDump(captureThreadDump());
        
        // 에이전트 상태 스냅샷
        diagnostics.setAgentSnapshot(agent.captureSnapshot());
        
        // 실행 추적
        diagnostics.setExecutionTrace(agent.getLastExecutionTrace());
        
        attempt.setDiagnostics(diagnostics);
    }
}
```

## 19.14 고급 모니터링 구성

### 19.14.1 커스텀 메트릭 및 알림

```java
@Configuration
public class AdvancedMonitoringConfig {
    
    @Bean
    public MeterRegistry customMeterRegistry() {
        CompositeMeterRegistry composite = new CompositeMeterRegistry();
        
        // Prometheus 레지스트리
        PrometheusMeterRegistry prometheus = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
        composite.add(prometheus);
        
        // CloudWatch 레지스트리
        CloudWatchMeterRegistry cloudWatch = new CloudWatchMeterRegistry(
            cloudWatchConfig(), Clock.SYSTEM, CloudWatchAsyncClient.create());
        composite.add(cloudWatch);
        
        // 커스텀 필터 및 변환
        composite.config()
            .meterFilter(new AgentMetricFilter())
            .meterFilter(MeterFilter.denyNameStartsWith("jvm"))
            .meterFilter(MeterFilter.maximumAllowableMetrics(10000));
        
        return composite;
    }
    
    @Component
    public static class AgentMetricFilter implements MeterFilter {
        
        @Override
        public Meter.Id map(Meter.Id id) {
            // AI 관련 메트릭에 태그 추가
            if (id.getName().startsWith("gen_ai") || id.getName().startsWith("spring.ai")) {
                return id.withTag("service", "ai-agent");
            }
            return id;
        }
        
        @Override
        public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
            // 응답 시간 메트릭에 대한 히스토그램 구성
            if (id.getName().contains("duration")) {
                return DistributionStatisticConfig.builder()
                    .percentilesHistogram(true)
                    .percentiles(0.5, 0.75, 0.95, 0.99)
                    .serviceLevelObjectives(
                        Duration.ofMillis(100).toNanos(),
                        Duration.ofMillis(500).toNanos(),
                        Duration.ofSeconds(1).toNanos(),
                        Duration.ofSeconds(5).toNanos()
                    )
                    .build()
                    .merge(config);
            }
            return config;
        }
    }
    
    @Bean
    public AlertManager alertManager(MeterRegistry meterRegistry) {
        return AlertManager.builder()
            .meterRegistry(meterRegistry)
            .alertRules(Arrays.asList(
                // 높은 오류율 알림
                AlertRule.builder()
                    .name("high-error-rate")
                    .query("rate(gen_ai_client_operation_errors_total[5m]) > 0.1")
                    .severity(AlertSeverity.CRITICAL)
                    .annotations(Map.of(
                        "summary", "High AI operation error rate",
                        "description", "Error rate is above 10% for the last 5 minutes"
                    ))
                    .build(),
                
                // 토큰 사용량 급증 알림
                AlertRule.builder()
                    .name("token-usage-spike")
                    .query("rate(gen_ai_client_token_usage_total[5m]) > 10000")
                    .severity(AlertSeverity.WARNING)
                    .annotations(Map.of(
                        "summary", "Token usage spike detected",
                        "description", "Token consumption rate exceeds 10k tokens/5min"
                    ))
                    .build(),
                
                // 응답 시간 저하 알림
                AlertRule.builder()
                    .name("slow-response-time")
                    .query("histogram_quantile(0.95, gen_ai_client_operation_duration_seconds_bucket) > 10")
                    .severity(AlertSeverity.WARNING)
                    .annotations(Map.of(
                        "summary", "Slow AI response times",
                        "description", "95th percentile response time is above 10 seconds"
                    ))
                    .build()
            ))
            .build();
    }
}
```

### 19.14.2 실시간 대시보드 구성

```java
@RestController
@RequestMapping("/api/monitoring/dashboard")
public class RealTimeDashboardController {
    
    private final MeterRegistry meterRegistry;
    private final AgentMetricsService metricsService;
    private final SseEmitter.SseEventBuilder eventBuilder;
    
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamMetrics() {
        SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);
        
        // 실시간 메트릭 스트리밍
        ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
        executor.scheduleAtFixedRate(() -> {
            try {
                DashboardMetrics metrics = collectCurrentMetrics();
                emitter.send(SseEmitter.event()
                    .name("metrics")
                    .data(metrics));
            } catch (IOException e) {
                emitter.completeWithError(e);
            }
        }, 0, 1, TimeUnit.SECONDS);
        
        emitter.onCompletion(() -> executor.shutdown());
        emitter.onTimeout(() -> executor.shutdown());
        
        return emitter;
    }
    
    private DashboardMetrics collectCurrentMetrics() {
        DashboardMetrics metrics = new DashboardMetrics();
        metrics.setTimestamp(Instant.now());
        
        // 토큰 사용량
        metrics.setTokenUsage(new TokenUsageMetrics(
            meterRegistry.counter("gen_ai.client.token.usage", 
                "gen_ai.usage.type", "input").count(),
            meterRegistry.counter("gen_ai.client.token.usage", 
                "gen_ai.usage.type", "output").count()
        ));
        
        // 응답 시간 통계
        Timer responseTimer = meterRegistry.timer("gen_ai.client.operation.duration");
        metrics.setResponseTimeStats(new ResponseTimeStats(
            responseTimer.mean(TimeUnit.MILLISECONDS),
            responseTimer.max(TimeUnit.MILLISECONDS),
            getPercentile(responseTimer, 0.95),
            getPercentile(responseTimer, 0.99)
        ));
        
        // 에러율
        double totalRequests = meterRegistry.counter("gen_ai.client.operation.total").count();
        double errors = meterRegistry.counter("gen_ai.client.operation.errors").count();
        metrics.setErrorRate(totalRequests > 0 ? errors / totalRequests : 0);
        
        // 활성 에이전트 상태
        metrics.setActiveAgents(metricsService.getActiveAgentStates());
        
        return metrics;
    }
    
    @GetMapping("/config")
    public ResponseEntity<DashboardConfiguration> getDashboardConfig() {
        DashboardConfiguration config = new DashboardConfiguration();
        
        // 위젯 구성
        config.setWidgets(Arrays.asList(
            new Widget("token-usage", "Token Usage", "line-chart", 
                Map.of("metrics", Arrays.asList("input_tokens", "output_tokens"))),
            
            new Widget("response-time", "Response Time", "gauge", 
                Map.of("metric", "p95_response_time", "max", 10000)),
            
            new Widget("error-rate", "Error Rate", "percentage", 
                Map.of("metric", "error_rate", "threshold", 0.05)),
            
            new Widget("agent-status", "Agent Status", "status-grid", 
                Map.of("agents", metricsService.getRegisteredAgents()))
        ));
        
        return ResponseEntity.ok(config);
    }
}
```

### 19.14.3 예측적 모니터링

```java
@Service
public class PredictiveMonitoringService {
    
    private final TimeSeriesAnalyzer timeSeriesAnalyzer;
    private final AnomalyDetector anomalyDetector;
    private final ChatModel chatModel;
    
    @Scheduled(fixedRate = 300000) // 5분마다 실행
    public void analyzeTrendsAndPredict() {
        // 1. 시계열 데이터 수집
        Map<String, TimeSeries> metrics = collectTimeSeriesData();
        
        // 2. 트렌드 분석
        Map<String, TrendAnalysis> trends = new HashMap<>();
        for (Map.Entry<String, TimeSeries> entry : metrics.entrySet()) {
            TrendAnalysis trend = timeSeriesAnalyzer.analyzeTrend(entry.getValue());
            trends.put(entry.getKey(), trend);
        }
        
        // 3. 이상 징후 감지
        List<Anomaly> anomalies = anomalyDetector.detectAnomalies(metrics);
        
        // 4. 예측 생성
        List<Prediction> predictions = generatePredictions(trends, anomalies);
        
        // 5. 조치 권고사항 생성
        List<ActionRecommendation> recommendations = generateRecommendations(predictions);
        
        // 6. 알림 발송
        notifyIfCritical(predictions, recommendations);
    }
    
    private List<Prediction> generatePredictions(
            Map<String, TrendAnalysis> trends, 
            List<Anomaly> anomalies) {
        
        List<Prediction> predictions = new ArrayList<>();
        
        // 토큰 사용량 예측
        TrendAnalysis tokenTrend = trends.get("token_usage");
        if (tokenTrend != null && tokenTrend.isIncreasing()) {
            double projectedUsage = tokenTrend.projectValue(Duration.ofHours(24));
            if (projectedUsage > getTokenLimit() * 0.8) {
                predictions.add(new Prediction(
                    "token_limit_breach",
                    "Token limit may be exceeded within 24 hours",
                    PredictionSeverity.HIGH,
                    0.85,
                    LocalDateTime.now().plusHours(20)
                ));
            }
        }
        
        // 성능 저하 예측
        TrendAnalysis responseTrend = trends.get("response_time");
        if (responseTrend != null && responseTrend.getDegradationRate() > 0.1) {
            predictions.add(new Prediction(
                "performance_degradation",
                "Response time degradation detected",
                PredictionSeverity.MEDIUM,
                0.75,
                LocalDateTime.now().plusHours(6)
            ));
        }
        
        // AI 기반 예측
        String aiPredictionPrompt = buildPredictionPrompt(trends, anomalies);
        ChatResponse response = ChatClient.builder(chatModel)
            .build()
            .prompt(aiPredictionPrompt)
            .call()
            .chatResponse();
        
        predictions.addAll(parseAIPredictions(response.getResult().getOutput().getText()));
        
        return predictions;
    }
    
    private List<ActionRecommendation> generateRecommendations(List<Prediction> predictions) {
        List<ActionRecommendation> recommendations = new ArrayList<>();
        
        for (Prediction prediction : predictions) {
            switch (prediction.getType()) {
                case "token_limit_breach":
                    recommendations.add(new ActionRecommendation(
                        "Optimize prompts or increase token limit",
                        ActionPriority.HIGH,
                        Arrays.asList(
                            "Review and optimize prompt templates",
                            "Implement prompt caching",
                            "Consider upgrading API limits"
                        )
                    ));
                    break;
                    
                case "performance_degradation":
                    recommendations.add(new ActionRecommendation(
                        "Investigate performance bottlenecks",
                        ActionPriority.MEDIUM,
                        Arrays.asList(
                            "Check vector store query performance",
                            "Review agent complexity",
                            "Consider response caching"
                        )
                    ));
                    break;
            }
        }
        
        return recommendations;
    }
}
```

## 19.15 모범 사례 및 주의사항

### 19.15.1 평가 모범 사례

1. **체계적인 평가 계획 수립**
   - 평가 목표와 기준을 명확히 정의
   - 다양한 관점에서의 평가 지표 설정
   - 정기적인 평가 주기 설정

2. **균형 잡힌 평가 데이터셋**
   - 실제 사용 사례를 반영한 데이터셋 구성
   - 엣지 케이스와 일반적인 경우의 균형
   - 주기적인 데이터셋 업데이트

3. **자동화된 평가 파이프라인**
   - CI/CD 파이프라인에 평가 통합
   - 회귀 테스트 자동화
   - 성능 벤치마크 추적

### 19.15.2 디버깅 모범 사례

1. **구조화된 로깅**
   ```java
   @Component
   public class StructuredAgentLogger {
       private static final Logger logger = LoggerFactory.getLogger(StructuredAgentLogger.class);
       
       public void logAgentExecution(AgentContext context, AgentResult result) {
           logger.info("Agent execution completed", 
               kv("agent_id", context.getAgentId()),
               kv("session_id", context.getSessionId()),
               kv("execution_time_ms", result.getExecutionTimeMs()),
               kv("token_count", result.getTokenCount()),
               kv("success", result.isSuccess())
           );
       }
   }
   ```

2. **컨텍스트 보존**
   - 요청-응답 전체 사이클 추적
   - 관련 메타데이터 보존
   - 재현 가능한 디버그 정보 수집

3. **점진적 디버깅**
   - 로그 레벨 동적 조정
   - 특정 세션/사용자에 대한 상세 로깅
   - 성능 영향 최소화

### 19.15.3 모니터링 모범 사례

1. **계층적 모니터링**
   - 인프라 레벨 모니터링
   - 애플리케이션 레벨 모니터링
   - 비즈니스 레벨 모니터링

2. **알림 피로 방지**
   - 의미 있는 임계값 설정
   - 알림 그룹화 및 억제
   - 우선순위 기반 알림

3. **비용 최적화**
   - 메트릭 수집 최적화
   - 로그 보존 정책 설정
   - 샘플링 전략 구현

### 19.15.4 일반적인 함정과 해결책

1. **과도한 메트릭 수집**
   - 문제: 스토리지 비용 증가, 성능 저하
   - 해결: 중요 메트릭 선별, 집계 사용

2. **민감 정보 노출**
   - 문제: 프롬프트/응답에 민감 정보 포함
   - 해결: 자동 마스킹, 접근 제어

3. **평가 편향**
   - 문제: 특정 사용 사례에 편향된 평가
   - 해결: 다양한 평가 시나리오, A/B 테스팅

## 19.16 결론

이 장에서는 Spring AI 에이전트의 평가, 디버깅, 모니터링에 대한 포괄적인 방법론과 도구를 살펴보았습니다. 효과적인 관찰가능성 구현은 AI 시스템의 신뢰성과 성능을 보장하는 핵심 요소입니다.

주요 내용을 요약하면:

1. **평가 프레임워크**: 체계적인 에이전트 성능 평가
2. **디버깅 도구**: 프로덕션 환경에서의 효과적인 문제 해결
3. **모니터링 시스템**: 실시간 성능 추적 및 이상 징후 감지
4. **지속적 개선**: 데이터 기반 최적화 및 개선

이러한 도구와 기법을 활용하여 안정적이고 신뢰할 수 있는 AI 에이전트 시스템을 구축하고 운영할 수 있습니다.

## 연습 문제

1. Spring AI 에이전트를 평가하기 위한 간단한 평가 프레임워크를 설계하고 구현해 보세요. 정확성, 일관성, 응답 시간 등의 지표를 포함해야 합니다.
2. 에이전트의 응답을 평가하기 위한 의미적 유사성 메트릭을 구현해 보세요. Embedding 서비스를 활용하여 코사인 유사도를 계산하는 방법을 사용하세요.
3. 에이전트 실행을 추적하고 디버깅하기 위한 로깅 시스템을 구현해 보세요. 각 단계별 실행 시간과 중간 결과를 기록해야 합니다.
4. 에이전트의 성능을 실시간으로 모니터링하기 위한 메트릭 수집 시스템을 설계하고 구현해 보세요. Micrometer와 Spring Boot Actuator를 활용하세요.
5. 두 에이전트 버전을 비교하기 위한 A/B 테스트 시스템을 구현해 보세요. 사용자 요청을 무작위로 두 버전에 분배하고 결과를 비교하는 기능을 포함해야 합니다.