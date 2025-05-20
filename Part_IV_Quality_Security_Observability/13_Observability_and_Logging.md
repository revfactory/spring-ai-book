# Observability와 로깅

## 개요

AI 애플리케이션은 복잡한 특성으로 인해 전통적인 소프트웨어보다 관측성(Observability)과 로깅이 더욱 중요합니다. Spring AI 기반 애플리케이션에서는 AI 모델 호출, 사용자 입력, 생성된 응답, 오류 상황 등을 체계적으로 추적하고 분석해야 합니다. 이 장에서는 Spring AI 애플리케이션의 관측성과 로깅을 효과적으로 구현하는 방법을 살펴보고, Spring 생태계의 도구를 활용하여 AI 애플리케이션을 모니터링하고 디버깅하는 방법을 알아보겠습니다.

## 관측성의 핵심 개념

### 관측성이란?

관측성(Observability)은 시스템의 내부 상태를 외부에서 관찰할 수 있는 능력을 의미합니다. 관측성이 높은 시스템은 시스템 동작에 대한 충분한 정보를 제공하여, 어떤 문제가 발생했을 때 그 원인을 쉽게 추적하고 해결할 수 있게 합니다.

관측성은 세 가지 핵심 구성 요소로 이루어집니다:

1. **로그(Logs)**: 시스템에서 발생하는 이벤트의 시간순 기록
2. **메트릭(Metrics)**: 시스템 성능과 동작을 수치로 표현한 데이터
3. **추적(Traces)**: 요청이 시스템을 통과하는 경로와 각 단계에서 소요되는 시간 정보

### AI 애플리케이션의 관측성 과제

AI 애플리케이션, 특히 LLM을 사용하는 애플리케이션은 다음과 같은 고유한 관측성 과제가 있습니다:

1. **비결정적 동작**: 동일한 입력에도 다양한 출력이 생성될 수 있어 디버깅이 어려움
2. **블랙박스 모델**: 모델 내부 작동 방식의 불투명성으로 결과 이해가 어려움
3. **외부 API 의존성**: 제3자 서비스 장애나 지연에 취약함
4. **토큰 사용량과 비용**: API 호출 비용 추적의 복잡성
5. **응답 품질 측정**: 생성된 응답의 질적 평가가 어려움
6. **프롬프트 버전 관리**: 프롬프트 변경 추적과 성능 영향 측정의 필요성

이러한 과제를 해결하기 위해 Spring AI 애플리케이션에서 관측성을 체계적으로 구현해야 합니다.

## Spring Boot Actuator를 통한 기본 관측성

Spring Boot Actuator는 Spring 애플리케이션의 모니터링과 관리를 위한 기능을 제공합니다. Spring AI 애플리케이션에서 Actuator를 활용하는 방법을 살펴보겠습니다.

### Actuator 설정

먼저, 필요한 의존성을 추가합니다:

```kotlin
// Gradle (build.gradle.kts)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus") // Prometheus 지원
}
```

```xml
<!-- Maven (pom.xml) -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
</dependencies>
```

다음으로, `application.yml` 파일에 Actuator 설정을 추가합니다:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true
  info:
    env:
      enabled: true
    java:
      enabled: true
    os:
      enabled: true
```

### AI 관련 건강 지표(Health Indicators) 구현

Spring AI 서비스의 건강 상태를 모니터링하기 위한 커스텀 건강 지표를 구현합니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class AiServiceHealthIndicator implements HealthIndicator {

    private final ChatClient chatClient;
    
    public AiServiceHealthIndicator(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    @Override
    public Health health() {
        try {
            // 간단한 프롬프트로 AI 서비스 연결 확인
            Prompt testPrompt = new Prompt(new UserMessage("test"));
            long startTime = System.currentTimeMillis();
            chatClient.call(testPrompt);
            long responseTime = System.currentTimeMillis() - startTime;
            
            return Health.up()
                    .withDetail("status", "AI 서비스 연결 정상")
                    .withDetail("responseTime", responseTime + "ms")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("status", "AI 서비스 연결 실패")
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

이 건강 지표는 `/actuator/health` 엔드포인트를 통해 AI 서비스의 연결 상태를 확인할 수 있게 해줍니다.

## 커스텀 메트릭 수집

Spring AI 애플리케이션의 성능과 동작을 모니터링하기 위한 커스텀 메트릭을 정의하고 수집하는 방법을 알아보겠습니다.

### Micrometer를 사용한 메트릭 정의

Micrometer는 다양한 모니터링 시스템(Prometheus, Datadog 등)을 지원하는 메트릭 파사드(facade)입니다. Spring Boot는 기본적으로 Micrometer를 통합하고 있어 쉽게 커스텀 메트릭을 정의할 수 있습니다.

```java
import io.micrometer.core.instrument.*;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class MetricsAiService {

    private final ChatClient chatClient;
    private final MeterRegistry meterRegistry;
    
    // 메트릭 정의
    private final Counter aiRequestCounter;
    private final Counter aiErrorCounter;
    private final Timer aiResponseTimer;
    private final DistributionSummary tokenUsageSummary;
    
    public MetricsAiService(ChatClient chatClient, MeterRegistry meterRegistry) {
        this.chatClient = chatClient;
        this.meterRegistry = meterRegistry;
        
        // 요청 카운터 설정
        this.aiRequestCounter = Counter.builder("ai.requests")
                .description("Number of AI model requests")
                .register(meterRegistry);
        
        // 오류 카운터 설정
        this.aiErrorCounter = Counter.builder("ai.errors")
                .description("Number of AI model errors")
                .register(meterRegistry);
        
        // 응답 시간 타이머 설정
        this.aiResponseTimer = Timer.builder("ai.response.time")
                .description("AI model response time")
                .register(meterRegistry);
        
        // 토큰 사용량 분포 요약 설정
        this.tokenUsageSummary = DistributionSummary.builder("ai.token.usage")
                .description("AI model token usage")
                .baseUnit("tokens")
                .register(meterRegistry);
    }
    
    public String generateResponse(Prompt prompt) {
        // 요청 카운터 증가
        aiRequestCounter.increment();
        
        try {
            // 응답 시간 측정
            return aiResponseTimer.record(() -> {
                var response = chatClient.call(prompt);
                
                // 토큰 사용량 기록 (OpenAI만 가능, 다른 제공자는 응답에서 이 정보를 제공하지 않을 수 있음)
                var usage = response.getMetadata().get("usage");
                if (usage instanceof Map) {
                    Map<String, Object> usageMap = (Map<String, Object>) usage;
                    int totalTokens = (int) usageMap.getOrDefault("total_tokens", 0);
                    tokenUsageSummary.record(totalTokens);
                    
                    // 프롬프트 및 완성 토큰 개별 기록
                    recordTokenUsage("prompt", (int) usageMap.getOrDefault("prompt_tokens", 0));
                    recordTokenUsage("completion", (int) usageMap.getOrDefault("completion_tokens", 0));
                }
                
                return response.getResult().getOutput().getContent();
            });
        } catch (Exception e) {
            // 오류 카운터 증가
            aiErrorCounter.increment();
            
            // 오류 유형별 카운터 증가
            Tag errorTypeTag = Tag.of("error.type", e.getClass().getSimpleName());
            meterRegistry.counter("ai.errors", Tags.of(errorTypeTag)).increment();
            
            throw e;
        }
    }
    
    private void recordTokenUsage(String type, int tokens) {
        meterRegistry.summary("ai.token.usage.detailed", Tags.of(Tag.of("type", type)))
                .record(tokens);
    }
}
```

### 비즈니스 메트릭 추적

AI 애플리케이션에 중요한 비즈니스 메트릭을 추적하는 방법도 알아보겠습니다:

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Tag;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class BusinessMetricsCollector {

    private final MeterRegistry meterRegistry;
    
    public BusinessMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    // 쿼리 주제별 요청 추적
    public void trackQueryByTopic(String topic) {
        meterRegistry.counter("ai.query.topic", "topic", topic).increment();
    }
    
    // 응답 품질 추적
    public void trackResponseQuality(String quality) {
        // quality: "excellent", "good", "fair", "poor"
        meterRegistry.counter("ai.response.quality", "quality", quality).increment();
    }
    
    // 사용자 피드백 추적
    public void trackUserFeedback(boolean helpful) {
        meterRegistry.counter("ai.user.feedback", "helpful", String.valueOf(helpful)).increment();
    }
    
    // 응답 생성 시간 측정
    public Timer.Sample startResponseGenerationTimer() {
        return Timer.start(meterRegistry);
    }
    
    public void stopResponseGenerationTimer(Timer.Sample sample, String modelType) {
        sample.stop(meterRegistry.timer("ai.response.generation.time", "model", modelType));
    }
    
    // 비용 추적
    public void trackCost(double cost, String modelType) {
        meterRegistry.gauge("ai.cost", Tag.of("model", modelType), cost);
    }
    
    // 검색 정확도 추적 (RAG 시스템용)
    public void trackRetrievalAccuracy(double accuracy) {
        meterRegistry.summary("ai.retrieval.accuracy").record(accuracy);
    }
}
```

## 구조화된 로깅 전략

효과적인 로깅은 AI 애플리케이션 디버깅과 모니터링에 필수적입니다. 구조화된 로깅 전략을 구현하는 방법을 알아보겠습니다.

### 로깅 프레임워크 설정

Spring Boot는 기본적으로 SLF4J와 Logback을 지원합니다. 구조화된 로깅을 위해 Logback 설정을 JSON 형식으로 변경해 보겠습니다:

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- JSON 형식 로깅을 위한 인코더 -->
        </encoder>
    </appender>
    
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- JSON 형식 로깅을 위한 인코더 -->
        </encoder>
    </appender>
    
    <!-- Spring AI 관련 로그 레벨 설정 -->
    <logger name="org.springframework.ai" level="INFO" />
    
    <!-- 애플리케이션 로그 레벨 설정 -->
    <logger name="com.example.aiapp" level="DEBUG" />
    
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

필요한 의존성을 추가합니다:

```kotlin
// Gradle (build.gradle.kts)
dependencies {
    implementation("net.logstash.logback:logstash-logback-encoder:7.2")
}
```

```xml
<!-- Maven (pom.xml) -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.2</version>
</dependency>
```

### AI 상호작용 로깅

AI 모델과의 상호작용을 체계적으로 로깅하는 방법을 살펴보겠습니다:

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.messages.Message;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

@Service
public class LoggingAiService {

    private static final Logger logger = LoggerFactory.getLogger(LoggingAiService.class);
    private final ChatClient chatClient;
    private final ObjectMapper objectMapper;
    
    public LoggingAiService(ChatClient chatClient, ObjectMapper objectMapper) {
        this.chatClient = chatClient;
        this.objectMapper = objectMapper;
    }
    
    public String generateResponse(Prompt prompt, String userId) {
        // 고유 요청 ID 생성
        String requestId = UUID.randomUUID().toString();
        
        try {
            // 요청 로깅
            logRequest(requestId, prompt, userId);
            
            // AI 호출
            long startTime = System.currentTimeMillis();
            var response = chatClient.call(prompt);
            long duration = System.currentTimeMillis() - startTime;
            
            // 응답 로깅
            String responseContent = response.getResult().getOutput().getContent();
            logResponse(requestId, responseContent, duration, response.getMetadata(), userId);
            
            return responseContent;
        } catch (Exception e) {
            // 오류 로깅
            logError(requestId, e, userId);
            throw e;
        }
    }
    
    private void logRequest(String requestId, Prompt prompt, String userId) {
        Map<String, Object> logData = new HashMap<>();
        logData.put("event", "ai_request");
        logData.put("requestId", requestId);
        logData.put("userId", userId);
        logData.put("messageCount", prompt.getMessages().size());
        
        // 메시지 내용 추출 (민감 정보 필터링 필요)
        String messages = prompt.getMessages().stream()
                .map(Message::getContent)
                .collect(Collectors.joining("\n---\n"));
        logData.put("messages", maskSensitiveInfo(messages));
        
        logger.info("AI Request: {}", serializeLogData(logData));
    }
    
    private void logResponse(String requestId, String responseContent, long duration, 
                             Map<String, Object> metadata, String userId) {
        Map<String, Object> logData = new HashMap<>();
        logData.put("event", "ai_response");
        logData.put("requestId", requestId);
        logData.put("userId", userId);
        logData.put("duration_ms", duration);
        
        // 토큰 사용량 추출 (가능한 경우)
        if (metadata.containsKey("usage")) {
            logData.put("tokenUsage", metadata.get("usage"));
        }
        
        // 응답 내용 (민감 정보 필터링 필요)
        logData.put("response", maskSensitiveInfo(responseContent));
        
        logger.info("AI Response: {}", serializeLogData(logData));
    }
    
    private void logError(String requestId, Exception e, String userId) {
        Map<String, Object> logData = new HashMap<>();
        logData.put("event", "ai_error");
        logData.put("requestId", requestId);
        logData.put("userId", userId);
        logData.put("errorType", e.getClass().getSimpleName());
        logData.put("errorMessage", e.getMessage());
        
        logger.error("AI Error: {}", serializeLogData(logData), e);
    }
    
    private String maskSensitiveInfo(String text) {
        // 민감 정보 마스킹 로직 구현
        // 예: 이메일, 전화번호, 신용카드 번호 등
        return text;
    }
    
    private String serializeLogData(Map<String, Object> logData) {
        try {
            return objectMapper.writeValueAsString(logData);
        } catch (Exception e) {
            logger.warn("Failed to serialize log data", e);
            return logData.toString();
        }
    }
}
```

### 프롬프트 및 응답 로깅 주의사항

AI 상호작용을 로깅할 때는 다음 주의사항을 고려해야 합니다:

1. **개인 식별 정보(PII) 마스킹**: 이메일, 전화번호, 주소 등을 마스킹하여 로깅
2. **법규 준수**: GDPR, HIPAA 등 관련 규정 준수
3. **로그 수준 최적화**: 중요 정보만 적절한 로그 수준으로 기록
4. **로그 보관 정책**: 필요한 기간 동안만 로그 보관
5. **로그 접근 제한**: 로그에 대한 접근 권한 제한

민감한 정보를 마스킹하는 유틸리티 클래스 예제:

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class SensitiveDataMasker {

    private static final Pattern EMAIL_PATTERN = 
            Pattern.compile("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}");
    private static final Pattern PHONE_PATTERN = 
            Pattern.compile("(\\+\\d{1,3}[- ]?)?\\d{3}[- ]?\\d{3,4}[- ]?\\d{4}");
    private static final Pattern CREDIT_CARD_PATTERN = 
            Pattern.compile("\\d{4}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}");
    private static final Pattern SSN_PATTERN = 
            Pattern.compile("\\d{3}[-]\\d{2}[-]\\d{4}");

    public static String maskSensitiveData(String text) {
        if (text == null || text.isEmpty()) {
            return text;
        }
        
        String masked = text;
        
        // 이메일 마스킹
        Matcher emailMatcher = EMAIL_PATTERN.matcher(masked);
        while (emailMatcher.find()) {
            String email = emailMatcher.group();
            String maskedEmail = maskEmail(email);
            masked = masked.replace(email, maskedEmail);
        }
        
        // 전화번호 마스킹
        Matcher phoneMatcher = PHONE_PATTERN.matcher(masked);
        while (phoneMatcher.find()) {
            String phone = phoneMatcher.group();
            masked = masked.replace(phone, "PHONE-***-***-****");
        }
        
        // 신용카드 번호 마스킹
        Matcher ccMatcher = CREDIT_CARD_PATTERN.matcher(masked);
        while (ccMatcher.find()) {
            String cc = ccMatcher.group();
            masked = masked.replace(cc, "CC-****-****-****-****");
        }
        
        // 주민등록번호/SSN 마스킹
        Matcher ssnMatcher = SSN_PATTERN.matcher(masked);
        while (ssnMatcher.find()) {
            String ssn = ssnMatcher.group();
            masked = masked.replace(ssn, "SSN-***-**-****");
        }
        
        return masked;
    }
    
    private static String maskEmail(String email) {
        int atIndex = email.indexOf('@');
        if (atIndex <= 1) {
            return "***@" + email.substring(atIndex + 1);
        }
        
        String username = email.substring(0, atIndex);
        String domain = email.substring(atIndex);
        
        int visibleChars = Math.min(3, username.length());
        return username.substring(0, visibleChars) + "***" + domain;
    }
}
```

## 분산 추적 구현

분산 시스템에서 요청 흐름을 추적하기 위해 Spring Cloud Sleuth와 Zipkin을 활용한 분산 추적을 구현하는 방법을 알아보겠습니다.

### Spring Cloud Sleuth 설정

먼저, 필요한 의존성을 추가합니다:

```kotlin
// Gradle (build.gradle.kts)
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-sleuth")
    implementation("org.springframework.cloud:spring-cloud-sleuth-zipkin")
}
```

```xml
<!-- Maven (pom.xml) -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-sleuth-zipkin</artifactId>
    </dependency>
</dependencies>
```

다음으로, `application.yml` 파일에 Sleuth와 Zipkin 설정을 추가합니다:

```yaml
spring:
  application:
    name: ai-application
  sleuth:
    sampler:
      probability: 1.0  # 개발 환경에서는 모든 트레이스 샘플링
  zipkin:
    base-url: http://localhost:9411
```

### AI 서비스 추적

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