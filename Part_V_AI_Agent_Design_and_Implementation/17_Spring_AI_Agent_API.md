# 17장: Spring AI Agent API

## 17.1 Spring AI Agent API 개요

Spring AI Agent API는 지능형 에이전트를 쉽게 구축하고 관리할 수 있는 일관된 인터페이스와 추상화를 제공합니다. 이 장에서는 Spring AI Agent API의 핵심 구성요소, 아키텍처, 그리고 다양한 에이전트 구현 방법에 대해 살펴보겠습니다.

### 17.1.1 Agent API의 설계 원칙

- 일관성 있는 에이전트 인터페이스
- 확장 가능한 아키텍처
- 다양한 AI 모델 지원
- 손쉬운 통합 및 구성

### 17.1.2 Agent API 아키텍처 개요

```
+----------------+      +----------------+      +----------------+
|                |      |                |      |                |
|  Agent API     |----->|  Agent Engine  |----->|  Model Client  |
|                |      |                |      |                |
+----------------+      +----------------+      +----------------+
        |                      |                        |
        v                      v                        v
+----------------+      +----------------+      +----------------+
|                |      |                |      |                |
|  Tools & Skills|      |  Memory        |      |  Retrieval     |
|                |      |                |      |                |
+----------------+      +----------------+      +----------------+
```

## 17.2 Agent 인터페이스와 기본 구현체

Spring AI는 다양한 에이전트 유형을 위한 핵심 인터페이스와 기본 구현체를 제공합니다.

### 17.2.1 Agent 인터페이스

```java
public interface Agent {
    /**
     * 에이전트에 메시지를 전송하고 응답을 받습니다.
     */
    AgentResponse execute(AgentRequest request);
    
    /**
     * 에이전트에 메시지를 스트리밍 방식으로 전송하고 응답을 스트리밍으로 받습니다.
     */
    Flux<AgentResponse> executeStreaming(AgentRequest request);
    
    /**
     * 에이전트가 가진 기능(툴)을 반환합니다.
     */
    List<Tool> getTools();
    
    /**
     * 에이전트의 구성 정보를 반환합니다.
     */
    AgentConfig getConfig();
}
```

### 17.2.2 기본 에이전트 구현체

```java
@Component
public class SimpleAgent implements Agent {
    
    private final ModelClient modelClient;
    private final List<Tool> tools;
    private final AgentMemory memory;
    
    public SimpleAgent(ModelClient modelClient, List<Tool> tools, AgentMemory memory) {
        this.modelClient = modelClient;
        this.tools = tools;
        this.memory = memory;
    }
    
    @Override
    public AgentResponse execute(AgentRequest request) {
        // 구현 내용...
    }
    
    @Override
    public Flux<AgentResponse> executeStreaming(AgentRequest request) {
        // 구현 내용...
    }
    
    @Override
    public List<Tool> getTools() {
        return this.tools;
    }
    
    @Override
    public AgentConfig getConfig() {
        // 구성 정보 반환
    }
}
```

### 17.2.3 자동 구성 지원

```java
@Configuration
@ConditionalOnClass(Agent.class)
public class AgentAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public AgentRegistry agentRegistry() {
        return new DefaultAgentRegistry();
    }
    
    @Bean
    @ConditionalOnMissingBean
    public AgentMemory agentMemory(VectorStore vectorStore) {
        return new DefaultAgentMemory(vectorStore);
    }
    
    @Bean
    @ConditionalOnMissingBean
    public ToolRegistry toolRegistry() {
        return new DefaultToolRegistry();
    }
}
```

## 17.3 AgentRequest와 AgentResponse

Agent API는 메시지 교환을 위한 표준화된 요청-응답 구조를 제공합니다.

### 17.3.1 AgentRequest 구조

```java
public class AgentRequest {
    private final String prompt;
    private final Map<String, Object> inputs;
    private final List<Message> messages;
    private final Map<String, Object> metadata;
    
    // 생성자, 빌더, 게터 등...
    
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        // 빌더 구현...
    }
}
```

### 17.3.2 AgentResponse 구조

```java
public class AgentResponse {
    private final String content;
    private final Map<String, Object> outputs;
    private final List<Message> messages;
    private final List<ToolExecution> toolExecutions;
    private final Map<String, Object> metadata;
    
    // 생성자, 빌더, 게터 등...
    
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        // 빌더 구현...
    }
}
```

### 17.3.3 메시지 타입과 구조

```java
public interface Message {
    String getContent();
    MessageType getType();
    Map<String, Object> getMetadata();
}

public enum MessageType {
    USER,
    SYSTEM,
    ASSISTANT,
    TOOL,
    FUNCTION,
    ERROR
}
```

## 17.4 Tool 인터페이스와 구현

Agent API의 핵심 기능 중 하나는 도구(Tool) 통합입니다. 이 섹션에서는 에이전트가 외부 기능을 활용할 수 있게 해주는 Tool 인터페이스와 구현에 대해 알아봅니다.

### 17.4.1 Tool 인터페이스

```java
public interface Tool {
    String getName();
    String getDescription();
    ToolParameterSchema getParameterSchema();
    Object execute(Map<String, Object> parameters);
}
```

### 17.4.2 @Tool 어노테이션을 사용한 도구 정의

```java
@Service
public class WeatherService {
    
    @Tool(name = "get_weather", description = "Get current weather for a location")
    @ToolParameters({
        @ToolParameter(name = "location", description = "City name or zip code", required = true),
        @ToolParameter(name = "units", description = "Temperature units (celsius/fahrenheit)", defaultValue = "celsius")
    })
    public WeatherInfo getWeather(String location, String units) {
        // 날씨 정보를 가져오는 구현...
        return weatherInfo;
    }
}
```

### 17.4.3 ToolRegistry와 도구 관리

```java
@Service
public class DefaultToolRegistry implements ToolRegistry {
    
    private final Map<String, Tool> tools = new ConcurrentHashMap<>();
    
    @Override
    public void registerTool(Tool tool) {
        tools.put(tool.getName(), tool);
    }
    
    @Override
    public Tool getTool(String name) {
        return tools.get(name);
    }
    
    @Override
    public List<Tool> getAllTools() {
        return new ArrayList<>(tools.values());
    }
    
    @Override
    public void registerToolsFromBean(Object bean) {
        // 빈에서 @Tool 어노테이션이 있는 메서드를 찾아 등록
    }
}
```

## 17.5 에이전트 메모리 관리

에이전트가 이전 상호작용을 기억하고 문맥을 유지하기 위한 메모리 관리 방법을 알아봅니다.

### 17.5.1 AgentMemory 인터페이스

```java
public interface AgentMemory {
    void saveMessage(String sessionId, Message message);
    List<Message> getMessages(String sessionId);
    void saveState(String sessionId, String key, Object value);
    Optional<Object> getState(String sessionId, String key);
    void clear(String sessionId);
}
```

### 17.5.2 메모리 구현 전략

```java
@Component
public class DefaultAgentMemory implements AgentMemory {
    
    private final Map<String, List<Message>> messageStore = new ConcurrentHashMap<>();
    private final Map<String, Map<String, Object>> stateStore = new ConcurrentHashMap<>();
    
    @Override
    public void saveMessage(String sessionId, Message message) {
        messageStore.computeIfAbsent(sessionId, k -> new ArrayList<>()).add(message);
    }
    
    @Override
    public List<Message> getMessages(String sessionId) {
        return messageStore.getOrDefault(sessionId, Collections.emptyList());
    }
    
    @Override
    public void saveState(String sessionId, String key, Object value) {
        stateStore.computeIfAbsent(sessionId, k -> new HashMap<>()).put(key, value);
    }
    
    @Override
    public Optional<Object> getState(String sessionId, String key) {
        Map<String, Object> sessionState = stateStore.get(sessionId);
        if (sessionState == null) {
            return Optional.empty();
        }
        return Optional.ofNullable(sessionState.get(key));
    }
    
    @Override
    public void clear(String sessionId) {
        messageStore.remove(sessionId);
        stateStore.remove(sessionId);
    }
}
```

### 17.5.3 벡터 저장소를 이용한 장기 메모리

```java
@Component
public class VectorStoreAgentMemory implements AgentMemory {
    
    private final VectorStore vectorStore;
    private final TextEmbedding textEmbedding;
    private final DefaultAgentMemory shortTermMemory;
    
    // 구현 내용...
    
    @Override
    public List<Message> getMessages(String sessionId) {
        // 단기 메모리에서 최근 메시지 가져오기
        List<Message> recentMessages = shortTermMemory.getMessages(sessionId);
        
        // 관련 장기 메모리 검색
        String currentContext = recentMessages.stream()
            .map(Message::getContent)
            .collect(Collectors.joining(" "));
        
        List<Document> relevantDocuments = retrieveRelevantMemories(currentContext);
        
        // 관련 장기 메모리를 메시지로 변환하여 추가
        List<Message> augmentedMessages = new ArrayList<>(recentMessages);
        // ... 장기 메모리 통합 로직 ...
        
        return augmentedMessages;
    }
    
    private List<Document> retrieveRelevantMemories(String context) {
        // 벡터 저장소에서 관련 메모리 검색
    }
}
```

## 17.6 에이전트 생성 패턴

Spring AI에서 다양한 유형의 에이전트를 생성하고 구성하는 방법을 알아봅니다.

### 17.6.1 Bean 기반 에이전트 정의

```java
@Configuration
public class AgentConfiguration {
    
    @Bean
    public Agent customerSupportAgent(
            @Qualifier("openai") ModelClient modelClient,
            ToolRegistry toolRegistry,
            AgentMemory memory) {
        
        return SimpleAgent.builder()
            .modelClient(modelClient)
            .tools(toolRegistry.getAllTools())
            .memory(memory)
            .systemPrompt("당신은 고객 지원 전문가입니다. 친절하고 정확하게 답변해 주세요.")
            .build();
    }
    
    @Bean
    public Agent dataAnalysisAgent(
            @Qualifier("anthropic") ModelClient modelClient,
            ToolRegistry toolRegistry,
            AgentMemory memory) {
        
        return SimpleAgent.builder()
            .modelClient(modelClient)
            .tools(Arrays.asList(
                toolRegistry.getTool("query_database"),
                toolRegistry.getTool("generate_chart"),
                toolRegistry.getTool("statistical_analysis")
            ))
            .memory(memory)
            .systemPrompt("당신은 데이터 분석 전문가입니다. 정확한 분석과 인사이트를 제공해 주세요.")
            .build();
    }
}
```

### 17.6.2 AgentTemplate을 이용한 동적 에이전트 생성

```java
@Service
public class AgentFactory {
    
    private final AgentTemplate agentTemplate;
    private final ToolRegistry toolRegistry;
    private final Map<String, ModelClient> modelClients;
    
    public Agent createAgentFromSpec(AgentSpec spec) {
        ModelClient modelClient = modelClients.get(spec.getModelProvider());
        List<Tool> tools = spec.getTools().stream()
            .map(toolRegistry::getTool)
            .collect(Collectors.toList());
        
        return agentTemplate.createAgent(
            modelClient,
            tools,
            spec.getSystemPrompt(),
            spec.getConfig()
        );
    }
}
```

### 17.6.3 프로그래밍 방식의 에이전트 구성

```java
@RestController
@RequestMapping("/agents")
public class AgentController {
    
    private final Map<String, Agent> agents = new ConcurrentHashMap<>();
    private final ModelClient defaultModel;
    private final ToolRegistry toolRegistry;
    private final AgentMemory memory;
    
    @PostMapping
    public ResponseEntity<String> createAgent(@RequestBody AgentCreationRequest request) {
        Agent agent = SimpleAgent.builder()
            .modelClient(defaultModel)
            .tools(request.getToolNames().stream()
                .map(toolRegistry::getTool)
                .collect(Collectors.toList()))
            .memory(memory)
            .systemPrompt(request.getSystemPrompt())
            .temperature(request.getTemperature())
            .maxTokens(request.getMaxTokens())
            .build();
        
        String agentId = UUID.randomUUID().toString();
        agents.put(agentId, agent);
        
        return ResponseEntity.ok(agentId);
    }
}
```

## 17.7 에이전트 기능 확장

기본 에이전트 구현을 확장하고 커스터마이징하여 특수한 요구사항을 충족시키는 방법을 알아봅니다.

### 17.7.1 에이전트 데코레이터 패턴

```java
public class LoggingAgentDecorator implements Agent {
    
    private final Agent delegate;
    private final Logger logger = LoggerFactory.getLogger(LoggingAgentDecorator.class);
    
    public LoggingAgentDecorator(Agent delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public AgentResponse execute(AgentRequest request) {
        logger.info("Agent request: {}", request);
        AgentResponse response = delegate.execute(request);
        logger.info("Agent response: {}", response);
        return response;
    }
    
    // 기타 메서드 구현...
}
```

### 17.7.2 에이전트 기능 확장을 위한 인터셉터

```java
public interface AgentInterceptor {
    AgentRequest preHandle(AgentRequest request);
    AgentResponse postHandle(AgentRequest request, AgentResponse response);
    void afterCompletion(AgentRequest request, AgentResponse response, Exception ex);
}

@Component
public class RateLimitingInterceptor implements AgentInterceptor {
    
    private final RateLimiter rateLimiter;
    
    @Override
    public AgentRequest preHandle(AgentRequest request) {
        rateLimiter.acquire();
        return request;
    }
    
    // 기타 메서드 구현...
}
```

### 17.7.3 플러그인 기반 에이전트 확장

```java
public interface AgentPlugin {
    void initialize(Agent agent);
    void beforeExecution(AgentRequest request);
    void afterExecution(AgentResponse response);
    void shutdown();
}

@Component
public class AgentPluginManager {
    
    private final List<AgentPlugin> plugins;
    
    public void registerPlugin(AgentPlugin plugin) {
        plugins.add(plugin);
    }
    
    public void initializePlugins(Agent agent) {
        plugins.forEach(plugin -> plugin.initialize(agent));
    }
    
    // 기타 메서드...
}
```

## 17.8 에이전트의 실행 컨텍스트

에이전트가 작동하는 환경과 컨텍스트를 구성하고 관리하는 방법을 알아봅니다.

### 17.8.1 AgentContext 인터페이스

```java
public interface AgentContext {
    String getSessionId();
    List<Message> getHistory();
    Map<String, Object> getState();
    List<Tool> getAvailableTools();
    void addMessage(Message message);
    void setState(String key, Object value);
}
```

### 17.8.2 컨텍스트 관리 전략

```java
@Component
public class DefaultAgentContextManager implements AgentContextManager {
    
    private final Map<String, AgentContext> contexts = new ConcurrentHashMap<>();
    private final AgentMemory memory;
    
    @Override
    public AgentContext getContext(String sessionId) {
        return contexts.computeIfAbsent(sessionId, id -> {
            List<Message> history = memory.getMessages(id);
            return new DefaultAgentContext(id, history, memory);
        });
    }
    
    @Override
    public void clearContext(String sessionId) {
        contexts.remove(sessionId);
        memory.clear(sessionId);
    }
}
```

### 17.8.3 멀티 턴 대화 관리

```java
@Service
public class ConversationalAgent implements Agent {
    
    private final ModelClient modelClient;
    private final List<Tool> tools;
    private final AgentContextManager contextManager;
    
    @Override
    public AgentResponse execute(AgentRequest request) {
        String sessionId = request.getMetadata().getOrDefault("sessionId", "default").toString();
        AgentContext context = contextManager.getContext(sessionId);
        
        // 사용자 메시지를 컨텍스트에 추가
        context.addMessage(UserMessage.of(request.getPrompt()));
        
        // 모델에 전체 대화 컨텍스트 전달
        ChatRequest chatRequest = ChatRequest.builder()
            .messages(context.getHistory())
            .tools(tools)
            .build();
        
        ChatResponse chatResponse = modelClient.call(chatRequest);
        
        // 도구 실행 처리
        List<ToolExecution> toolExecutions = executeTools(chatResponse);
        
        // 결과 메시지 생성 및 컨텍스트에 추가
        Message assistantMessage = AssistantMessage.of(chatResponse.getResult());
        context.addMessage(assistantMessage);
        
        return AgentResponse.builder()
            .content(chatResponse.getResult())
            .messages(context.getHistory())
            .toolExecutions(toolExecutions)
            .build();
    }
    
    private List<ToolExecution> executeTools(ChatResponse response) {
        // 모델이 요청한 도구 실행 로직
    }
}
```

## 17.9 에이전트 채팅 인터페이스

Spring AI Agent API를 활용하여 대화형 인터페이스를 구축하는 방법을 알아봅니다.

### 17.9.1 채팅 컨트롤러 구현

```java
@RestController
@RequestMapping("/chat")
public class ChatController {
    
    private final Agent agent;
    private final AgentContextManager contextManager;
    
    @PostMapping("/{sessionId}")
    public ChatResponse chat(
            @PathVariable String sessionId,
            @RequestBody ChatRequest chatRequest) {
        
        AgentRequest agentRequest = AgentRequest.builder()
            .prompt(chatRequest.getMessage())
            .metadata(Map.of("sessionId", sessionId))
            .build();
        
        AgentResponse agentResponse = agent.execute(agentRequest);
        
        return ChatResponse.builder()
            .message(agentResponse.getContent())
            .toolResults(agentResponse.getToolExecutions().stream()
                .map(this::mapToolExecutionToResult)
                .collect(Collectors.toList()))
            .build();
    }
    
    @GetMapping("/{sessionId}/history")
    public List<MessageDto> getChatHistory(@PathVariable String sessionId) {
        AgentContext context = contextManager.getContext(sessionId);
        return context.getHistory().stream()
            .map(this::mapMessageToDto)
            .collect(Collectors.toList());
    }
    
    @DeleteMapping("/{sessionId}")
    public void clearChat(@PathVariable String sessionId) {
        contextManager.clearContext(sessionId);
    }
    
    // 매핑 메서드들...
}
```

### 17.9.2 WebSocket을 이용한 실시간 채팅

```java
@Controller
public class WebSocketChatController {
    
    private final Agent agent;
    private final AgentContextManager contextManager;
    
    @MessageMapping("/chat/{sessionId}")
    @SendTo("/topic/chat/{sessionId}")
    public ChatMessage chat(@DestinationVariable String sessionId, ChatMessage chatMessage) {
        AgentRequest agentRequest = AgentRequest.builder()
            .prompt(chatMessage.getContent())
            .metadata(Map.of("sessionId", sessionId))
            .build();
        
        AgentResponse agentResponse = agent.execute(agentRequest);
        
        return ChatMessage.builder()
            .sender("agent")
            .content(agentResponse.getContent())
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    @MessageMapping("/chat/{sessionId}/stream")
    @SendTo("/topic/chat/{sessionId}/stream")
    public Flux<ChatMessage> chatStream(@DestinationVariable String sessionId, ChatMessage chatMessage) {
        AgentRequest agentRequest = AgentRequest.builder()
            .prompt(chatMessage.getContent())
            .metadata(Map.of("sessionId", sessionId))
            .build();
        
        return agent.executeStreaming(agentRequest)
            .map(response -> ChatMessage.builder()
                .sender("agent")
                .content(response.getContent())
                .timestamp(LocalDateTime.now())
                .build());
    }
}
```

### 17.9.3 채팅 히스토리 시각화

```java
@RestController
@RequestMapping("/chat-visualization")
public class ChatVisualizationController {
    
    private final AgentContextManager contextManager;
    
    @GetMapping("/{sessionId}/graph")
    public ChatGraph generateChatGraph(@PathVariable String sessionId) {
        AgentContext context = contextManager.getContext(sessionId);
        List<Message> messages = context.getHistory();
        
        // 대화를 그래프 형태로 변환
        List<ChatNode> nodes = new ArrayList<>();
        List<ChatEdge> edges = new ArrayList<>();
        
        // 노드와 엣지 생성 로직...
        
        return new ChatGraph(nodes, edges);
    }
    
    @GetMapping("/{sessionId}/timeline")
    public ChatTimeline generateChatTimeline(@PathVariable String sessionId) {
        AgentContext context = contextManager.getContext(sessionId);
        List<Message> messages = context.getHistory();
        
        // 타임라인 생성 로직...
        
        return chatTimeline;
    }
}
```

## 17.10 실전 프로젝트: 지능형 고객 지원 에이전트

이 섹션에서는 Spring AI Agent API를 활용하여 실제 지능형 고객 지원 에이전트를 구현하는 방법을 살펴봅니다.

### 17.10.1 프로젝트 설정

```java
@SpringBootApplication
public class CustomerSupportAgentApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(CustomerSupportAgentApplication.class, args);
    }
    
    @Bean
    public ModelClient openaiModelClient() {
        // OpenAI 모델 클라이언트 구성
    }
    
    @Bean
    public VectorStore vectorStore() {
        // 제품 정보와 FAQ를 저장할 벡터 저장소
    }
}
```

### 17.10.2 도구 및 기능 구현

```java
@Service
public class CustomerSupportTools {
    
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    private final KnowledgeBaseService knowledgeBase;
    
    @Tool(name = "search_products", description = "Search for products in the catalog")
    @ToolParameters({
        @ToolParameter(name = "query", description = "Search query", required = true),
        @ToolParameter(name = "category", description = "Product category"),
        @ToolParameter(name = "max_results", description = "Maximum number of results", defaultValue = "5")
    })
    public List<ProductDto> searchProducts(String query, String category, int maxResults) {
        // 제품 검색 구현
    }
    
    @Tool(name = "check_order_status", description = "Check the status of an order")
    @ToolParameters({
        @ToolParameter(name = "order_id", description = "Order ID", required = true)
    })
    public OrderStatus checkOrderStatus(String orderId) {
        // 주문 상태 확인 구현
    }
    
    @Tool(name = "search_knowledge_base", description = "Search the knowledge base for information")
    @ToolParameters({
        @ToolParameter(name = "query", description = "Search query", required = true)
    })
    public List<KnowledgeArticle> searchKnowledgeBase(String query) {
        // 지식 베이스 검색 구현
    }
    
    @Tool(name = "escalate_to_human", description = "Escalate the conversation to a human support agent")
    @ToolParameters({
        @ToolParameter(name = "reason", description = "Reason for escalation", required = true),
        @ToolParameter(name = "priority", description = "Priority level", defaultValue = "medium")
    })
    public EscalationTicket escalateToHuman(String reason, String priority) {
        // 인간 상담원 에스컬레이션 구현
    }
}
```

### 17.10.3 고객 지원 에이전트 구현

```java
@Component
public class CustomerSupportAgent implements Agent {
    
    private final ModelClient modelClient;
    private final List<Tool> tools;
    private final AgentMemory memory;
    private final VectorStore knowledgeBase;
    
    @Override
    public AgentResponse execute(AgentRequest request) {
        String sessionId = request.getMetadata().getOrDefault("sessionId", "default").toString();
        
        // 대화 기록 가져오기
        List<Message> history = memory.getMessages(sessionId);
        
        // 관련 지식베이스 문서 검색
        List<Document> relevantDocs = retrieveRelevantDocuments(request.getPrompt());
        
        // 시스템 프롬프트 생성
        String systemPrompt = buildSystemPrompt(relevantDocs);
        
        // 채팅 요청 생성
        ChatRequest chatRequest = ChatRequest.builder()
            .systemPrompt(systemPrompt)
            .messages(history)
            .tools(tools)
            .build();
        
        // 모델 호출
        ChatResponse chatResponse = modelClient.call(chatRequest);
        
        // 도구 실행 처리
        List<ToolExecution> toolExecutions = handleToolExecutions(chatResponse);
        
        // 대화 기록 업데이트
        UserMessage userMessage = UserMessage.of(request.getPrompt());
        AssistantMessage assistantMessage = AssistantMessage.of(chatResponse.getResult());
        memory.saveMessage(sessionId, userMessage);
        memory.saveMessage(sessionId, assistantMessage);
        
        // 응답 생성
        return AgentResponse.builder()
            .content(chatResponse.getResult())
            .toolExecutions(toolExecutions)
            .build();
    }
    
    private List<Document> retrieveRelevantDocuments(String query) {
        // 벡터 저장소에서 관련 문서 검색
    }
    
    private String buildSystemPrompt(List<Document> relevantDocs) {
        // 시스템 프롬프트 구성
    }
    
    private List<ToolExecution> handleToolExecutions(ChatResponse chatResponse) {
        // 도구 실행 처리
    }
    
    // 기타 메서드 구현...
}
```

### 17.10.4 고객 지원 웹 인터페이스

```java
@Controller
public class CustomerSupportController {
    
    private final Agent customerSupportAgent;
    private final AgentContextManager contextManager;
    
    @GetMapping("/support")
    public String supportPage(Model model) {
        String sessionId = UUID.randomUUID().toString();
        model.addAttribute("sessionId", sessionId);
        return "support";
    }
    
    @MessageMapping("/support/{sessionId}")
    @SendTo("/topic/support/{sessionId}")
    public ChatMessage handleSupportMessage(
            @DestinationVariable String sessionId,
            ChatMessage userMessage) {
        
        AgentRequest agentRequest = AgentRequest.builder()
            .prompt(userMessage.getContent())
            .metadata(Map.of("sessionId", sessionId))
            .build();
        
        AgentResponse agentResponse = customerSupportAgent.execute(agentRequest);
        
        return ChatMessage.builder()
            .sender("agent")
            .content(agentResponse.getContent())
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

## 17.11 결론

이 장에서는 Spring AI Agent API의 핵심 구성요소, 아키텍처, 그리고 다양한 에이전트 구현 방법에 대해 살펴보았습니다. Agent API는 Spring AI의 강력한 기능을 활용하여 지능형 에이전트를 쉽게 구축하고 관리할 수 있는 일관된 인터페이스와 추상화를 제공합니다.

다음 장에서는 여러 에이전트를 연결하여 더 복잡한 작업을 수행할 수 있는 에이전트 체인과 워크플로 구축에 대해 알아보겠습니다.

## 연습 문제

1. Spring AI Agent API의 주요 인터페이스와 구성요소를 설명하고, 각각의 역할을 설명하세요.
2. @Tool 어노테이션을 사용해 데이터베이스 검색, 외부 API 호출, 파일 시스템 액세스 등의 기능을 수행하는 도구를 최소 3개 이상 구현해 보세요.
3. 에이전트 메모리를 확장하여 대화 요약 기능을 추가하는 방법을 설계하고 구현해 보세요.
4. AgentInterceptor 인터페이스를 구현하여 에이전트 요청과 응답을 로깅하고 모니터링하는 기능을 추가해 보세요.
5. Spring AI Agent API를 사용하여 특정 도메인(예: 여행 계획, 건강 코칭, 학습 도우미 등)에 특화된 에이전트를 설계하고 구현해 보세요.