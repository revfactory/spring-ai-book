# 15. Spring AI를 활용한 에이전트 구축

## 15.1 Spring AI 1.0.0 GA 에이전트 구축 개요

Spring AI 1.0.0 GA는 현대적이고 실용적인 AI 에이전트 구축을 위한 포괄적인 도구와 패턴을 제공합니다. ChatClient API, Advisors, Tools, MCP(Model Context Protocol) 통합을 통해 엔터프라이즈급 에이전트 시스템을 손쉽게 구축할 수 있습니다.

### 15.1.1 Spring AI 에이전트 구축의 핵심 구성 요소

Spring AI 1.0.0 GA의 에이전트 구축은 다음 핵심 요소들로 구성됩니다:

- **ChatClient**: 통합된 대화형 AI 인터페이스와 빌더 패턴
- **Advisors**: 재사용 가능한 AI 상호작용 패턴 캡슐화
- **Tools**: 외부 시스템과의 통합을 위한 도구 호출 메커니즘
- **Memory**: 대화 맥락과 상태 관리를 위한 메모리 시스템
- **Observability**: 내장된 메트릭, 추적, 로깅 기능
- **MCP Integration**: 표준화된 모델-컨텍스트 프로토콜 지원

이러한 구성 요소들이 유기적으로 연결되어 강력하고 확장 가능한 AI 에이전트를 구현합니다.

### 15.1.2 Spring AI 1.0.0 GA의 새로운 접근법

Spring AI 1.0.0 GA는 전통적인 Agent 인터페이스 대신 **ChatClient 중심의 컴포지션 패턴**을 채택했습니다:

```java
// ChatClient 빌더 패턴을 통한 에이전트 구성
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        MessageChatMemoryAdvisor.builder(chatMemory).build(),
        QuestionAnswerAdvisor.builder(vectorStore).build()
    )
    .defaultTools("weatherTool", "calculatorTool")
    .build();
```

**주요 구축 패턴:**

- **Advisor-based Architecture**: 재사용 가능한 AI 패턴을 Advisor로 캡슐화
- **Tool Integration**: 선언적 도구 정의와 동적 해결
- **Memory Management**: 다양한 메모리 전략 지원 (InMemory, Vector, Database)
- **Observability First**: 모든 구성 요소에 내장된 관측성

### 15.1.3 Spring AI 1.0.0 GA 의존성 설정

Spring AI 1.0.0 GA 에이전트 구축을 위한 최신 의존성 설정:

```xml
<dependencies>
    <!-- Core Spring AI Starter -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
    
    <!-- 벡터 스토어 (RAG용) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-vectorstore-pgvector</artifactId>
    </dependency>
    
    <!-- MCP 클라이언트 지원 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-mcp-client</artifactId>
    </dependency>
    
    <!-- 관측성 지원 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
    <!-- Web 지원 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
</dependencies>
```

```properties
# application.yml 기본 설정
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4
          temperature: 0.7
    
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,traces
```

## 15.2 ChatClient를 활용한 에이전트 구현

### 15.2.1 ChatClient 기반 에이전트 설정

Spring AI 1.0.0 GA에서는 ChatClient를 중심으로 한 에이전트 구성이 표준입니다:

```java
@Configuration
public class AgentConfig {

    @Bean
    public ChatClient simpleChatClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel)
            .defaultSystem("당신은 도움을 주는 AI 어시스턴트입니다. 사용자의 질문에 명확하고 정확하게 답변해 주세요.")
            .build();
    }
    
    @Bean
    public ChatClient advancedAgent(ChatModel chatModel, 
                                   InMemoryChatMemory chatMemory,
                                   VectorStore vectorStore) {
        return ChatClient.builder(chatModel)
            .defaultSystem("당신은 고급 기능을 갖춘 AI 어시스턴트입니다.")
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build(),
                QuestionAnswerAdvisor.builder(vectorStore).build()
            )
            .build();
    }
}
```

### 15.2.2 ChatClient 에이전트 컨트롤러 구현

ChatClient를 활용한 현대적인 REST API 컨트롤러 구현:

```java
@RestController
@RequestMapping("/api/agent")
public class ChatClientAgentController {

    private final ChatClient chatClient;
    
    public ChatClientAgentController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    @PostMapping("/chat")
    public ResponseEntity<Map<String, String>> chat(@RequestBody Map<String, String> request) {
        String userInput = request.get("message");
        
        String response = chatClient.prompt()
            .user(userInput)
            .call()
            .content();
        
        Map<String, String> responseBody = new HashMap<>();
        responseBody.put("response", response);
        
        return ResponseEntity.ok(responseBody);
    }
    
    @PostMapping("/stream")
    public Flux<ServerSentEvent<String>> streamChat(@RequestBody Map<String, String> request) {
        String userInput = request.get("message");
        
        return chatClient.prompt()
            .user(userInput)
            .stream()
            .content()
            .map(chunk -> ServerSentEvent.builder(chunk).build());
    }
    
    @PostMapping("/chat-with-memory")
    public ResponseEntity<Map<String, String>> chatWithMemory(
            @RequestHeader("User-ID") String userId,
            @RequestBody Map<String, String> request) {
        
        String userInput = request.get("message");
        
        String response = chatClient.prompt()
            .user(userInput)
            .advisors(advisor -> advisor.param(ChatMemory.CONVERSATION_ID, userId))
            .call()
            .content();
        
        Map<String, String> responseBody = new HashMap<>();
        responseBody.put("response", response);
        
        return ResponseEntity.ok(responseBody);
    }
}
```

### 15.2.3 ChatClient 에이전트 테스트

ChatClient를 활용한 테스트 구현:

```java
@SpringBootTest
class ChatClientAgentTest {

    @Autowired
    private ChatClient chatClient;
    
    @Test
    void testSimpleChatClient() {
        // Given
        String userInput = "Spring AI 1.0.0 GA의 주요 특징은 무엇인가요?";
            
        // When
        String response = chatClient.prompt()
            .user(userInput)
            .call()
            .content();
        
        // Then
        assertThat(response).isNotEmpty();
        assertThat(response).containsIgnoringCase("Spring AI");
        System.out.println("Agent Response: " + response);
    }
    
    @Test
    void testStreamingResponse() {
        // Given
        String userInput = "AI 에이전트의 장점을 설명해 주세요.";
        
        // When
        Flux<String> responseStream = chatClient.prompt()
            .user(userInput)
            .stream()
            .content();
        
        // Then
        StepVerifier.create(responseStream)
            .expectNextMatches(chunk -> !chunk.isEmpty())
            .thenCancel()
            .verify();
    }
    
    @Test
    void testChatClientWithAdvisors() {
        // Given
        String userInput = "이전 대화 내용을 기억하나요?";
        String conversationId = "test-conversation-123";
        
        // When
        String response = chatClient.prompt()
            .user(userInput)
            .advisors(advisor -> advisor.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
        
        // Then
        assertThat(response).isNotEmpty();
        System.out.println("Memory-enabled response: " + response);
    }
}
```

## 15.3 Spring AI 1.0.0 GA 도구 사용 에이전트 개발

### 15.3.1 Spring AI 1.0.0 GA 도구(Tool) 시스템 이해하기

Spring AI 1.0.0 GA의 도구 시스템은 `@Tool` 어노테이션과 `ToolCallback` 인터페이스를 중심으로 합니다:

```java
// @Tool 어노테이션을 사용한 선언적 도구 정의
class WeatherTools {
    
    @Tool(description = "현재 날씨 정보를 제공합니다. 도시 이름을 입력하면 해당 도시의 날씨를 알려줍니다.")
    public String getCurrentWeather(@ToolParam(description = "날씨 정보를 조회할 도시 이름") String city) {
        // 실제 날씨 API 호출 로직
        return "서울의 현재 날씨: 맑음, 기온 22°C";
    }
    
    @Tool(description = "날씨 예보 정보를 제공합니다.")
    public String getWeatherForecast(
            @ToolParam(description = "예보를 조회할 도시") String city,
            @ToolParam(description = "예보 일수 (1-7일)", required = false) Integer days) {
        int forecastDays = days != null ? days : 3;
        return String.format("%s의 %d일 예보: 대체로 맑음", city, forecastDays);
    }
}
```

**주요 도구 시스템 특징:**
- **@Tool 어노테이션**: 메서드를 도구로 자동 등록
- **@ToolParam**: 매개변수에 대한 상세 정보 제공
- **자동 JSON 스키마 생성**: 매개변수 타입에서 자동으로 스키마 생성
- **동적 도구 해결**: 빈 이름으로 도구를 동적으로 해결

### 15.3.2 @Tool 어노테이션을 활용한 도구 구현

다양한 도구를 @Tool 어노테이션으로 구현하는 방법:

```java
@Component
public class BusinessTools {

    private final WeatherService weatherService;
    private final DatabaseService databaseService;
    
    public BusinessTools(WeatherService weatherService, DatabaseService databaseService) {
        this.weatherService = weatherService;
        this.databaseService = databaseService;
    }
    
    @Tool(description = "현재 날씨 정보를 조회합니다")
    public WeatherInfo getCurrentWeather(
            @ToolParam(description = "날씨를 조회할 도시 이름") String city) {
        return weatherService.getWeatherForCity(city);
    }
    
    @Tool(description = "고객 정보를 조회합니다")
    public CustomerInfo getCustomerInfo(
            @ToolParam(description = "고객 ID") Long customerId) {
        return databaseService.findCustomerById(customerId);
    }
    
    @Tool(description = "이메일을 전송합니다")
    public String sendEmail(
            @ToolParam(description = "수신자 이메일") String to,
            @ToolParam(description = "이메일 제목") String subject,
            @ToolParam(description = "이메일 본문") String body) {
        // 이메일 전송 로직
        return "이메일이 성공적으로 전송되었습니다.";
    }
    
    @Tool(description = "복잡한 데이터 분석을 수행합니다", returnDirect = true)
    public AnalysisResult performAnalysis(
            @ToolParam(description = "분석할 데이터 ID") String dataId,
            @ToolParam(description = "분석 유형", required = false) String analysisType) {
        // 복잡한 분석 로직
        return new AnalysisResult(dataId, analysisType);
    }
}
```

### 15.3.3 ChatClient에 도구를 통합하는 방법

ChatClient에 도구를 통합하는 다양한 방법:

```java
@Configuration
public class ToolAwareChatClientConfig {

    @Bean
    public ChatClient toolEnabledChatClient(ChatModel chatModel, BusinessTools businessTools) {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                당신은 도구를 활용할 수 있는 AI 어시스턴트입니다.
                사용자의 요청을 이해하고 적절한 도구를 선택하여 정확한 정보를 제공하세요.
                도구 사용 시 필요한 매개변수를 정확히 제공하고, 결과를 이해하기 쉽게 설명하세요.
                """)
            .defaultTools(businessTools) // 기본 도구 설정
            .build();
    }
    
    @Bean
    public ChatClient dynamicToolChatClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel)
            .defaultSystem("동적 도구 해결을 지원하는 AI 어시스턴트입니다.")
            .defaultTools("getCurrentWeather", "getCustomerInfo", "sendEmail") // 도구 이름으로 동적 해결
            .build();
    }
}

@Service
public class ToolEnabledAgentService {
    
    private final ChatClient chatClient;
    
    public ToolEnabledAgentService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String processRequest(String userInput) {
        return chatClient.prompt()
            .user(userInput)
            .call()
            .content();
    }
    
    public String processWithSpecificTools(String userInput, Object... tools) {
        return chatClient.prompt()
            .user(userInput)
            .tools(tools) // 런타임에 특정 도구 추가
            .call()
            .content();
    }
    
    public Flux<String> processStreaming(String userInput) {
        return chatClient.prompt()
            .user(userInput)
            .stream()
            .content();
    }
}
```

### 15.3.4 도구 선택 및 사용 흐름

도구를 활용하는 에이전트의 일반적인 작업 흐름:

1. 사용자 입력 분석: 에이전트는 사용자의 요청을 분석합니다.
2. 도구 필요성 판단: 요청을 처리하기 위해 도구가 필요한지 판단합니다.
3. 도구 선택: 적절한 도구를 선택합니다.
4. 매개변수 추출: 사용자 입력에서 도구 실행에 필요한 매개변수를 추출합니다.
5. 도구 실행: 준비된 매개변수로 도구를 실행합니다.
6. 결과 처리: 도구 실행 결과를 처리하고 사용자에게 적절한 응답을 생성합니다.

## 15.4 대화형 에이전트와 메모리 관리

### 15.4.1 대화 맥락 유지하기

대화 맥락을 유지하는 에이전트 구현:

```java
@Configuration
public class ConversationalAgentConfig {

    @Bean
    public MemoryStore memoryStore() {
        return new InMemoryMemoryStore();
    }
    
    @Bean
    public Agent conversationalAgent(ChatClient chatClient, MemoryStore memoryStore) {
        String systemPrompt = "당신은 사용자와의 대화를 기억하고 이를 바탕으로 맥락을 이해하는 AI 어시스턴트입니다.";
        
        return AgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .memoryStore(memoryStore)
            .build();
    }
}
```

### 15.4.2 메모리 저장소 구현

다양한 메모리 저장소 구현 방식:

1. **InMemoryMemoryStore**: 기본 인메모리 구현체
   ```java
   MemoryStore memoryStore = new InMemoryMemoryStore();
   ```

2. **RedisMemoryStore**: Redis를 활용한 분산 메모리 저장소
   ```java
   @Bean
   public MemoryStore redisMemoryStore(RedisConnectionFactory connectionFactory) {
       return new RedisMemoryStore(connectionFactory);
   }
   ```

3. **JpaMemoryStore**: 관계형 데이터베이스를 활용한 영구 저장소
   ```java
   @Bean
   public MemoryStore jpaMemoryStore(MemoryEntryRepository repository) {
       return new JpaMemoryStore(repository);
   }
   ```

### 15.4.3 메모리 관리 전략

효과적인 메모리 관리를 위한 전략:

```java
@Service
public class MemoryManagementService {

    private final MemoryStore memoryStore;
    
    public MemoryManagementService(MemoryStore memoryStore) {
        this.memoryStore = memoryStore;
    }
    
    public void storeUserInteraction(String userId, String message, String response) {
        ConversationMemory memory = new ConversationMemory(userId, message, response);
        memoryStore.save(userId, memory);
    }
    
    public List<ConversationMemory> getRecentInteractions(String userId, int limit) {
        return memoryStore.findByUserIdOrderByTimestampDesc(userId, limit);
    }
    
    public void summarizeAndCompressMemory(String userId) {
        List<ConversationMemory> memories = memoryStore.findByUserId(userId);
        
        if (memories.size() > 20) {
            // 오래된 대화 요약
            List<ConversationMemory> oldMemories = memories.subList(10, memories.size());
            String summary = summarizeConversations(oldMemories);
            
            // 요약본 저장 및 오래된 항목 제거
            ConversationMemory summaryMemory = new ConversationMemory(userId, "SUMMARY", summary);
            memoryStore.save(userId, summaryMemory);
            
            for (ConversationMemory oldMemory : oldMemories) {
                memoryStore.delete(userId, oldMemory.getId());
            }
        }
    }
    
    private String summarizeConversations(List<ConversationMemory> memories) {
        // LLM을 활용한 대화 요약 로직
        // ...
        return "요약된 대화 내용";
    }
}
```

### 15.4.4 사용자별 메모리 분리 및 관리

여러 사용자의 대화를 관리하는 방법:

```java
@RestController
@RequestMapping("/api/conversation")
public class ConversationalAgentController {

    private final Agent agent;
    private final MemoryStore memoryStore;
    
    public ConversationalAgentController(Agent agent, MemoryStore memoryStore) {
        this.agent = agent;
        this.memoryStore = memoryStore;
    }
    
    @PostMapping("/chat")
    public ResponseEntity<Map<String, String>> chat(
            @RequestHeader("User-ID") String userId,
            @RequestBody Map<String, String> request) {
        
        String userInput = request.get("message");
        
        // 사용자별 대화 기록 조회
        List<Memory> userMemory = memoryStore.findByUserId(userId);
        
        // 에이전트 요청 생성
        AgentRequest agentRequest = AgentRequest.builder()
            .message(userInput)
            .userId(userId)
            .memories(userMemory)
            .build();
            
        // 에이전트 실행
        AgentResponse response = agent.execute(agentRequest);
        
        // 응답 및 대화 기록 저장
        memoryStore.save(userId, new ConversationMemory(userId, userInput, response.getText()));
        
        Map<String, String> responseBody = new HashMap<>();
        responseBody.put("response", response.getText());
        
        return ResponseEntity.ok(responseBody);
    }
    
    @DeleteMapping("/history/{userId}")
    public ResponseEntity<Void> clearHistory(@PathVariable String userId) {
        memoryStore.deleteByUserId(userId);
        return ResponseEntity.noContent().build();
    }
}
```

## 15.5 고급 에이전트 패턴 구현

### 15.5.1 ReAct 패턴 구현

Reasoning and Acting(ReAct) 패턴을 활용한 에이전트 구현:

```java
@Configuration
public class ReActAgentConfig {

    @Bean
    public Agent reactAgent(ChatClient chatClient, List<Tool> tools, MemoryStore memoryStore) {
        String systemPrompt = """
            당신은 ReAct 패턴을 활용하는 AI 어시스턴트입니다.
            사용자의 요청을 해결하기 위해 다음 단계를 따르세요:
            
            1. Thought: 문제 해결을 위한 생각과 추론 과정을 상세히 설명합니다.
            2. Action: 수행할 도구와 매개변수를 선택합니다.
            3. Observation: 도구 실행 결과를 관찰합니다.
            4. 최종 답변: 관찰 결과를 바탕으로 최종 답변을 제공합니다.
            
            각 단계를 명확히 구분하여 진행하세요.
            """;
        
        return ReActAgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(tools)
            .memoryStore(memoryStore)
            .maxIterations(5)
            .build();
    }
}
```

### 15.5.2 플래너(Planner) 에이전트 구현

복잡한 작업을 계획하고 실행하는 플래너 에이전트:

```java
@Configuration
public class PlannerAgentConfig {

    @Bean
    public Agent plannerAgent(ChatClient chatClient, List<Tool> tools, MemoryStore memoryStore) {
        String systemPrompt = """
            당신은 복잡한 작업을 여러 단계로 분해하고 실행하는 계획 수립 에이전트입니다.
            
            사용자의 요청을 분석하여 다음을 수행하세요:
            1. 목표 이해: 사용자의 최종 목표가 무엇인지 파악합니다.
            2. 계획 수립: 목표 달성을 위한 세부 단계를 계획합니다.
            3. 도구 선택: 각 단계에 필요한 적절한 도구를 선택합니다.
            4. 단계별 실행: 계획한 단계를 순차적으로 실행합니다.
            5. 진행 상황 보고: 각 단계의 실행 결과를 보고합니다.
            6. 최종 결과 제공: 모든 단계가 완료되면 최종 결과를 제공합니다.
            """;
        
        return PlannerAgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(tools)
            .memoryStore(memoryStore)
            .build();
    }
}
```

### 15.5.3 마이크로서비스 기반 에이전트 아키텍처

마이크로서비스 아키텍처를 적용한 에이전트 시스템 설계:

```java
@Configuration
public class MicroserviceAgentConfig {

    @Bean
    public AgentRegistry agentRegistry() {
        return new DefaultAgentRegistry();
    }
    
    @Bean
    public AgentRouter agentRouter(AgentRegistry agentRegistry) {
        return new SkillBasedAgentRouter(agentRegistry);
    }
    
    @Bean
    public AgentService agentService(AgentRouter router, MemoryStore memoryStore) {
        return new AgentServiceImpl(router, memoryStore);
    }
    
    @Bean
    @Qualifier("weatherAgent")
    public Agent weatherAgent(ChatClient chatClient, @Qualifier("weatherTools") List<Tool> weatherTools) {
        String systemPrompt = "당신은 날씨 정보를 제공하는 전문 에이전트입니다.";
        
        return AgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(weatherTools)
            .build();
    }
    
    @Bean
    @Qualifier("calendarAgent")
    public Agent calendarAgent(ChatClient chatClient, @Qualifier("calendarTools") List<Tool> calendarTools) {
        String systemPrompt = "당신은 일정 관리를 돕는 전문 에이전트입니다.";
        
        return AgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(calendarTools)
            .build();
    }
    
    @Bean
    public Agent orchestratorAgent(ChatClient chatClient, AgentRegistry agentRegistry) {
        String systemPrompt = """
            당신은 여러 전문 에이전트를 조율하는 오케스트레이터입니다.
            사용자의 요청을 분석하여 적절한 전문 에이전트에게 작업을 위임하세요.
            """;
        
        return OrchestratorAgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .agentRegistry(agentRegistry)
            .build();
    }
}
```

### 15.5.4 멀티모달 에이전트 구현

텍스트뿐만 아니라 이미지와 같은 다양한 모달리티를 처리하는 에이전트:

```java
@Configuration
public class MultimodalAgentConfig {

    @Bean
    public MultimodalChatClient multimodalChatClient(OpenAiApi openAiApi) {
        return new OpenAiMultimodalChatClient(openAiApi);
    }
    
    @Bean
    public Agent multimodalAgent(MultimodalChatClient chatClient, List<Tool> tools) {
        String systemPrompt = """
            당신은 텍스트와 이미지를 모두 처리할 수 있는 멀티모달 AI 어시스턴트입니다.
            사용자가 이미지를 제공하면 이미지를 분석하고 관련된 정보를 제공해 주세요.
            """;
        
        return MultimodalAgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(tools)
            .build();
    }
    
    @RestController
    @RequestMapping("/api/multimodal")
    public class MultimodalAgentController {
    
        private final Agent multimodalAgent;
        
        public MultimodalAgentController(Agent multimodalAgent) {
            this.multimodalAgent = multimodalAgent;
        }
        
        @PostMapping(value = "/process", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
        public ResponseEntity<Map<String, String>> processMultimodalInput(
                @RequestParam("message") String message,
                @RequestParam(value = "image", required = false) MultipartFile image) throws IOException {
            
            // 멀티모달 요청 생성
            MultimodalAgentRequest.Builder requestBuilder = MultimodalAgentRequest.builder()
                .message(message);
                
            if (image != null && !image.isEmpty()) {
                byte[] imageData = image.getBytes();
                requestBuilder.addMedia(new ImageMedia(imageData, image.getContentType()));
            }
            
            // 에이전트 실행
            AgentResponse response = multimodalAgent.execute(requestBuilder.build());
            
            Map<String, String> responseBody = new HashMap<>();
            responseBody.put("response", response.getText());
            
            return ResponseEntity.ok(responseBody);
        }
    }
}
```

## 15.6 에이전트 배포 및 운영

### 15.6.1 에이전트 확장성 고려사항

에이전트 시스템의 확장성을 위한 고려사항:

1. **모델 선택과 성능**: 요구사항에 맞는 적절한 LLM 선택
2. **비동기 처리**: 장시간 실행 작업을 위한 비동기 처리 메커니즘
3. **캐싱 전략**: 중복 계산 방지를 위한 응답 캐싱
4. **부하 분산**: 여러 LLM 제공자 또는 인스턴스 간 부하 분산
5. **자원 관리**: 토큰 소비, API 호출, 컴퓨팅 자원 관리

### 15.6.2 에이전트 모니터링 및 로깅

에이전트 성능 및 동작 모니터링을 위한 구현:

```java
@Configuration
public class AgentMonitoringConfig {

    @Bean
    public AgentMetrics agentMetrics(MeterRegistry registry) {
        return new AgentMetrics(registry);
    }
    
    @Bean
    public AgentInterceptor agentMonitoringInterceptor(AgentMetrics metrics, AgentLogger logger) {
        return new MonitoringAgentInterceptor(metrics, logger);
    }
    
    @Bean
    public AgentLogger agentLogger() {
        return new StructuredAgentLogger();
    }
}

@Component
public class MonitoringAgentInterceptor implements AgentInterceptor {

    private final AgentMetrics metrics;
    private final AgentLogger logger;
    
    public MonitoringAgentInterceptor(AgentMetrics metrics, AgentLogger logger) {
        this.metrics = metrics;
        this.logger = logger;
    }
    
    @Override
    public void beforeExecution(AgentRequest request, AgentOptions options) {
        metrics.incrementRequestCount();
        logger.logRequest(request);
    }
    
    @Override
    public void afterExecution(AgentRequest request, AgentResponse response, long executionTime) {
        metrics.recordExecutionTime(executionTime);
        metrics.recordTokenUsage(response.getTokenUsage());
        logger.logResponse(request, response, executionTime);
        
        if (response.containsToolCalls()) {
            metrics.incrementToolCallCount();
        }
    }
    
    @Override
    public void onError(AgentRequest request, Exception exception) {
        metrics.incrementErrorCount();
        logger.logError(request, exception);
    }
}
```

### 15.6.3 에이전트 성능 최적화

에이전트 성능 최적화를 위한 전략:

```java
@Configuration
public class OptimizedAgentConfig {

    @Bean
    public ResponseCache responseCache() {
        return new InMemoryResponseCache(1000, Duration.ofHours(1));
    }
    
    @Bean
    public PromptCompressor promptCompressor() {
        return new TokenBasedPromptCompressor();
    }
    
    @Bean
    public Agent optimizedAgent(ChatClient chatClient, List<Tool> tools, 
                               ResponseCache cache, PromptCompressor compressor) {
        String systemPrompt = "당신은 최적화된 성능을 제공하는 AI 어시스턴트입니다.";
        
        return OptimizedAgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(tools)
            .responseCache(cache)
            .promptCompressor(compressor)
            .tokenLimit(4000)
            .build();
    }
}

public class TokenBasedPromptCompressor implements PromptCompressor {

    private final int maxTokens;
    
    public TokenBasedPromptCompressor() {
        this(3000); // 기본 토큰 제한
    }
    
    public TokenBasedPromptCompressor(int maxTokens) {
        this.maxTokens = maxTokens;
    }
    
    @Override
    public String compress(String prompt, List<Memory> memories) {
        // 토큰 계산 및 압축 로직
        int promptTokens = estimateTokens(prompt);
        int availableTokens = maxTokens - promptTokens;
        
        if (availableTokens <= 0) {
            // 프롬프트가 이미 토큰 제한을 초과한 경우 요약
            return summarizePrompt(prompt, maxTokens / 2);
        }
        
        // 메모리 압축
        List<Memory> compressedMemories = compressMemories(memories, availableTokens);
        
        // 압축된 메모리를 프롬프트에 통합
        return integrateMemoriesToPrompt(prompt, compressedMemories);
    }
    
    // 구현 메서드들...
}
```

### 15.6.4 에이전트 보안 고려사항

에이전트 보안 강화를 위한 구현:

```java
@Configuration
public class SecureAgentConfig {

    @Bean
    public ContentFilter contentFilter() {
        return new DefaultContentFilter();
    }
    
    @Bean
    public PermissionManager permissionManager() {
        return new RoleBasedPermissionManager();
    }
    
    @Bean
    public DataSanitizer dataSanitizer() {
        return new DefaultDataSanitizer();
    }
    
    @Bean
    public Agent secureAgent(ChatClient chatClient, List<Tool> tools,
                           ContentFilter filter, PermissionManager permissionManager,
                           DataSanitizer sanitizer) {
        String systemPrompt = """
            당신은 보안을 최우선으로 하는 AI 어시스턴트입니다.
            민감한 정보 요청, 유해한 콘텐츠 생성, 보안 우회 시도에 응답하지 마세요.
            """;
        
        return SecureAgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(tools)
            .contentFilter(filter)
            .permissionManager(permissionManager)
            .dataSanitizer(sanitizer)
            .build();
    }
}

@Component
public class SecurityInterceptor implements HandlerInterceptor {

    private final PermissionManager permissionManager;
    private final AuditLogger auditLogger;
    
    public SecurityInterceptor(PermissionManager permissionManager, AuditLogger auditLogger) {
        this.permissionManager = permissionManager;
        this.auditLogger = auditLogger;
    }
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 사용자 인증 및 권한 확인
        String userId = extractUserId(request);
        String action = determineAction(request);
        
        if (!permissionManager.hasPermission(userId, action)) {
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            return false;
        }
        
        // 입력 검증
        String input = extractUserInput(request);
        if (containsSuspiciousContent(input)) {
            auditLogger.logSecurityEvent(userId, "SUSPICIOUS_INPUT", input);
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            return false;
        }
        
        return true;
    }
    
    // 구현 메서드들...
}
```

## 15.7 에이전트 서비스 통합 방안

### 15.7.1 마이크로서비스 아키텍처 통합

Spring Cloud와 에이전트 서비스 통합:

```java
@EnableDiscoveryClient
@SpringBootApplication
public class AgentServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AgentServiceApplication.class, args);
    }
    
    @Bean
    public Agent serviceAgent(ChatClient chatClient, List<Tool> tools) {
        String systemPrompt = "당신은 마이크로서비스 환경에서 동작하는 AI 어시스턴트입니다.";
        
        return AgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(tools)
            .build();
    }
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}

@Configuration
public class ServiceToolsConfig {

    @Bean
    public Tool userServiceTool(WebClient.Builder webClientBuilder) {
        return new MicroserviceTool(
            "user-service",
            "사용자 정보 관리 서비스",
            webClientBuilder.baseUrl("http://user-service").build()
        );
    }
    
    @Bean
    public Tool productServiceTool(WebClient.Builder webClientBuilder) {
        return new MicroserviceTool(
            "product-service",
            "제품 정보 관리 서비스",
            webClientBuilder.baseUrl("http://product-service").build()
        );
    }
    
    @Bean
    public Tool orderServiceTool(WebClient.Builder webClientBuilder) {
        return new MicroserviceTool(
            "order-service",
            "주문 처리 서비스",
            webClientBuilder.baseUrl("http://order-service").build()
        );
    }
}
```

### 15.7.2 이벤트 기반 아키텍처 통합

Spring Cloud Stream과 에이전트 통합:

```java
@Configuration
public class EventDrivenAgentConfig {

    @Bean
    public Function<Message<AgentRequest>, Message<AgentResponse>> processAgentRequest(Agent agent) {
        return message -> {
            AgentRequest request = message.getPayload();
            AgentResponse response = agent.execute(request);
            
            return MessageBuilder
                .withPayload(response)
                .copyHeaders(message.getHeaders())
                .build();
        };
    }
    
    @Bean
    public Consumer<Message<AgentRequest>> asyncAgentProcessor(Agent agent, StreamBridge streamBridge) {
        return message -> {
            AgentRequest request = message.getPayload();
            
            CompletableFuture.runAsync(() -> {
                try {
                    AgentResponse response = agent.execute(request);
                    
                    Message<AgentResponse> responseMessage = MessageBuilder
                        .withPayload(response)
                        .copyHeaders(message.getHeaders())
                        .build();
                        
                    streamBridge.send("agentResponses", responseMessage);
                }
                catch (Exception e) {
                    // 오류 처리 및 로깅
                }
            });
        };
    }
}

@Component
public class AgentEventListener {

    private final Agent agent;
    private final StreamBridge streamBridge;
    
    public AgentEventListener(Agent agent, StreamBridge streamBridge) {
        this.agent = agent;
        this.streamBridge = streamBridge;
    }
    
    @Bean
    public Consumer<UserEvent> handleUserEvent() {
        return event -> {
            // 사용자 이벤트를 에이전트 요청으로 변환
            AgentRequest request = convertUserEventToAgentRequest(event);
            
            // 에이전트 실행
            AgentResponse response = agent.execute(request);
            
            // 에이전트 응답 처리
            if (response.requiresAction()) {
                streamBridge.send("actionRequests", response.getActionRequest());
            }
            else {
                streamBridge.send("notifications", buildNotification(event.getUserId(), response));
            }
        };
    }
    
    // 구현 메서드들...
}
```

### 15.7.3 RESTful API 통합

에이전트를 RESTful API로 노출하는 방법:

```java
@RestController
@RequestMapping("/api/v1/agent")
public class AgentApiController {

    private final Agent agent;
    
    public AgentApiController(Agent agent) {
        this.agent = agent;
    }
    
    @PostMapping("/execute")
    public ResponseEntity<AgentApiResponse> executeAgent(@RequestBody AgentApiRequest apiRequest) {
        // API 요청을 에이전트 요청으로 변환
        AgentRequest agentRequest = AgentRequest.builder()
            .message(apiRequest.getMessage())
            .userId(apiRequest.getUserId())
            .sessionId(apiRequest.getSessionId())
            .build();
            
        // 에이전트 실행
        AgentResponse agentResponse = agent.execute(agentRequest);
        
        // 응답 생성
        AgentApiResponse apiResponse = new AgentApiResponse(
            agentResponse.getText(),
            agentResponse.getToolResults(),
            agentResponse.getMetadata()
        );
        
        return ResponseEntity.ok(apiResponse);
    }
    
    @PostMapping("/stream")
    public Flux<ServerSentEvent<String>> streamAgent(@RequestBody AgentApiRequest apiRequest) {
        // 스트리밍 응답을 위한 에이전트 옵션 설정
        AgentOptions options = AgentOptions.builder()
            .streaming(true)
            .build();
            
        // API 요청을 에이전트 요청으로 변환
        AgentRequest agentRequest = AgentRequest.builder()
            .message(apiRequest.getMessage())
            .userId(apiRequest.getUserId())
            .sessionId(apiRequest.getSessionId())
            .build();
            
        // 스트리밍 응답 생성
        return Flux.create(sink -> {
            agent.executeStreaming(agentRequest, options, chunk -> {
                sink.next(ServerSentEvent.builder(chunk).build());
                
                if (chunk.isLast()) {
                    sink.complete();
                }
            });
        });
    }
}
```

### 15.7.4 봇 프레임워크 통합

Spring AI 에이전트와 메시징 플랫폼 통합:

```java
@Configuration
public class BotConfig {

    @Bean
    public Agent botAgent(ChatClient chatClient, List<Tool> tools, MemoryStore memoryStore) {
        String systemPrompt = "당신은 메시징 플랫폼에서 사용자를 돕는 AI 봇입니다.";
        
        return AgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(tools)
            .memoryStore(memoryStore)
            .build();
    }
}

@Component
public class SlackBotAdapter {

    private final Agent botAgent;
    private final MemoryStore memoryStore;
    
    public SlackBotAdapter(Agent botAgent, MemoryStore memoryStore) {
        this.botAgent = botAgent;
        this.memoryStore = memoryStore;
    }
    
    @EventListener
    public void handleSlackMessage(SlackMessageEvent event) {
        // 슬랙 메시지를 에이전트 요청으로 변환
        String userId = event.getUserId();
        String channelId = event.getChannelId();
        String message = event.getText();
        
        // 사용자 메모리 조회
        List<Memory> userMemory = memoryStore.findByUserId(userId);
        
        // 에이전트 요청 생성
        AgentRequest agentRequest = AgentRequest.builder()
            .message(message)
            .userId(userId)
            .sessionId(channelId)
            .memories(userMemory)
            .build();
            
        // 에이전트 실행
        AgentResponse response = botAgent.execute(agentRequest);
        
        // 메모리 저장
        memoryStore.save(userId, new ConversationMemory(userId, message, response.getText()));
        
        // 슬랙으로 응답 전송
        sendSlackResponse(channelId, response.getText());
    }
    
    private void sendSlackResponse(String channelId, String message) {
        // 슬랙 API를 통한 메시지 전송 로직
        // ...
    }
}
```

## 15.8 Google ADK와 Spring AI 통합

### 15.8.1 Google ADK 소개

Google Agent Development Kit (ADK)는 2025년 Google Cloud NEXT에서 발표된 오픈소스 코드-우선(code-first) 프레임워크로, 정교한 AI 에이전트와 멀티 에이전트 시스템을 개발할 수 있도록 설계되었습니다.

**주요 특징:**
- **멀티 에이전트 설계**: 특화된 에이전트들을 계층적으로 구성
- **풍부한 모델 생태계**: Gemini 최적화 및 200개 이상 모델 지원
- **A2A (Agent-to-Agent) Protocol**: 표준화된 에이전트 간 통신
- **MCP (Model Context Protocol) 통합**: 표준화된 도구 통신
- **Google Cloud 네이티브 통합**: Vertex AI와 완전 통합

### 15.8.2 Spring AI와 Google ADK 통합 아키텍처

```java
@Configuration
public class GoogleADKIntegrationConfig {
    
    @Bean
    public VertexAiGeminiChatModel vertexAiChatModel() {
        return VertexAiGeminiChatModel.builder()
            .projectId("${spring.ai.vertex.ai.gemini.project-id}")
            .location("${spring.ai.vertex.ai.gemini.location}")
            .modelName("gemini-2.0-flash")
            .build();
    }
    
    @Bean
    public ChatClient adkIntegratedChatClient(VertexAiGeminiChatModel chatModel,
                                             ADKBridgeTools adkTools) {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                당신은 Google ADK와 통합된 Spring AI 에이전트입니다.
                복잡한 멀티 에이전트 작업이 필요한 경우 ADK 에이전트에게 위임하세요.
                """)
            .defaultTools(adkTools)
            .build();
    }
}
```

### 15.8.3 A2A (Agent-to-Agent) 프로토콜 구현

A2A 프로토콜을 활용한 Spring AI와 Google ADK 간 통신:

```java
@Component
public class ADKBridgeTools {
    
    private final WebClient adkWebClient;
    private final ObjectMapper objectMapper;
    
    public ADKBridgeTools(WebClient.Builder webClientBuilder, ObjectMapper objectMapper) {
        this.adkWebClient = webClientBuilder
            .baseUrl("${adk.agent.base-url}")
            .defaultHeader("Authorization", "Bearer ${adk.agent.token}")
            .build();
        this.objectMapper = objectMapper;
    }
    
    @Tool(description = "복잡한 연구 작업을 ADK 연구 에이전트에게 위임합니다")
    public Mono<ResearchResult> delegateResearch(
            @ToolParam(description = "연구할 주제") String topic,
            @ToolParam(description = "연구 깊이 (shallow, medium, deep)") String depth) {
        
        Map<String, Object> request = Map.of(
            "action", "research",
            "parameters", Map.of(
                "topic", topic,
                "depth", depth,
                "max_sources", 10
            )
        );
        
        return adkWebClient.post()
            .uri("/api/agent/task")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(ResearchResult.class);
    }
    
    @Tool(description = "멀티 에이전트 워크플로우를 ADK에서 실행합니다")
    public Mono<WorkflowResult> executeWorkflow(
            @ToolParam(description = "워크플로우 정의") String workflowSpec,
            @ToolParam(description = "입력 데이터") Map<String, Object> inputData) {
        
        Map<String, Object> request = Map.of(
            "action", "execute_workflow",
            "workflow", workflowSpec,
            "data", inputData
        );
        
        return adkWebClient.post()
            .uri("/api/workflow/execute")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(WorkflowResult.class);
    }
}
```

### 15.8.4 MCP를 통한 도구 공유

Spring AI의 MCP 클라이언트를 활용한 ADK 도구 통합:

```java
@Configuration
public class MCPADKIntegrationConfig {
    
    @Bean
    public McpClient adkMcpClient() {
        return McpClient.builder()
            .transport(SseClientTransport.builder()
                .uri("${adk.mcp.endpoint}")
                .build())
            .build();
    }
    
    @Bean
    public ChatClient mcpEnabledChatClient(ChatModel chatModel, McpClient mcpClient) {
        return ChatClient.builder(chatModel)
            .defaultSystem("MCP를 통해 ADK 도구에 접근할 수 있는 에이전트입니다.")
            .defaultTools("mcp::google_search", "mcp::web_fetch", "mcp::code_executor")
            .toolContext(Map.of("mcpClient", mcpClient))
            .build();
    }
}

@Component 
public class MCPToolProxy {
    
    @Tool(description = "MCP를 통해 Google 검색을 실행합니다")
    public String executeGoogleSearch(
            @ToolParam(description = "검색 쿼리") String query,
            ToolContext context) {
        
        McpClient mcpClient = (McpClient) context.getContext().get("mcpClient");
        
        return mcpClient.callTool("google_search", Map.of("query", query))
            .map(result -> result.getContent())
            .block();
    }
}
```

### 15.8.5 하이브리드 에이전트 시스템 구현

Spring AI와 Google ADK를 활용한 하이브리드 시스템:

```java
@Service
public class HybridAgentOrchestrator {
    
    private final ChatClient springAiChatClient;
    private final ADKBridgeTools adkBridge;
    
    public HybridAgentOrchestrator(ChatClient springAiChatClient, ADKBridgeTools adkBridge) {
        this.springAiChatClient = springAiChatClient;
        this.adkBridge = adkBridge;
    }
    
    public Mono<AgentResponse> processComplexRequest(ComplexRequest request) {
        // 1. Spring AI로 초기 분석
        String initialAnalysis = springAiChatClient.prompt()
            .user("다음 요청을 분석하고 처리 방향을 제시하세요: " + request.getDescription())
            .call()
            .content();
        
        // 2. 복잡성 판단
        if (requiresMultiAgent(initialAnalysis)) {
            // ADK 멀티 에이전트 시스템으로 위임
            return adkBridge.executeWorkflow(
                buildWorkflowSpec(request),
                Map.of("initialAnalysis", initialAnalysis)
            ).map(this::convertToAgentResponse);
        } else {
            // Spring AI로 직접 처리
            return Mono.just(processWithSpringAI(request, initialAnalysis));
        }
    }
    
    private boolean requiresMultiAgent(String analysis) {
        return analysis.toLowerCase().contains("복잡한") || 
               analysis.toLowerCase().contains("다단계") ||
               analysis.toLowerCase().contains("전문가");
    }
    
    private String buildWorkflowSpec(ComplexRequest request) {
        return """
            {
              "agents": [
                {"name": "researcher", "role": "정보 수집"},
                {"name": "analyzer", "role": "데이터 분석"},
                {"name": "synthesizer", "role": "결과 종합"}
              ],
              "workflow": "sequential",
              "coordination": "llm_based"
            }
            """;
    }
}
```

### 15.8.6 ADK 에이전트 디스커버리 및 라우팅

```java
@Service
public class ADKAgentRegistry {
    
    private final Map<String, AgentInfo> registeredAgents = new ConcurrentHashMap<>();
    private final WebClient discoveryClient;
    
    @PostConstruct
    public void discoverAgents() {
        // A2A 프로토콜을 통한 에이전트 디스커버리
        discoveryClient.get()
            .uri("/api/agents/discover")
            .retrieve()
            .bodyToFlux(AgentInfo.class)
            .doOnNext(agent -> registeredAgents.put(agent.getName(), agent))
            .subscribe();
    }
    
    @Tool(description = "적절한 ADK 에이전트를 찾아 작업을 라우팅합니다")
    public Mono<String> routeToAgent(
            @ToolParam(description = "작업 설명") String taskDescription,
            @ToolParam(description = "필요한 전문성") String requiredExpertise) {
        
        AgentInfo bestAgent = findBestAgent(taskDescription, requiredExpertise);
        
        if (bestAgent == null) {
            return Mono.just("적절한 에이전트를 찾을 수 없습니다.");
        }
        
        return WebClient.create(bestAgent.getEndpoint())
            .post()
            .uri("/api/task")
            .bodyValue(Map.of(
                "description", taskDescription,
                "expertise", requiredExpertise
            ))
            .retrieve()
            .bodyToMono(String.class);
    }
    
    private AgentInfo findBestAgent(String taskDescription, String expertise) {
        return registeredAgents.values().stream()
            .filter(agent -> agent.getCapabilities().contains(expertise))
            .max(Comparator.comparing(agent -> 
                calculateRelevanceScore(agent, taskDescription)))
            .orElse(null);
    }
}
```

### 15.8.7 모니터링 및 관측성

```java
@Component
public class HybridAgentObservability {
    
    private final MeterRegistry meterRegistry;
    private final Tracer tracer;
    
    public HybridAgentObservability(MeterRegistry meterRegistry, Tracer tracer) {
        this.meterRegistry = meterRegistry;
        this.tracer = tracer;
    }
    
    @EventListener
    public void onSpringAIAgentCall(SpringAIAgentCallEvent event) {
        meterRegistry.counter("agent.calls.spring_ai",
            "model", event.getModelName(),
            "success", String.valueOf(event.isSuccessful()))
            .increment();
        
        meterRegistry.timer("agent.execution.time.spring_ai")
            .record(event.getExecutionTime(), TimeUnit.MILLISECONDS);
    }
    
    @EventListener
    public void onADKAgentCall(ADKAgentCallEvent event) {
        meterRegistry.counter("agent.calls.adk",
            "agent_name", event.getAgentName(),
            "protocol", event.getProtocol())
            .increment();
    }
    
    public Span createHybridAgentSpan(String operation) {
        return tracer.nextSpan()
            .name("hybrid-agent-" + operation)
            .tag("component", "spring-ai-adk-bridge")
            .start();
    }
}
```

## 15.9 요약 및 다음 단계

이 장에서는 Spring AI 1.0.0 GA를 활용한 에이전트 개발의 실질적인 구현 방법에 대해 알아보았습니다. ChatClient 중심의 현대적 접근법부터 Google ADK와의 통합까지 다양한 측면을 살펴보았습니다.

**주요 학습 내용:**

1. **Spring AI 1.0.0 GA 에이전트 아키텍처**
   - ChatClient 중심의 컴포지션 패턴
   - Advisors API를 통한 재사용 가능한 AI 패턴
   - @Tool 어노테이션을 활용한 선언적 도구 정의

2. **Google ADK 통합**
   - A2A 프로토콜을 통한 에이전트 간 통신
   - MCP를 활용한 표준화된 도구 공유
   - 하이브리드 에이전트 시스템 구축

3. **엔터프라이즈 고려사항**
   - 보안과 인증
   - 모니터링과 관측성
   - 확장성과 성능 최적화

Spring AI 1.0.0 GA는 풍부한 기능과 확장 가능한 구조를 제공하여 다양한 복잡도의 AI 에이전트 애플리케이션을 개발할 수 있게 해줍니다. Google ADK와의 통합을 통해 멀티 에이전트 시스템의 강력한 기능을 활용할 수 있으며, A2A 프로토콜을 통해 에이전트 간 상호운용성을 보장할 수 있습니다.

다음 장에서는 A2A (Agent-to-Agent) 프로토콜에 대해 더 자세히 살펴보고, 에이전트 간 통신과 협업 패턴을 구현하는 방법을 알아보겠습니다.