# 스트리밍 응답과 비동기 처리

## 개요

실시간 및 장문의 응답이 필요한 현대적인 AI 애플리케이션에서 스트리밍 응답과 비동기 처리는 필수적인 기능입니다. 사용자 경험을 향상시키고, 서버 리소스를 효율적으로 활용하며, 복잡한 AI 워크플로우를 구현하기 위해서는 이러한 기술이 필요합니다.

이 장에서는 Spring AI를 사용하여 스트리밍 응답을 처리하고 비동기 API를 설계하는 방법을 살펴보겠습니다. Spring WebFlux, Project Reactor, 그리고 Spring AI의 스트리밍 기능을 활용하여 확장 가능하고 반응형 AI 애플리케이션을 구축하는 방법을 배우게 됩니다.

## 스트리밍 응답 이해하기

### 스트리밍 응답의 중요성

AI 모델, 특히 LLM은 일반적으로 응답을 토큰 단위로 순차적으로 생성합니다. 전통적인 요청-응답 모델에서는 AI가 전체 응답을 생성한 후에야 사용자에게 응답이 표시됩니다. 이는 다음과 같은 문제를 야기합니다:

1. **긴 대기 시간**: 특히 긴 응답의 경우 사용자는 전체 응답이 생성될 때까지 기다려야 합니다.
2. **사용자 경험 저하**: 응답이 없는 시간이 길어지면 사용자는 시스템이 작동하지 않는다고 생각할 수 있습니다.
3. **리소스 비효율성**: 클라이언트와 서버 간의 연결이 응답이 생성되는 전체 시간 동안 유지되어야 합니다.

스트리밍 응답은 이러한 문제를 해결하고 다음과 같은 이점을 제공합니다:

1. **실시간 피드백**: 사용자는 응답이 생성되는 대로 즉시 볼 수 있습니다.
2. **개선된 사용자 경험**: 사용자는 시스템이 작동 중임을 즉시 확인할 수 있습니다.
3. **조기 중단 가능**: 사용자는 원하는 정보를 얻으면 응답 생성을 중단할 수 있습니다.

### Spring AI의 스트리밍 지원

Spring AI는 두 가지 주요 모델 인터페이스를 통해 스트리밍을 지원합니다:

1. **ChatModel**: 동기식 응답을 위한 `call()` 메서드 제공
2. **StreamingChatModel**: 비동기 스트리밍을 위한 `stream()` 메서드 제공

```java
public interface ChatModel extends Model<Prompt, ChatResponse> {
    default String call(String message) {...}
    ChatResponse call(Prompt prompt);
}

public interface StreamingChatModel extends StreamingModel<Prompt, ChatResponse> {
    default Flux<String> stream(String message) {...}
    Flux<ChatResponse> stream(Prompt prompt);
}
```

## Spring WebFlux와 Project Reactor 기초

Spring AI의 스트리밍 기능은 Project Reactor의 리액티브 타입을 활용합니다.

### Project Reactor 핵심 개념

Project Reactor는 JVM에서 리액티브 프로그래밍을 위한 라이브러리로, Spring WebFlux의 기반이 됩니다. 핵심 타입으로는:

1. **Mono<T>**: 0 또는 1개의 결과를 비동기적으로 생성하는 Publisher
2. **Flux<T>**: 0개 이상의 결과를 비동기적으로 생성하는 Publisher

이러한 타입들은 다음과 같은 특징을 가집니다:

- **선언적**: 실행할 작업 흐름을 선언적으로 정의
- **조합 가능**: 여러 연산자를 결합하여 복잡한 데이터 흐름 생성
- **백프레셔 지원**: 데이터 생성자가 소비자의 처리 능력에 맞춰 데이터 생성 속도 조절

## ChatClient를 사용한 스트리밍 구현

Spring AI의 ChatClient는 스트리밍 응답을 처리하는 fluent API를 제공합니다.

### 의존성 설정

```xml
<!-- Maven -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

### 기본 스트리밍 구현

```java
@Service
public class ChatStreamingService {
    
    private final ChatClient chatClient;
    
    public ChatStreamingService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }
    
    // 동기식 호출
    public String chat(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .call()
            .content();
    }
    
    // 비동기 스트리밍 호출
    public Flux<String> streamChat(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .stream()
            .content();
    }
    
    // ChatResponse 스트리밍
    public Flux<ChatResponse> streamChatResponse(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .stream()
            .chatResponse();
    }
}
```

### 스트리밍 컨트롤러 구현

```java
@RestController
@RequestMapping("/api/chat")
public class ChatStreamingController {
    
    private final ChatStreamingService chatStreamingService;
    
    @Autowired
    public ChatStreamingController(ChatStreamingService chatStreamingService) {
        this.chatStreamingService = chatStreamingService;
    }
    
    // Server-Sent Events를 사용한 스트리밍
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String message) {
        return chatStreamingService.streamChat(message);
    }
    
    // JSON 스트리밍
    @GetMapping(value = "/stream-json", produces = MediaType.APPLICATION_NDJSON_VALUE)
    public Flux<ChatResponse> streamChatJson(@RequestParam String message) {
        return chatStreamingService.streamChatResponse(message);
    }
}
```

### 클라이언트 측 스트리밍 처리

```javascript
// JavaScript EventSource를 사용한 SSE 처리
const eventSource = new EventSource('/api/chat/stream?message=' + encodeURIComponent(userMessage));

let accumulatedContent = '';

eventSource.onmessage = function(event) {
    accumulatedContent += event.data;
    // UI 업데이트
    updateChatUI(accumulatedContent);
};

eventSource.onerror = function(error) {
    console.error('SSE Error:', error);
    eventSource.close();
};
```

## 고급 스트리밍 패턴

### 스트리밍 응답 버퍼링 및 최적화

```java
@Service
public class OptimizedStreamingService {
    
    private final ChatClient chatClient;
    
    public OptimizedStreamingService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
    
    // 시간 기반 버퍼링
    public Flux<String> streamWithTimeBuffering(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .stream()
            .content()
            .bufferTimeout(5, Duration.ofMillis(100))
            .map(chunks -> String.join("", chunks));
    }
    
    // 크기 기반 집계
    public Flux<String> streamWithSizeAggregation(String userMessage, int bufferSize) {
        return chatClient.prompt()
            .user(userMessage)
            .stream()
            .content()
            .windowTimeout(bufferSize, Duration.ofMillis(200))
            .flatMap(window -> window.reduce(String::concat));
    }
}
```

### 스트리밍과 함께 메타데이터 처리

```java
@Service
public class StreamingMetadataService {
    
    private final ChatClient chatClient;
    
    public StreamingMetadataService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
    
    public Flux<StreamingResponse> streamWithMetadata(String userMessage) {
        AtomicInteger tokenCount = new AtomicInteger(0);
        AtomicReference<String> fullContent = new AtomicReference<>("");
        
        return chatClient.prompt()
            .user(userMessage)
            .stream()
            .chatResponse()
            .map(response -> {
                String content = response.getResult().getOutput().getContent();
                fullContent.updateAndGet(existing -> existing + content);
                tokenCount.incrementAndGet();
                
                return new StreamingResponse(
                    content,
                    fullContent.get(),
                    tokenCount.get(),
                    response.getMetadata()
                );
            });
    }
    
    @Data
    @AllArgsConstructor
    public static class StreamingResponse {
        private String delta;
        private String fullContent;
        private int tokensSoFar;
        private ChatResponseMetadata metadata;
    }
}
```

## ChatClient의 비동기 기능

### ChatClient.Builder 설정

```java
@Configuration
public class ChatClientConfig {
    
    @Bean
    public ChatClient chatClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel)
            .defaultSystem("You are a helpful assistant")
            .defaultOptions(ChatOptionsBuilder.builder()
                .withTemperature(0.7)
                .withMaxTokens(1000)
                .build())
            .build();
    }
}
```

### 비동기 응답 처리

```java
@Service
public class AsyncChatService {
    
    private final ChatClient chatClient;
    
    public AsyncChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    // Mono를 반환하는 비동기 호출
    public Mono<String> getChatResponseAsync(String prompt) {
        return Mono.fromCallable(() -> 
            chatClient.prompt()
                .user(prompt)
                .call()
                .content()
        );
    }
    
    // 병렬 처리 예제
    public Flux<ChatResponse> processMultiplePromptsInParallel(List<String> prompts) {
        return Flux.fromIterable(prompts)
            .parallel()
            .runOn(Schedulers.parallel())
            .flatMap(prompt -> 
                Mono.fromCallable(() -> 
                    chatClient.prompt()
                        .user(prompt)
                        .call()
                        .chatResponse()
                )
            )
            .sequential();
    }
}
```

## Advisors와 스트리밍

Spring AI의 Advisor 패턴은 스트리밍과 함께 사용할 수 있습니다.

### 스트리밍과 함께 사용하는 Advisor

```java
@Service
public class StreamingWithAdvisorsService {
    
    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    
    public StreamingWithAdvisorsService(
            ChatClient.Builder builder,
            VectorStore vectorStore) {
        this.vectorStore = vectorStore;
        this.chatClient = builder.build();
    }
    
    public Flux<String> streamWithRAG(String userQuestion) {
        return chatClient.prompt()
            .advisors(QuestionAnswerAdvisor.builder(vectorStore)
                .withSearchRequest(SearchRequest.defaults()
                    .withTopK(5))
                .build())
            .user(userQuestion)
            .stream()
            .content();
    }
    
    public Flux<String> streamWithLogging(String userMessage) {
        return chatClient.prompt()
            .advisors(new SimpleLoggerAdvisor())
            .user(userMessage)
            .stream()
            .content();
    }
}
```

## WebSocket을 사용한 양방향 스트리밍

### WebSocket 핸들러 구현

```java
@Component
public class ChatWebSocketHandler implements WebSocketHandler {
    
    private final ChatClient chatClient;
    private final ObjectMapper objectMapper;
    
    public ChatWebSocketHandler(ChatClient chatClient, ObjectMapper objectMapper) {
        this.chatClient = chatClient;
        this.objectMapper = objectMapper;
    }
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        return session.receive()
            .map(WebSocketMessage::getPayloadAsText)
            .flatMap(message -> processMessage(session, message))
            .then();
    }
    
    private Flux<Void> processMessage(WebSocketSession session, String message) {
        try {
            ChatRequest request = objectMapper.readValue(message, ChatRequest.class);
            
            return chatClient.prompt()
                .user(request.getPrompt())
                .stream()
                .content()
                .map(content -> {
                    ChatResponse response = new ChatResponse(content, false);
                    return objectMapper.writeValueAsString(response);
                })
                .onErrorResume(error -> 
                    Mono.just(objectMapper.writeValueAsString(
                        new ChatResponse(error.getMessage(), true)))
                )
                .map(session::textMessage)
                .flatMap(session::send);
                
        } catch (Exception e) {
            return Flux.error(e);
        }
    }
    
    @Data
    static class ChatRequest {
        private String prompt;
    }
    
    @Data
    @AllArgsConstructor
    static class ChatResponse {
        private String content;
        private boolean error;
    }
}
```

### WebSocket 설정

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    
    private final ChatWebSocketHandler chatWebSocketHandler;
    
    public WebSocketConfig(ChatWebSocketHandler chatWebSocketHandler) {
        this.chatWebSocketHandler = chatWebSocketHandler;
    }
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatWebSocketHandler, "/ws/chat")
            .setAllowedOrigins("*");
    }
}
```

## 에러 처리와 복원력

### 스트리밍 중 에러 처리

```java
@Service
public class ResilientStreamingService {
    
    private final ChatClient chatClient;
    
    public ResilientStreamingService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
    
    public Flux<String> streamWithErrorHandling(String prompt) {
        return chatClient.prompt()
            .user(prompt)
            .stream()
            .content()
            .onErrorResume(error -> {
                log.error("Streaming error occurred", error);
                return Flux.just("An error occurred: " + error.getMessage());
            })
            .timeout(Duration.ofSeconds(30))
            .retry(2);
    }
    
    public Flux<String> streamWithFallback(String prompt) {
        return chatClient.prompt()
            .user(prompt)
            .stream()
            .content()
            .onErrorResume(error -> {
                log.warn("Primary stream failed, using fallback", error);
                return getFallbackResponse(prompt);
            });
    }
    
    private Flux<String> getFallbackResponse(String prompt) {
        // 간단한 폴백 응답 또는 다른 모델 사용
        return Flux.just("I apologize, but I'm unable to process your request at the moment.");
    }
}
```

## 성능 최적화 전략

### 연결 풀링과 타임아웃 설정

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient() {
        ConnectionProvider connectionProvider = ConnectionProvider.builder("custom")
            .maxConnections(100)
            .maxIdleTime(Duration.ofSeconds(20))
            .maxLifeTime(Duration.ofSeconds(60))
            .pendingAcquireTimeout(Duration.ofSeconds(60))
            .evictInBackground(Duration.ofSeconds(120))
            .build();
            
        HttpClient httpClient = HttpClient.create(connectionProvider)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000)
            .responseTimeout(Duration.ofSeconds(60))
            .doOnConnected(conn -> 
                conn.addHandlerLast(new ReadTimeoutHandler(60))
                    .addHandlerLast(new WriteTimeoutHandler(60)));
                    
        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

### 백프레셔 관리

```java
@Service
public class BackpressureManagementService {
    
    private final ChatClient chatClient;
    
    public BackpressureManagementService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
    
    public Flux<String> streamWithBackpressure(String prompt) {
        return chatClient.prompt()
            .user(prompt)
            .stream()
            .content()
            .onBackpressureBuffer(100, 
                dropped -> log.warn("Dropped item due to backpressure: {}", dropped),
                BufferOverflowStrategy.DROP_OLDEST);
    }
    
    public Flux<String> streamWithRateLimiting(String prompt) {
        return chatClient.prompt()
            .user(prompt)
            .stream()
            .content()
            .delayElements(Duration.ofMillis(50)) // Rate limiting
            .limitRate(10); // Request at most 10 items at a time
    }
}
```

## 모니터링과 관찰 가능성

### 스트리밍 메트릭 수집

```java
@Component
public class StreamingMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    
    public StreamingMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public <T> Flux<T> monitorStreaming(Flux<T> stream, String metricName) {
        Counter tokenCounter = meterRegistry.counter(metricName + ".tokens");
        Timer.Sample sample = Timer.start(meterRegistry);
        
        return stream
            .doOnNext(item -> tokenCounter.increment())
            .doOnComplete(() -> 
                sample.stop(meterRegistry.timer(metricName + ".duration")))
            .doOnError(error -> 
                meterRegistry.counter(metricName + ".errors").increment());
    }
}
```

## 실제 사용 사례: 대화형 문서 분석기

```java
@RestController
@RequestMapping("/api/document-analysis")
public class DocumentAnalysisController {
    
    private final ChatClient chatClient;
    private final DocumentProcessor documentProcessor;
    
    public DocumentAnalysisController(
            ChatClient.Builder builder,
            DocumentProcessor documentProcessor) {
        this.chatClient = builder.build();
        this.documentProcessor = documentProcessor;
    }
    
    @PostMapping(value = "/analyze-stream", 
                 consumes = MediaType.MULTIPART_FORM_DATA_VALUE,
                 produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<AnalysisUpdate> analyzeDocument(
            @RequestParam("file") MultipartFile file,
            @RequestParam("query") String query) {
            
        return documentProcessor.processDocument(file)
            .flatMapMany(documentContent -> {
                String prompt = String.format(
                    "Analyze the following document and answer this query: %s\n\nDocument:\n%s",
                    query, documentContent
                );
                
                AtomicInteger progress = new AtomicInteger(0);
                
                return chatClient.prompt()
                    .user(prompt)
                    .stream()
                    .content()
                    .map(chunk -> {
                        int currentProgress = Math.min(95, progress.addAndGet(5));
                        return new AnalysisUpdate(chunk, currentProgress, false);
                    })
                    .concatWith(Mono.just(new AnalysisUpdate("", 100, true)));
            });
    }
    
    @Data
    @AllArgsConstructor
    static class AnalysisUpdate {
        private String content;
        private int progress;
        private boolean complete;
    }
}
```

## 테스트 전략

### 스트리밍 응답 테스트

```java
@SpringBootTest
@AutoConfigureMockMvc
class StreamingControllerTest {
    
    @Autowired
    private WebTestClient webTestClient;
    
    @MockBean
    private ChatClient chatClient;
    
    @Test
    void testStreamingResponse() {
        // Given
        Flux<String> mockStream = Flux.just("Hello", " ", "World")
            .delayElements(Duration.ofMillis(100));
            
        when(chatClient.prompt().user(anyString()).stream().content())
            .thenReturn(mockStream);
            
        // When & Then
        webTestClient.get()
            .uri("/api/chat/stream?message=test")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(MediaType.TEXT_EVENT_STREAM)
            .returnResult(String.class)
            .getResponseBody()
            .as(StepVerifier::create)
            .expectNext("Hello")
            .expectNext(" ")
            .expectNext("World")
            .verifyComplete();
    }
}
```

## 모범 사례

### 1. 리소스 관리
- 적절한 타임아웃 설정
- 연결 풀 크기 최적화
- 백프레셔 전략 구현

### 2. 에러 처리
- 재시도 메커니즘 구현
- 폴백 전략 준비
- 명확한 에러 메시지 제공

### 3. 사용자 경험
- 진행 상황 표시
- 스트림 중단 기능 제공
- 부분 결과 저장 옵션

### 4. 성능 최적화
- 적절한 버퍼링 사용
- 필요시 레이트 제한 적용
- 리소스 사용량 모니터링

## 결론

Spring AI의 스트리밍 응답과 비동기 처리 기능은 현대적인 AI 애플리케이션 구축에 필수적입니다. ChatClient의 fluent API와 Project Reactor의 강력한 기능을 활용하면 반응형이고 확장 가능한 AI 애플리케이션을 구축할 수 있습니다.

주요 내용 요약:

1. **ChatClient 스트리밍 API**: `stream()` 메서드를 통한 간단한 스트리밍 구현
2. **Project Reactor 통합**: Flux와 Mono를 활용한 비동기 처리
3. **다양한 통신 프로토콜**: SSE, WebSocket, NDJSON 등 다양한 스트리밍 방식 지원
4. **에러 처리와 복원력**: 재시도, 타임아웃, 폴백 전략
5. **성능 최적화**: 백프레셔, 버퍼링, 레이트 제한

다음 장에서는 AI 애플리케이션 테스트 전략에 대해 알아보고, 스트리밍 응답을 포함한 다양한 시나리오에 대한 테스트 방법을 살펴보겠습니다.