# 15. Spring AI를 활용한 에이전트 구축

## 15.1 Spring AI 에이전트 프레임워크 개요

Spring AI는 LLM 기반 AI 에이전트를 쉽게 개발할 수 있는 포괄적인 프레임워크를 제공합니다. 이 프레임워크는 Spring 생태계의 핵심 원칙과 패턴을 활용하여 견고하고 확장 가능한 에이전트 애플리케이션 구축을 지원합니다.

### 15.1.1 에이전트 프레임워크 아키텍처

Spring AI 에이전트 프레임워크는 다음과 같은 핵심 구성 요소로 이루어져 있습니다:

- **Agent**: 사용자 요청을 처리하고 적절한 도구를 선택하여 작업을 수행하는 중앙 컴포넌트
- **Tool**: 에이전트가 특정 작업을 수행하기 위해 활용하는 기능 단위
- **Prompt**: 에이전트의 동작을 정의하는 지시문
- **Memory**: 대화 맥락과 상태를 유지하는 저장소
- **Input/Output Processors**: 입력과 출력을 처리하는 컴포넌트

이러한 구성 요소들이 함께 작동하여 에이전트의 인지-추론-행동 사이클을 구현합니다.

### 15.1.2 에이전트 인터페이스 및 구현체

Spring AI는 에이전트 개발을 위한 다양한 인터페이스와 구현체를 제공합니다:

```java
public interface Agent {
    AgentResponse execute(AgentRequest request);
    AgentResponse execute(AgentRequest request, AgentOptions options);
}
```

주요 에이전트 구현체:

- **SimpleAgent**: 기본적인 단일 요청-응답 에이전트
- **ConversationalAgent**: 대화 맥락을 유지하는 에이전트
- **ToolAwareAgent**: 다양한 도구를 활용할 수 있는 에이전트
- **ReActAgent**: ReAct(Reasoning and Acting) 패턴을 구현한 에이전트

### 15.1.3 의존성 설정

Spring AI 에이전트 기능을 활용하기 위한 Maven 의존성 설정:

```xml
<dependencies>
    <!-- Spring AI Core -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-core</artifactId>
        <version>${spring-ai.version}</version>
    </dependency>
    
    <!-- Spring AI Agents -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-agents</artifactId>
        <version>${spring-ai.version}</version>
    </dependency>
    
    <!-- LLM Provider (예: OpenAI) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai</artifactId>
        <version>${spring-ai.version}</version>
    </dependency>
    
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

## 15.2 첫 번째 에이전트 구현하기

### 15.2.1 기본 에이전트 설정

간단한 에이전트를 구현하기 위한 기본 설정부터 시작해 보겠습니다:

```java
@Configuration
public class AgentConfig {

    @Bean
    public ChatClient chatClient(OpenAiApi openAiApi) {
        return new OpenAiChatClient(openAiApi);
    }
    
    @Bean
    public Agent simpleAgent(ChatClient chatClient) {
        String systemPrompt = "당신은 도움을 주는 AI 어시스턴트입니다. 사용자의 질문에 명확하고 정확하게 답변해 주세요.";
        
        return AgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .build();
    }
}
```

### 15.2.2 에이전트 컨트롤러 구현

에이전트와 상호작용하기 위한 REST API 컨트롤러:

```java
@RestController
@RequestMapping("/api/agent")
public class AgentController {

    private final Agent agent;
    
    public AgentController(Agent agent) {
        this.agent = agent;
    }
    
    @PostMapping("/chat")
    public ResponseEntity<Map<String, String>> chat(@RequestBody Map<String, String> request) {
        String userInput = request.get("message");
        
        AgentRequest agentRequest = AgentRequest.builder()
            .message(userInput)
            .build();
            
        AgentResponse response = agent.execute(agentRequest);
        
        Map<String, String> responseBody = new HashMap<>();
        responseBody.put("response", response.getText());
        
        return ResponseEntity.ok(responseBody);
    }
}
```

### 15.2.3 에이전트 테스트

구현한 에이전트를 테스트하는 방법:

```java
@SpringBootTest
class SimpleAgentTest {

    @Autowired
    private Agent agent;
    
    @Test
    void testSimpleAgent() {
        // Given
        String userInput = "Spring AI란 무엇인가요?";
        AgentRequest request = AgentRequest.builder()
            .message(userInput)
            .build();
            
        // When
        AgentResponse response = agent.execute(request);
        
        // Then
        assertNotNull(response);
        assertFalse(response.getText().isEmpty());
        System.out.println("Agent Response: " + response.getText());
    }
}
```

## 15.3 도구 사용 에이전트 개발

### 15.3.1 도구(Tool) 인터페이스 이해하기

Spring AI의 Tool 인터페이스는 에이전트가 외부 시스템과 상호작용하기 위한 방법을 정의합니다:

```java
public interface Tool {
    String getName();
    String getDescription();
    Object execute(Map<String, Object> parameters);
    ToolParameterDefinition getParameterDefinition();
}
```

Tool 인터페이스의 주요 구성 요소:
- **getName()**: 도구의 고유 이름을 반환
- **getDescription()**: 도구의 기능 설명을 반환
- **execute()**: 도구의 실제 기능을 실행
- **getParameterDefinition()**: 도구가 필요로 하는 매개변수 정의를 반환

### 15.3.2 사용자 정의 도구 구현

날씨 정보를 제공하는 커스텀 도구 구현 예시:

```java
@Component
public class WeatherTool implements Tool {

    private final WeatherService weatherService;
    
    public WeatherTool(WeatherService weatherService) {
        this.weatherService = weatherService;
    }
    
    @Override
    public String getName() {
        return "weather";
    }
    
    @Override
    public String getDescription() {
        return "현재 날씨 정보를 제공합니다. 도시 이름을 입력하면 해당 도시의 날씨를 알려줍니다.";
    }
    
    @Override
    public Object execute(Map<String, Object> parameters) {
        String city = (String) parameters.get("city");
        if (city == null || city.isBlank()) {
            throw new IllegalArgumentException("도시 이름이 필요합니다.");
        }
        
        WeatherInfo weatherInfo = weatherService.getWeatherForCity(city);
        
        Map<String, Object> result = new HashMap<>();
        result.put("temperature", weatherInfo.getTemperature());
        result.put("condition", weatherInfo.getCondition());
        result.put("humidity", weatherInfo.getHumidity());
        
        return result;
    }
    
    @Override
    public ToolParameterDefinition getParameterDefinition() {
        return ToolParameterDefinition.builder()
            .addParameter("city", ParameterType.STRING, "날씨 정보를 조회할 도시 이름")
            .build();
    }
}
```

### 15.3.3 도구를 활용하는 에이전트 구성

여러 도구를 사용하는 에이전트 구성 예시:

```java
@Configuration
public class ToolAwareAgentConfig {

    @Bean
    public Agent toolAwareAgent(ChatClient chatClient, List<Tool> tools) {
        String systemPrompt = """
            당신은 도움을 주는 AI 어시스턴트입니다.
            사용자의 요청을 이해하고 적절한 도구를 활용하여 정확한 정보를 제공해 주세요.
            사용자가 요청한 작업을 수행하기 위해 필요한 도구를 선택하고 활용하세요.
            도구를 사용할 때는 필요한 매개변수를 정확히 제공해야 합니다.
            """;
        
        return AgentBuilder.builder()
            .chatClient(chatClient)
            .systemPrompt(systemPrompt)
            .tools(tools)
            .build();
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

## 15.8 요약 및 다음 단계

이 장에서는 Spring AI를 활용한 에이전트 개발의 실질적인 구현 방법에 대해 알아보았습니다. 기본 에이전트 구성부터 도구 사용, 대화 맥락 관리, 고급 패턴 구현, 배포 및 통합에 이르기까지 다양한 측면을 살펴보았습니다.

Spring AI 에이전트 프레임워크는 풍부한 기능과 확장 가능한 구조를 제공하여 다양한 복잡도의 AI 에이전트 애플리케이션을 개발할 수 있게 해줍니다. 기본 원칙과 패턴을 이해하고, 제공된 구성 요소들을 활용하면 비즈니스 요구사항에 맞는 맞춤형 에이전트를 효과적으로 개발할 수 있습니다.

다음 장에서는 더 고급 에이전트 기능에 대해 살펴보고, 실제 비즈니스 시나리오에 적용하는 방법을 알아보겠습니다. 이를 통해 에이전트의 역량을 더욱 확장하고, 더 복잡한 문제를 해결하는 방법을 배우게 될 것입니다.