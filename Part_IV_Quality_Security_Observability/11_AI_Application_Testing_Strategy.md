# AI 애플리케이션 테스트 전략

## 개요

AI 애플리케이션 테스팅은 전통적인 소프트웨어 테스팅과는 다른 접근 방식이 필요합니다. AI 모델의 비결정적 특성, 외부 API 의존성, 입력과 출력의 다양성 등이 테스트를 복잡하게 만듭니다. 이 장에서는 Spring AI를 사용하는 애플리케이션의 효과적인 테스트 전략과 방법론을 살펴보고, 특히 Spring AI의 최신 테스트 기능들을 활용하는 방법에 대해 자세히 알아보겠습니다.

## AI 애플리케이션 테스트의 과제

### 전통적인 테스트와의 차이점

AI 애플리케이션 테스트는 다음과 같은 이유로 전통적인 소프트웨어 테스트와 다릅니다:

1. **비결정성**: 동일한 입력에도 AI 모델은 다양한 출력을 생성할 수 있습니다.
2. **외부 API 의존성**: 대부분의 AI 기능은 OpenAI, Anthropic 등의 외부 서비스에 의존합니다.
3. **비용**: 실제 API 호출은 비용이 발생하므로 테스트 비용이 상당히 증가할 수 있습니다.
4. **지연 시간**: API 호출은 응답 시간이 길어 테스트 속도가 느려질 수 있습니다.
5. **서비스 가용성**: 외부 서비스의 중단이나 변경에 취약합니다.
6. **복잡한 응답 구조**: 응답이 자유 형식 텍스트이거나 복잡한 구조를 가질 수 있습니다.

### 테스트 전략의 주요 목표

AI 애플리케이션 테스트 전략은 다음 목표를 달성해야 합니다:

1. **재현 가능성**: 테스트 결과가 예측 가능하고 일관되어야 합니다.
2. **독립성**: 외부 서비스에 의존하지 않고 테스트를 실행할 수 있어야 합니다.
3. **속도**: 테스트가 빠르게 실행되어야 개발 생산성이 향상됩니다.
4. **비용 효율성**: 불필요한 API 호출 비용을 최소화해야 합니다.
5. **포괄성**: 애플리케이션의 모든 중요 경로를 테스트해야 합니다.
6. **유지 관리성**: 테스트 코드가 쉽게 유지 관리되어야 합니다.

## Spring AI의 평가 테스팅

### Evaluator 인터페이스

Spring AI는 생성된 콘텐츠를 평가하여 AI 모델이 환각(hallucination)된 응답을 생성하지 않았는지 확인하는 `Evaluator` 인터페이스를 제공합니다:

```java
@FunctionalInterface
public interface Evaluator {
    EvaluationResponse evaluate(EvaluationRequest evaluationRequest);
}
```

평가 입력인 `EvaluationRequest`는 다음과 같이 정의됩니다:

```java
public class EvaluationRequest {
    private final String userText;
    private final List<Content> dataList;
    private final String responseContent;
    
    public EvaluationRequest(String userText, List<Content> dataList, String responseContent) {
        this.userText = userText;
        this.dataList = dataList;
        this.responseContent = responseContent;
    }
    // ... getters
}
```

- `userText`: 사용자의 원본 입력
- `dataList`: RAG 등에서 가져온 컨텍스트 데이터
- `responseContent`: AI 모델의 응답 내용

### RelevancyEvaluator

`RelevancyEvaluator`는 AI가 생성한 응답이 제공된 컨텍스트와 관련이 있는지 평가하는 `Evaluator` 구현체입니다:

```java
@Test
void evaluateRelevancy() {
    String question = "Spring AI의 주요 기능은 무엇인가요?";
    
    RetrievalAugmentationAdvisor ragAdvisor = RetrievalAugmentationAdvisor.builder()
        .documentRetriever(VectorStoreDocumentRetriever.builder()
            .vectorStore(pgVectorStore)
            .build())
        .build();
    
    ChatResponse chatResponse = ChatClient.builder(chatModel).build()
        .prompt(question)
        .advisors(ragAdvisor)
        .call()
        .chatResponse();
    
    EvaluationRequest evaluationRequest = new EvaluationRequest(
        // 원본 사용자 질문
        question,
        // RAG 흐름에서 검색된 컨텍스트
        chatResponse.getMetadata().get(RetrievalAugmentationAdvisor.DOCUMENT_CONTEXT),
        // AI 모델의 응답
        chatResponse.getResult().getOutput().getText()
    );
    
    RelevancyEvaluator evaluator = new RelevancyEvaluator(ChatClient.builder(chatModel));
    
    EvaluationResponse evaluationResponse = evaluator.evaluate(evaluationRequest);
    
    assertThat(evaluationResponse.isPass()).isTrue();
}
```

### FactCheckingEvaluator

`FactCheckingEvaluator`는 AI가 생성한 응답의 사실 정확성을 평가합니다. 이는 AI 출력의 환각을 감지하고 줄이는 데 도움이 됩니다:

```java
@Test
void testFactChecking() {
    // Ollama API 설정
    OllamaApi ollamaApi = new OllamaApi("http://localhost:11434");
    
    ChatModel chatModel = new OllamaChatModel(ollamaApi,
        OllamaOptions.builder()
            .model(BESPOKE_MINICHECK)
            .numPredict(2)
            .temperature(0.0d)
            .build());
    
    // FactCheckingEvaluator 생성
    var factCheckingEvaluator = new FactCheckingEvaluator(ChatClient.builder(chatModel));
    
    // 예제 컨텍스트와 주장
    String context = "Spring AI는 AI 애플리케이션 개발을 위한 프레임워크입니다.";
    String claim = "Spring AI는 데이터베이스 관리 시스템입니다.";
    
    // EvaluationRequest 생성
    EvaluationRequest evaluationRequest = new EvaluationRequest(
        context, 
        Collections.emptyList(), 
        claim
    );
    
    // 평가 수행
    EvaluationResponse evaluationResponse = factCheckingEvaluator.evaluate(evaluationRequest);
    
    assertFalse(evaluationResponse.isPass(), "주장은 컨텍스트에 의해 지원되지 않아야 합니다");
}
```

## Testcontainers 통합

Spring AI는 Testcontainers를 통해 모델 서비스나 벡터 스토어에 대한 연결을 자동으로 설정하는 Spring Boot 자동 구성을 제공합니다.

### 의존성 추가

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-spring-boot-testcontainers</artifactId>
</dependency>
```

```gradle
// Gradle
dependencies {
    implementation 'org.springframework.ai:spring-ai-spring-boot-testcontainers'
}
```

### 지원되는 서비스 연결

Spring AI의 testcontainers 모듈은 다음 서비스 연결을 지원합니다:

| Connection Details | Container Type |
|-------------------|----------------|
| `ChromaConnectionDetails` | `ChromaDBContainer` |
| `MilvusServiceClientConnectionDetails` | `MilvusContainer` |
| `OllamaConnectionDetails` | `OllamaContainer` |
| `QdrantConnectionDetails` | `QdrantContainer` |
| `WeaviateConnectionDetails` | `WeaviateContainer` |

### Testcontainers를 사용한 통합 테스트

```java
@SpringBootTest
@Testcontainers
public class VectorStoreIntegrationTest {
    
    @Container
    static ChromaDBContainer chromaContainer = new ChromaDBContainer("chromadb/chroma:latest");
    
    @Autowired
    private VectorStore vectorStore;
    
    @Test
    void testVectorStoreOperations() {
        // 문서 준비
        List<Document> documents = List.of(
            new Document("Spring AI는 AI 애플리케이션 개발을 단순화합니다.", Map.of("type", "description")),
            new Document("Spring AI는 다양한 AI 모델을 지원합니다.", Map.of("type", "feature"))
        );
        
        // 문서 저장
        vectorStore.add(documents);
        
        // 유사성 검색
        List<Document> results = vectorStore.similaritySearch(
            SearchRequest.query("Spring AI 기능").withTopK(2)
        );
        
        assertThat(results).hasSize(2);
        assertThat(results.get(0).getContent()).contains("Spring AI");
    }
}
```

## ChatClient를 사용한 단위 테스트

### Mock ChatModel 사용

```java
@SpringBootTest
class ChatServiceTest {
    
    @MockBean
    private ChatModel chatModel;
    
    @Autowired
    private ChatService chatService;
    
    @Test
    void testChatResponse() {
        // Given
        String userInput = "Spring AI에 대해 설명해주세요";
        String expectedResponse = "Spring AI는 AI 애플리케이션 개발을 위한 프레임워크입니다.";
        
        ChatResponse mockResponse = new ChatResponse(List.of(
            new Generation(new AssistantMessage(expectedResponse))
        ));
        
        when(chatModel.call(any(Prompt.class))).thenReturn(mockResponse);
        
        // When
        String actualResponse = chatService.chat(userInput);
        
        // Then
        assertThat(actualResponse).isEqualTo(expectedResponse);
        verify(chatModel).call(argThat(prompt -> 
            prompt.getInstructions().get(0).getText().equals(userInput)
        ));
    }
}
```

## 스트리밍 응답 테스트

### WebTestClient를 사용한 스트리밍 테스트

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class StreamingControllerTest {
    
    @Autowired
    private WebTestClient webTestClient;
    
    @MockBean
    private ChatModel chatModel;
    
    @Test
    void testStreamingResponse() {
        // Given
        Flux<ChatResponse> mockStream = Flux.just(
            new ChatResponse(List.of(new Generation(new AssistantMessage("안녕")))),
            new ChatResponse(List.of(new Generation(new AssistantMessage("하세요")))),
            new ChatResponse(List.of(new Generation(new AssistantMessage("!"))))
        ).delayElements(Duration.ofMillis(100));
        
        when(chatModel.stream(any(Prompt.class))).thenReturn(mockStream);
        
        // When & Then
        webTestClient.get()
            .uri("/api/chat/stream?message=안녕하세요")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(MediaType.TEXT_EVENT_STREAM)
            .returnResult(String.class)
            .getResponseBody()
            .as(StepVerifier::create)
            .expectNext("안녕")
            .expectNext("하세요")
            .expectNext("!")
            .verifyComplete();
    }
}
```

## 함수 호출(Function Calling) 테스트

### Function Calling Mock 테스트

```java
@Test
void testFunctionCalling() {
    // Function 정의
    Function<WeatherRequest, WeatherResponse> weatherFunction = request -> 
        new WeatherResponse(request.location(), "맑음", 25.5);
    
    // Mock 설정
    ChatOptions options = OpenAiChatOptions.builder()
        .withFunction("getWeather", "현재 날씨 정보 조회", weatherFunction)
        .build();
    
    ChatResponse mockResponse = ChatResponse.builder()
        .withGenerations(List.of(Generation.builder()
            .withAssistantMessage(AssistantMessage.builder()
                .withToolCalls(List.of(new ToolCall(
                    "call_123",
                    "function",
                    "getWeather",
                    """
                    {
                        "location": "서울"
                    }
                    """
                )))
                .build())
            .build()))
        .build();
    
    when(chatModel.call(any(Prompt.class))).thenReturn(mockResponse);
    
    // 테스트 실행
    String response = chatService.getWeatherInfo("서울의 날씨는 어때요?");
    
    // 검증
    assertThat(response).contains("서울", "맑음", "25.5");
}
```

## Advisor 테스트

### SimpleLoggerAdvisor를 사용한 디버깅

```java
@Test
void testAdvisorLogging() {
    // 로깅 레벨 설정
    LogbackTestAppender testAppender = new LogbackTestAppender();
    testAppender.start();
    
    Logger logger = (Logger) LoggerFactory.getLogger(SimpleLoggerAdvisor.class);
    logger.addAppender(testAppender);
    
    // ChatClient with SimpleLoggerAdvisor
    ChatClient chatClient = ChatClient.builder(chatModel)
        .build();
    
    String response = chatClient.prompt()
        .advisors(new SimpleLoggerAdvisor())
        .user("테스트 메시지")
        .call()
        .content();
    
    // 로그 검증
    List<ILoggingEvent> logs = testAppender.getEvents();
    assertThat(logs).anySatisfy(event -> {
        assertThat(event.getMessage()).contains("BEFORE:");
        assertThat(event.getMessage()).contains("테스트 메시지");
    });
    assertThat(logs).anySatisfy(event -> {
        assertThat(event.getMessage()).contains("AFTER:");
    });
}
```

## 성능 테스트

### 응답 시간 측정

```java
@Test
void testResponseTime() {
    // 테스트 데이터 준비
    List<String> prompts = List.of(
        "짧은 응답",
        "중간 길이의 응답을 생성해주세요",
        "매우 상세하고 긴 응답을 작성해주세요"
    );
    
    Map<String, Long> responseTimes = new HashMap<>();
    
    for (String prompt : prompts) {
        long startTime = System.currentTimeMillis();
        
        chatClient.prompt()
            .user(prompt)
            .call()
            .content();
        
        long endTime = System.currentTimeMillis();
        responseTimes.put(prompt, endTime - startTime);
    }
    
    // 성능 지표 검증
    responseTimes.forEach((prompt, time) -> {
        assertThat(time).isLessThan(5000); // 5초 이내 응답
        log.info("프롬프트: '{}' - 응답 시간: {}ms", prompt, time);
    });
}
```

## RAG 시스템 테스트

### 통합 RAG 테스트

```java
@SpringBootTest
@Testcontainers
class RagSystemTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("pgvector/pgvector:pg16");
    
    @Autowired
    private VectorStore vectorStore;
    
    @Autowired
    private ChatModel chatModel;
    
    @Test
    void testCompleteRagFlow() {
        // 1. 문서 임베딩 및 저장
        List<Document> documents = List.of(
            new Document("Spring AI는 2024년에 출시된 AI 프레임워크입니다."),
            new Document("Spring AI는 OpenAI, Anthropic, Ollama 등을 지원합니다."),
            new Document("Spring AI는 RAG 패턴을 쉽게 구현할 수 있게 해줍니다.")
        );
        
        vectorStore.add(documents);
        
        // 2. RAG Advisor 설정
        QuestionAnswerAdvisor ragAdvisor = QuestionAnswerAdvisor.builder()
            .vectorStore(vectorStore)
            .searchRequest(SearchRequest.defaults().withTopK(3))
            .build();
        
        // 3. 질문 및 응답 생성
        ChatClient chatClient = ChatClient.builder(chatModel).build();
        
        ChatResponse response = chatClient.prompt()
            .advisors(ragAdvisor)
            .user("Spring AI는 언제 출시되었나요?")
            .call()
            .chatResponse();
        
        // 4. 응답 검증
        String answer = response.getResult().getOutput().getContent();
        assertThat(answer).contains("2024년");
        
        // 5. 관련성 평가
        RelevancyEvaluator evaluator = new RelevancyEvaluator(ChatClient.builder(chatModel));
        
        EvaluationRequest evalRequest = new EvaluationRequest(
            "Spring AI는 언제 출시되었나요?",
            response.getMetadata().get(QuestionAnswerAdvisor.RETRIEVED_DOCUMENTS),
            answer
        );
        
        EvaluationResponse evalResponse = evaluator.evaluate(evalRequest);
        assertThat(evalResponse.isPass()).isTrue();
    }
}
```

## 테스트 모범 사례

### 1. 계층별 테스트 전략

```java
// 단위 테스트 - 서비스 레이어
@ExtendWith(MockitoExtension.class)
class ChatServiceUnitTest {
    @Mock
    private ChatModel chatModel;
    
    @InjectMocks
    private ChatService chatService;
    
    // 비즈니스 로직 테스트
}

// 통합 테스트 - API 레이어
@WebMvcTest(ChatController.class)
class ChatControllerTest {
    @MockBean
    private ChatService chatService;
    
    // REST API 테스트
}

// E2E 테스트 - 전체 플로우
@SpringBootTest
@Testcontainers
class ChatApplicationE2ETest {
    // 실제 컨테이너를 사용한 테스트
}
```

### 2. 테스트 데이터 관리

```java
@Component
public class TestDataFactory {
    
    public ChatResponse createChatResponse(String content) {
        return new ChatResponse(List.of(
            new Generation(new AssistantMessage(content))
        ));
    }
    
    public List<Document> createDocuments(String... contents) {
        return Arrays.stream(contents)
            .map(content -> new Document(content, Map.of("source", "test")))
            .collect(Collectors.toList());
    }
    
    public EvaluationRequest createEvaluationRequest(
            String question, 
            List<Content> context, 
            String response) {
        return new EvaluationRequest(question, context, response);
    }
}
```

### 3. 비동기 테스트 패턴

```java
@Test
void testAsyncChatProcessing() {
    // Given
    CompletableFuture<String> futureResponse = new CompletableFuture<>();
    
    // When
    chatService.processAsync("비동기 질문")
        .thenAccept(futureResponse::complete);
    
    // Then
    assertThat(futureResponse).succeedsWithin(Duration.ofSeconds(10))
        .satisfies(response -> {
            assertThat(response).isNotEmpty();
            assertThat(response).doesNotContain("error");
        });
}
```

## 결론

이 장에서는 Spring AI 애플리케이션을 효과적으로 테스트하기 위한 다양한 전략과 도구를 살펴보았습니다. AI 애플리케이션 테스트의 특수한 과제를 이해하고, Spring AI가 제공하는 평가 도구와 Testcontainers 통합을 활용하는 방법을 배웠습니다.

테스트 전략 요약:
1. **평가 기반 테스팅**: RelevancyEvaluator와 FactCheckingEvaluator를 사용한 응답 품질 검증
2. **Testcontainers 통합**: 실제 서비스 환경과 유사한 테스트 환경 구성
3. **계층별 테스트**: 단위, 통합, E2E 테스트의 적절한 조합
4. **스트리밍 테스트**: 비동기 응답에 대한 효과적인 검증
5. **성능 모니터링**: 응답 시간 및 리소스 사용량 추적

이러한 테스트 전략을 활용하면 Spring AI 애플리케이션의 안정성, 신뢰성, 그리고 품질을 크게 향상시킬 수 있습니다. 다음 장에서는 AI 애플리케이션의 데이터 보안과 비용 관리에 대해 알아보겠습니다.