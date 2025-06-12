# 17장: Spring AI Agent API

## 17.1 Spring AI Agent API 개요

Spring AI는 AI 에이전트를 구축하기 위한 포괄적인 API를 제공합니다. 최신 Spring AI는 Advisors API, Tool Calling, ChatMemory 등의 강력한 기능을 통해 정교한 AI 에이전트 시스템을 구축할 수 있도록 지원합니다.

### 17.1.1 Spring AI의 에이전트 접근 방식

Spring AI는 Anthropic의 "Building Effective Agents" 연구에서 제시된 원칙을 따릅니다:

**1. 워크플로우 vs 에이전트**
- **워크플로우**: 사전 정의된 코드 경로를 통해 LLM과 도구를 오케스트레이션하는 시스템
- **에이전트**: LLM이 동적으로 자신의 프로세스와 도구 사용을 지시하는 시스템

**2. 핵심 설계 원칙**
- 단순성과 구성 가능성 우선
- 복잡한 프레임워크보다 명확한 패턴 선호
- 예측 가능성과 일관성 중시
- 모델 독립적인 구현

### 17.1.2 Spring AI Agent 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                    Spring AI Agent API                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │  Advisors   │  │ Tool Calling │  │  Chat Memory  │ │
│  │     API     │  │     API      │  │      API      │ │
│  └──────┬──────┘  └──────┬───────┘  └───────┬───────┘ │
│         │                 │                   │         │
│  ┌──────▼────────────────▼───────────────────▼───────┐ │
│  │               ChatClient API                       │ │
│  └──────────────────────┬─────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼─────────────────────────────┐ │
│  │              ChatModel Implementations              │ │
│  │  (OpenAI, Anthropic, Google, Ollama, etc.)        │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## 17.2 Advisors API: 에이전트의 핵심 기반

Spring AI의 Advisors API는 AI 기반 상호작용을 가로채고, 수정하고, 향상시킬 수 있는 유연하고 강력한 방법을 제공합니다.

### 17.2.1 Advisor의 핵심 개념

**Advisor란?**
- AI 모델과의 상호작용을 가로채는 인터셉터
- 요청과 응답을 수정하거나 향상시킬 수 있음
- 재사용 가능한 AI 패턴을 캡슐화
- 모델 간 이식성 제공

### 17.2.2 Advisor 인터페이스

```java
public interface Advisor extends Ordered {
    String getName();
}

// 동기식 Advisor
public interface CallAroundAdvisor extends Advisor {
    AdvisedResponse aroundCall(AdvisedRequest advisedRequest, CallAroundAdvisorChain chain);
}

// 비동기식 Advisor
public interface StreamAroundAdvisor extends Advisor {
    Flux<AdvisedResponse> aroundStream(AdvisedRequest advisedRequest, StreamAroundAdvisorChain chain);
}
```

### 17.2.3 Advisor 체인 아키텍처

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Advisor 1  │────▶│   Advisor 2  │────▶│   Advisor N  │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────┐
│                    Chat Model                         │
└──────────────────────────────────────────────────────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Advisor N  │◀────│   Advisor 2  │◀────│   Advisor 1  │
└──────────────┘     └──────────────┘     └──────────────┘
```

### 17.2.4 기본 제공 Advisors

Spring AI는 여러 내장 Advisor를 제공합니다:

**1. 채팅 메모리 Advisors**
```java
// 메시지 기반 메모리 관리
MessageChatMemoryAdvisor memoryAdvisor = MessageChatMemoryAdvisor.builder()
    .chatMemory(chatMemory)
    .build();

// 프롬프트 기반 메모리 관리
PromptChatMemoryAdvisor promptAdvisor = new PromptChatMemoryAdvisor(chatMemory);

// 벡터 저장소 기반 메모리
VectorStoreChatMemoryAdvisor vectorAdvisor = new VectorStoreChatMemoryAdvisor(
    vectorStore,
    "conversation-id",
    10
);
```

**2. RAG (Retrieval-Augmented Generation) Advisor**
```java
QuestionAnswerAdvisor ragAdvisor = QuestionAnswerAdvisor.builder()
    .vectorStore(vectorStore)
    .searchRequest(SearchRequest.defaults()
        .withTopK(5)
        .withSimilarityThreshold(0.7))
    .build();
```

**3. 안전성 Advisor**
```java
SafeGuardAdvisor safetyAdvisor = new SafeGuardAdvisor();
```

### 17.2.5 커스텀 Advisor 구현 예제

**로깅 Advisor**
```java
public class LoggingAdvisor implements CallAroundAdvisor, StreamAroundAdvisor {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingAdvisor.class);
    
    @Override
    public String getName() {
        return "LoggingAdvisor";
    }
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE; // 가장 먼저 실행
    }
    
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        logger.debug("요청: {}", request);
        AdvisedResponse response = chain.nextAroundCall(request);
        logger.debug("응답: {}", response);
        return response;
    }
    
    @Override
    public Flux<AdvisedResponse> aroundStream(AdvisedRequest request, StreamAroundAdvisorChain chain) {
        logger.debug("스트리밍 요청: {}", request);
        return chain.nextAroundStream(request)
            .doOnNext(response -> logger.debug("스트리밍 응답: {}", response));
    }
}
```

**Re-Reading (Re2) Advisor**
```java
public class ReReadingAdvisor implements CallAroundAdvisor {
    
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        // Re2 기법 적용: 질문을 두 번 읽도록 프롬프트 수정
        Map<String, Object> params = new HashMap<>(request.userParams());
        params.put("original_query", request.userText());
        
        AdvisedRequest modifiedRequest = AdvisedRequest.from(request)
            .userText("""
                {original_query}
                질문을 다시 읽어주세요: {original_query}
                """)
            .userParams(params)
            .build();
            
        return chain.nextAroundCall(modifiedRequest);
    }
    
    @Override
    public String getName() {
        return "ReReadingAdvisor";
    }
    
    @Override
    public int getOrder() {
        return 0;
    }
}
```

### 17.2.6 ChatClient와 Advisor 통합

```java
@Configuration
public class AgentConfiguration {
    
    @Bean
    public ChatClient chatClient(ChatModel chatModel, ChatMemory chatMemory) {
        return ChatClient.builder(chatModel)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build(),
                new LoggingAdvisor(),
                new ReReadingAdvisor()
            )
            .build();
    }
    
    @Bean
    public ChatMemory chatMemory() {
        return MessageWindowChatMemory.builder()
            .maxMessages(20)
            .build();
    }
}

// 사용 예제
@Service
public class ConversationalAgent {
    
    private final ChatClient chatClient;
    
    public String chat(String conversationId, String userMessage) {
        return chatClient.prompt()
            .advisors(advisor -> advisor
                .param(ChatMemory.CONVERSATION_ID, conversationId))
            .user(userMessage)
            .call()
            .content();
    }
}
```

## 17.3 Tool Calling API: 에이전트의 도구 상자

Tool Calling은 AI 모델이 외부 도구나 API와 상호작용할 수 있게 해주는 Spring AI의 핵심 기능입니다.

### 17.3.1 Tool Calling 개요

**도구의 두 가지 주요 용도:**
1. **정보 검색**: 데이터베이스, 웹 서비스, 파일 시스템 등에서 정보 검색
2. **작업 수행**: 이메일 전송, 데이터베이스 레코드 생성, 워크플로 트리거 등

### 17.3.2 @Tool 어노테이션을 사용한 도구 정의

```java
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;

@Service
public class WeatherTools {
    
    @Tool(description = "현재 날씨 정보를 가져옵니다")
    public WeatherInfo getCurrentWeather(
            @ToolParam(description = "도시 이름") String city,
            @ToolParam(description = "온도 단위 (C/F)", required = false) String unit) {
        
        // 외부 날씨 API 호출
        return weatherService.getWeather(city, unit != null ? unit : "C");
    }
    
    @Tool(description = "날씨 예보를 가져옵니다")
    public List<WeatherForecast> getWeatherForecast(
            @ToolParam(description = "도시 이름") String city,
            @ToolParam(description = "예보 일수") int days) {
        
        return weatherService.getForecast(city, days);
    }
}
```

### 17.3.3 프로그래매틱 도구 정의

```java
// Function 기반 도구
public class DatabaseQueryTool implements Function<QueryRequest, QueryResult> {
    
    @Override
    public QueryResult apply(QueryRequest request) {
        return databaseService.executeQuery(request.getSql());
    }
}

// FunctionToolCallback으로 등록
ToolCallback databaseTool = FunctionToolCallback.builder()
    .name("query_database")
    .description("데이터베이스에 SQL 쿼리를 실행합니다")
    .inputType(QueryRequest.class)
    .toolFunction(new DatabaseQueryTool())
    .build();
```

### 17.3.4 도구 컨텍스트와 실행

```java
@Service
public class ContextAwareTools {
    
    @Tool(description = "사용자 정보를 조회합니다")
    public UserInfo getUserInfo(String userId, ToolContext toolContext) {
        // ToolContext에서 추가 정보 접근
        String tenantId = (String) toolContext.getContext().get("tenantId");
        return userService.getUser(tenantId, userId);
    }
}

// 사용 예제
String response = chatClient.prompt()
    .user("사용자 ID 12345의 정보를 알려주세요")
    .tools(new ContextAwareTools())
    .toolContext(Map.of("tenantId", "tenant-001"))
    .call()
    .content();
```

### 17.3.5 도구 실행 제어

```java
// Return Direct: 도구 결과를 직접 반환
@Tool(description = "FAQ 검색", returnDirect = true)
public String searchFAQ(String query) {
    // 결과가 모델을 거치지 않고 직접 사용자에게 반환됨
    return faqService.search(query);
}

// 커스텀 결과 변환
@Tool(description = "차트 생성", resultConverter = ChartResultConverter.class)
public Chart generateChart(ChartRequest request) {
    return chartService.generate(request);
}
```

## 17.4 에이전트 패턴: 효과적인 에이전트 구축

Spring AI는 Anthropic의 연구를 기반으로 5가지 핵심 에이전트 패턴을 제공합니다.

### 17.4.1 Chain Workflow 패턴

**언제 사용하나?**
- 명확한 순차적 단계가 있는 작업
- 정확성을 위해 지연 시간을 감수할 수 있을 때
- 각 단계가 이전 단계의 출력을 기반으로 할 때

```java
@Component
public class ChainWorkflowAgent {
    
    private final ChatClient chatClient;
    
    public String processChain(String input) {
        // 단계 1: 텍스트 분석
        String analysis = chatClient.prompt()
            .system("주어진 텍스트를 분석하고 주요 포인트를 추출하세요.")
            .user(input)
            .call()
            .content();
            
        // 단계 2: 요약 생성
        String summary = chatClient.prompt()
            .system("분석 결과를 바탕으로 간결한 요약을 작성하세요.")
            .user(analysis)
            .call()
            .content();
            
        // 단계 3: 실행 계획 수립
        String actionPlan = chatClient.prompt()
            .system("요약을 바탕으로 구체적인 실행 계획을 수립하세요.")
            .user(summary)
            .call()
            .content();
            
        return actionPlan;
    }
}
```

### 17.4.2 Parallelization Workflow 패턴

**언제 사용하나?**
- 독립적인 작업들을 동시에 처리해야 할 때
- 여러 관점에서의 분석이 필요할 때
- 처리 시간이 중요하고 작업이 병렬화 가능할 때

```java
@Component
public class ParallelizationWorkflowAgent {
    
    private final ChatClient chatClient;
    private final ExecutorService executor = Executors.newFixedThreadPool(4);
    
    public Map<String, String> analyzeStakeholders(String businessChange) {
        List<String> stakeholders = List.of(
            "고객", "직원", "투자자", "공급업체"
        );
        
        Map<String, CompletableFuture<String>> futures = new HashMap<>();
        
        for (String stakeholder : stakeholders) {
            CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> 
                chatClient.prompt()
                    .system("비즈니스 변화가 " + stakeholder + "에게 미치는 영향을 분석하세요.")
                    .user(businessChange)
                    .call()
                    .content(),
                executor
            );
            futures.put(stakeholder, future);
        }
        
        // 모든 분석 결과 수집
        Map<String, String> results = new HashMap<>();
        futures.forEach((stakeholder, future) -> {
            try {
                results.put(stakeholder, future.get());
            } catch (Exception e) {
                results.put(stakeholder, "분석 실패: " + e.getMessage());
            }
        });
        
        return results;
    }
}
```

### 17.4.3 Routing Workflow 패턴

**언제 사용하나?**
- 입력이 명확한 카테고리로 분류될 때
- 각 카테고리가 특화된 처리를 필요로 할 때
- 정확한 분류가 가능할 때

```java
@Component
public class RoutingWorkflowAgent {
    
    private final ChatClient chatClient;
    
    record RouteClassification(String category, String reasoning) {}
    
    public String route(String userInput) {
        // 입력 분류
        RouteClassification classification = chatClient.prompt()
            .system("""
                사용자 입력을 다음 카테고리 중 하나로 분류하세요:
                - billing: 결제 관련 문의
                - technical: 기술 지원 문의
                - general: 일반 문의
                """)
            .user(userInput)
            .call()
            .entity(RouteClassification.class);
            
        // 카테고리별 전문 처리
        String systemPrompt = switch (classification.category()) {
            case "billing" -> "당신은 결제 전문가입니다. 결제 문제를 해결하세요.";
            case "technical" -> "당신은 기술 지원 엔지니어입니다. 기술 문제를 해결하세요.";
            default -> "당신은 고객 서비스 담당자입니다. 친절하게 답변하세요.";
        };
        
        return chatClient.prompt()
            .system(systemPrompt)
            .user(userInput)
            .call()
            .content();
    }
}
```

### 17.4.4 Orchestrator-Workers 패턴

**언제 사용하나?**
- 하위 작업을 미리 예측할 수 없는 복잡한 작업
- 다양한 접근 방식이나 관점이 필요한 작업
- 적응적 문제 해결이 필요한 상황

```java
@Component
public class OrchestratorWorkersAgent {
    
    private final ChatClient chatClient;
    
    record TaskDecomposition(List<String> subtasks, String strategy) {}
    
    public String orchestrate(String complexTask) {
        // 오케스트레이터: 작업 분해
        TaskDecomposition decomposition = chatClient.prompt()
            .system("복잡한 작업을 하위 작업으로 분해하고 전략을 수립하세요.")
            .user(complexTask)
            .call()
            .entity(TaskDecomposition.class);
            
        // 워커: 각 하위 작업 수행
        List<String> results = decomposition.subtasks().parallelStream()
            .map(subtask -> chatClient.prompt()
                .system("다음 하위 작업을 수행하세요: " + subtask)
                .user(complexTask)
                .call()
                .content())
            .toList();
            
        // 오케스트레이터: 결과 통합
        return chatClient.prompt()
            .system("하위 작업 결과들을 통합하여 최종 결과를 생성하세요.")
            .user("원래 작업: " + complexTask + "\n\n결과들: " + String.join("\n", results))
            .call()
            .content();
    }
}
```

### 17.4.5 Evaluator-Optimizer 패턴

**언제 사용하나?**
- 명확한 평가 기준이 존재할 때
- 반복적 개선이 측정 가능한 가치를 제공할 때
- 여러 라운드의 비평이 도움이 되는 작업

```java
@Component
public class EvaluatorOptimizerAgent {
    
    private final ChatClient chatClient;
    private static final int MAX_ITERATIONS = 3;
    
    record Evaluation(double score, List<String> issues, List<String> suggestions) {}
    
    public String optimizeWithEvaluation(String task) {
        String currentSolution = generateInitialSolution(task);
        
        for (int i = 0; i < MAX_ITERATIONS; i++) {
            // 평가
            Evaluation evaluation = chatClient.prompt()
                .system("솔루션을 평가하고 개선점을 제시하세요.")
                .user("작업: " + task + "\n\n솔루션: " + currentSolution)
                .call()
                .entity(Evaluation.class);
                
            if (evaluation.score() >= 0.9) {
                break; // 충분히 좋은 솔루션
            }
            
            // 최적화
            currentSolution = chatClient.prompt()
                .system("평가 결과를 바탕으로 솔루션을 개선하세요.")
                .user("원래 솔루션: " + currentSolution + 
                      "\n\n문제점: " + String.join(", ", evaluation.issues()) +
                      "\n\n제안사항: " + String.join(", ", evaluation.suggestions()))
                .call()
                .content();
        }
        
        return currentSolution;
    }
    
    private String generateInitialSolution(String task) {
        return chatClient.prompt()
            .system("주어진 작업에 대한 초기 솔루션을 생성하세요.")
            .user(task)
            .call()
            .content();
    }
}
```

## 17.5 ChatMemory API: 대화형 에이전트의 메모리

Spring AI의 ChatMemory API는 에이전트가 대화 컨텍스트를 유지할 수 있게 해줍니다.

### 17.5.1 ChatMemory 개념

**ChatMemory vs ChatHistory**
- **ChatMemory**: LLM이 컨텍스트 인식을 유지하기 위해 보존하는 정보
- **ChatHistory**: 사용자와 모델 간 전체 대화 기록

### 17.5.2 메모리 타입

**1. MessageWindowChatMemory**
```java
@Bean
public ChatMemory messageWindowMemory() {
    return MessageWindowChatMemory.builder()
        .maxMessages(10)  // 최대 10개 메시지 유지
        .build();
}
```

**2. TokenWindowChatMemory (향후 지원 예정)**
```java
@Bean
public ChatMemory tokenWindowMemory() {
    return TokenWindowChatMemory.builder()
        .maxTokens(2000)  // 최대 2000 토큰 유지
        .build();
}
```

### 17.5.3 메모리 저장소

**1. In-Memory 저장소**
```java
@Bean
public ChatMemoryRepository inMemoryRepository() {
    return new InMemoryChatMemoryRepository();
}
```

**2. JDBC 저장소**
```java
@Bean
public JdbcChatMemoryRepository jdbcRepository(JdbcTemplate jdbcTemplate) {
    return JdbcChatMemoryRepository.builder()
        .jdbcTemplate(jdbcTemplate)
        .dialect(new PostgresChatMemoryDialect())
        .build();
}
```

**3. 벡터 저장소 기반 메모리**
```java
@Component
public class VectorStoreMemoryAdvisor extends VectorStoreChatMemoryAdvisor {
    
    public VectorStoreMemoryAdvisor(VectorStore vectorStore) {
        super(vectorStore, "", 10);
    }
    
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        String conversationId = request.advisorContext()
            .getOrDefault(CONVERSATION_ID, "default").toString();
            
        // 관련 기억 검색
        List<Document> relevantMemories = searchRelevantMemories(
            conversationId, 
            request.userText()
        );
        
        // 컨텍스트에 추가
        String augmentedPrompt = buildAugmentedPrompt(
            request.userText(), 
            relevantMemories
        );
        
        AdvisedRequest augmentedRequest = AdvisedRequest.from(request)
            .userText(augmentedPrompt)
            .build();
            
        return chain.nextAroundCall(augmentedRequest);
    }
}
```

### 17.5.4 대화형 에이전트 구현

```java
@Service
public class ConversationalAgent {
    
    private final ChatClient chatClient;
    private final ChatMemory chatMemory;
    
    public ConversationalAgent(ChatModel chatModel, ChatMemory chatMemory) {
        this.chatMemory = chatMemory;
        this.chatClient = ChatClient.builder(chatModel)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .systemTextAdvise("""
                        다음은 이전 대화 내용입니다:
                        {memory}
                        
                        위 대화를 참고하여 사용자의 질문에 답변하세요.
                        """)
                    .build()
            )
            .build();
    }
    
    public String chat(String conversationId, String userMessage) {
        return chatClient.prompt()
            .advisors(advisor -> advisor
                .param(ChatMemory.CONVERSATION_ID, conversationId))
            .user(userMessage)
            .call()
            .content();
    }
    
    public void clearConversation(String conversationId) {
        chatMemory.clear(conversationId);
    }
    
    public List<Message> getConversationHistory(String conversationId) {
        return chatMemory.get(conversationId);
    }
}
```

## 17.6 복합 에이전트 구현 예제

실제 비즈니스 시나리오에서 Spring AI의 여러 기능을 조합한 복합 에이전트 구현 예제를 살펴봅니다.

### 17.6.1 RAG 기반 고객 지원 에이전트

```java
@Configuration
public class CustomerSupportAgentConfig {
    
    @Bean
    public ChatClient customerSupportChatClient(
            ChatModel chatModel,
            VectorStore vectorStore,
            ChatMemory chatMemory) {
        
        return ChatClient.builder(chatModel)
            .defaultAdvisors(
                // RAG Advisor: 지식베이스 검색
                QuestionAnswerAdvisor.builder(vectorStore)
                    .searchRequest(SearchRequest.defaults()
                        .withTopK(5)
                        .withSimilarityThreshold(0.7))
                    .userTextAdvise("""
                        고객 문의: {question}
                        
                        관련 정보:
                        {question_answer_context}
                        """)
                    .build(),
                    
                // Memory Advisor: 대화 기록 유지
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .build(),
                    
                // Safety Advisor: 안전한 응답 보장
                new SafeGuardAdvisor()
            )
            .defaultSystem("""
                당신은 친절하고 전문적인 고객 지원 상담원입니다.
                항상 정확하고 도움이 되는 답변을 제공하세요.
                """)
            .build();
    }
}

@Service
public class CustomerSupportAgent {
    
    private final ChatClient chatClient;
    private final CustomerTools customerTools;
    
    public String handleCustomerQuery(String conversationId, String query) {
        return chatClient.prompt()
            .advisors(advisor -> advisor
                .param(ChatMemory.CONVERSATION_ID, conversationId))
            .user(query)
            .tools(customerTools)
            .call()
            .content();
    }
}

@Component
public class CustomerTools {
    
    @Tool(description = "주문 상태를 확인합니다")
    public OrderStatus checkOrderStatus(
            @ToolParam(description = "주문 번호") String orderId,
            ToolContext context) {
        
        String customerId = (String) context.getContext().get("customerId");
        return orderService.getOrderStatus(customerId, orderId);
    }
    
    @Tool(description = "반품 요청을 처리합니다")
    public ReturnRequest processReturn(
            @ToolParam(description = "주문 번호") String orderId,
            @ToolParam(description = "반품 사유") String reason) {
        
        return returnService.createReturnRequest(orderId, reason);
    }
}
```

### 17.6.2 코드 생성 에이전트

```java
@Service
public class CodeGenerationAgent {
    
    private final ChatClient chatClient;
    private final EvaluatorOptimizerAgent optimizer;
    
    public CodeGenerationAgent(ChatModel chatModel) {
        this.chatClient = ChatClient.builder(chatModel)
            .defaultSystem("""
                당신은 전문 Java 개발자입니다.
                깨끗하고 효율적이며 잘 문서화된 코드를 작성하세요.
                """)
            .build();
        this.optimizer = new EvaluatorOptimizerAgent(chatModel);
    }
    
    public GeneratedCode generateCode(CodeGenerationRequest request) {
        // 1. 초기 코드 생성
        String initialCode = chatClient.prompt()
            .user("""
                다음 요구사항에 맞는 Java 클래스를 작성하세요:
                {requirements}
                
                다음 테스트를 통과해야 합니다:
                {testCode}
                """,
                Map.of(
                    "requirements", request.getRequirements(),
                    "testCode", request.getTestCode()
                ))
            .call()
            .content();
            
        // 2. 코드 최적화 및 개선
        String optimizedCode = optimizer.optimizeWithEvaluation(
            "코드 품질 개선: " + initialCode
        );
        
        // 3. 문서화 추가
        String documentedCode = chatClient.prompt()
            .system("코드에 JavaDoc과 인라인 주석을 추가하세요.")
            .user(optimizedCode)
            .tools(new CodeAnalysisTools())
            .call()
            .content();
            
        return new GeneratedCode(documentedCode, extractClassName(documentedCode));
    }
}

@Component
public class CodeAnalysisTools {
    
    @Tool(description = "코드 복잡도를 분석합니다")
    public ComplexityReport analyzeComplexity(String code) {
        return codeAnalyzer.analyze(code);
    }
    
    @Tool(description = "코드 스타일을 검사합니다")
    public List<StyleIssue> checkStyle(String code) {
        return styleChecker.check(code);
    }
}
```

### 17.6.3 데이터 분석 에이전트

```java
@Service
public class DataAnalysisAgent {
    
    private final ChatClient chatClient;
    private final ParallelizationWorkflowAgent parallelAgent;
    
    public DataAnalysisAgent(ChatModel chatModel, VectorStore vectorStore) {
        this.chatClient = ChatClient.builder(chatModel)
            .defaultAdvisors(
                // 이전 분석 결과 참조
                new VectorStoreChatMemoryAdvisor(vectorStore, "", 5)
            )
            .defaultSystem("""
                당신은 데이터 분석 전문가입니다.
                통계적 근거를 바탕으로 인사이트를 제공하세요.
                """)
            .build();
        this.parallelAgent = new ParallelizationWorkflowAgent(chatModel);
    }
    
    public AnalysisReport analyzeDataset(DatasetInfo dataset) {
        // 1. 병렬로 여러 관점에서 분석
        Map<String, String> perspectives = parallelAgent.analyzeStakeholders(
            "데이터셋: " + dataset.getDescription()
        );
        
        // 2. 통합 분석 수행
        String comprehensiveAnalysis = chatClient.prompt()
            .user("""
                다음 데이터셋에 대한 종합 분석을 수행하세요:
                {datasetInfo}
                
                다양한 관점의 분석:
                {perspectives}
                """)
            .tools(new DataAnalysisTools())
            .stream()
            .content()
            .collectList()
            .block()
            .stream()
            .collect(Collectors.joining());
            
        // 3. 시각화 추천
        List<VisualizationRecommendation> visualizations = 
            chatClient.prompt()
                .user("분석 결과를 바탕으로 적절한 시각화를 추천하세요: " + comprehensiveAnalysis)
                .call()
                .entity(new ParameterizedTypeReference<List<VisualizationRecommendation>>() {});
                
        return new AnalysisReport(
            comprehensiveAnalysis,
            perspectives,
            visualizations
        );
    }
}

@Component
public class DataAnalysisTools {
    
    @Tool(description = "SQL 쿼리를 실행하여 데이터를 조회합니다")
    public QueryResult executeQuery(
            @ToolParam(description = "SQL 쿼리") String sql) {
        return dataService.executeQuery(sql);
    }
    
    @Tool(description = "통계 분석을 수행합니다")
    public StatisticsResult calculateStatistics(
            @ToolParam(description = "데이터셋 ID") String datasetId,
            @ToolParam(description = "분석 유형") String analysisType) {
        return statisticsService.analyze(datasetId, analysisType);
    }
    
    @Tool(description = "차트를 생성합니다", returnDirect = true)
    public ChartResult generateChart(
            @ToolParam(description = "차트 유형") String chartType,
            @ToolParam(description = "데이터") Map<String, Object> data) {
        return chartService.create(chartType, data);
    }
}
```

## 17.7 에이전트 관찰성과 모니터링

Spring AI는 에이전트의 동작을 모니터링하고 디버깅할 수 있는 포괄적인 관찰성 기능을 제공합니다.

### 17.7.1 Advisor 실행 추적

```java
@Component
public class ObservabilityAdvisor implements CallAroundAdvisor {
    
    private final MeterRegistry meterRegistry;
    private final Tracer tracer;
    
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        // 메트릭 수집
        Timer.Sample sample = Timer.start(meterRegistry);
        
        // 분산 추적
        Span span = tracer.nextSpan()
            .name("ai.advisor." + getName())
            .tag("ai.model", request.chatModel().getDefaultOptions().getModel())
            .tag("ai.conversation.id", getConversationId(request))
            .start();
            
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            AdvisedResponse response = chain.nextAroundCall(request);
            
            // 토큰 사용량 기록
            if (response.response() != null) {
                Usage usage = response.response().getMetadata().getUsage();
                meterRegistry.counter("ai.tokens.prompt").increment(usage.getPromptTokens());
                meterRegistry.counter("ai.tokens.completion").increment(usage.getGenerationTokens());
            }
            
            return response;
        } finally {
            span.end();
            sample.stop(Timer.builder("ai.advisor.duration")
                .tag("advisor", getName())
                .register(meterRegistry));
        }
    }
    
    @Override
    public String getName() {
        return "ObservabilityAdvisor";
    }
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

### 17.7.2 도구 실행 모니터링

```java
@Aspect
@Component
public class ToolExecutionMonitor {
    
    private final MeterRegistry meterRegistry;
    private final Logger logger = LoggerFactory.getLogger(ToolExecutionMonitor.class);
    
    @Around("@annotation(tool)")
    public Object monitorToolExecution(ProceedingJoinPoint joinPoint, Tool tool) throws Throwable {
        String toolName = tool.name().isEmpty() ? joinPoint.getSignature().getName() : tool.name();
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            logger.debug("도구 실행 시작: {}", toolName);
            Object result = joinPoint.proceed();
            
            meterRegistry.counter("ai.tool.executions.success", "tool", toolName).increment();
            logger.debug("도구 실행 성공: {}", toolName);
            
            return result;
            
        } catch (Exception e) {
            meterRegistry.counter("ai.tool.executions.error", 
                "tool", toolName,
                "error", e.getClass().getSimpleName()).increment();
            logger.error("도구 실행 실패: {}", toolName, e);
            throw e;
            
        } finally {
            sample.stop(Timer.builder("ai.tool.execution.duration")
                .tag("tool", toolName)
                .register(meterRegistry));
        }
    }
}
```

### 17.7.3 에이전트 대시보드 구현

```java
@RestController
@RequestMapping("/api/agent-metrics")
public class AgentMetricsController {
    
    private final MeterRegistry meterRegistry;
    private final ChatMemoryRepository memoryRepository;
    
    @GetMapping("/summary")
    public AgentMetricsSummary getMetricsSummary() {
        return AgentMetricsSummary.builder()
            .totalRequests(getCounterValue("ai.advisor.duration"))
            .successRate(calculateSuccessRate())
            .averageResponseTime(getTimerAverage("ai.advisor.duration"))
            .tokenUsage(TokenUsage.builder()
                .promptTokens(getCounterValue("ai.tokens.prompt"))
                .completionTokens(getCounterValue("ai.tokens.completion"))
                .build())
            .activeConversations(countActiveConversations())
            .build();
    }
    
    @GetMapping("/tools")
    public List<ToolMetrics> getToolMetrics() {
        return meterRegistry.getMeters().stream()
            .filter(meter -> meter.getId().getName().startsWith("ai.tool"))
            .collect(Collectors.groupingBy(meter -> meter.getId().getTag("tool")))
            .entrySet().stream()
            .map(entry -> buildToolMetrics(entry.getKey(), entry.getValue()))
            .collect(Collectors.toList());
    }
    
    @GetMapping("/conversations/{conversationId}/trace")
    public ConversationTrace getConversationTrace(@PathVariable String conversationId) {
        List<Message> messages = memoryRepository.getMessages(conversationId);
        return buildConversationTrace(messages);
    }
}
```

### 17.7.4 비용 추적 Advisor

```java
@Component
public class CostTrackingAdvisor implements CallAroundAdvisor {
    
    private final Map<String, BigDecimal> modelPricing = Map.of(
        "gpt-4", new BigDecimal("0.03"),  // per 1K tokens
        "gpt-3.5-turbo", new BigDecimal("0.001"),
        "claude-3-opus", new BigDecimal("0.015")
    );
    
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        AdvisedResponse response = chain.nextAroundCall(request);
        
        if (response.response() != null) {
            Usage usage = response.response().getMetadata().getUsage();
            String model = request.chatModel().getDefaultOptions().getModel();
            
            BigDecimal cost = calculateCost(model, usage);
            
            // 비용을 컨텍스트에 저장
            response = AdvisedResponse.from(response)
                .adviseContext(context -> {
                    context.put("cost", cost);
                    context.put("totalCost", 
                        ((BigDecimal) context.getOrDefault("totalCost", BigDecimal.ZERO)).add(cost));
                    return context;
                })
                .build();
                
            logger.info("AI 호출 비용: ${} (모델: {}, 토큰: {})", 
                cost, model, usage.getTotalTokens());
        }
        
        return response;
    }
    
    private BigDecimal calculateCost(String model, Usage usage) {
        BigDecimal pricePerKToken = modelPricing.getOrDefault(model, BigDecimal.ZERO);
        return pricePerKToken.multiply(
            new BigDecimal(usage.getTotalTokens()).divide(new BigDecimal(1000))
        );
    }
    
    @Override
    public String getName() {
        return "CostTrackingAdvisor";
    }
    
    @Override
    public int getOrder() {
        return 100;
    }
}
```

## 17.8 MCP (Model Context Protocol) 통합

Spring AI는 Model Context Protocol을 통해 표준화된 도구 및 리소스 상호작용을 지원합니다.

### 17.8.1 MCP 개요

MCP는 AI 모델이 외부 도구 및 리소스와 상호작용하기 위한 표준화된 프로토콜입니다.

```
┌────────────────────────────────────────────────┐
│              Spring AI MCP Integration          │
├────────────────────────────────────────────────┤
│                                                │
│  ┌──────────────┐        ┌──────────────────┐ │
│  │  MCP Client  │◀──────▶│    MCP Server    │ │
│  │   Starter    │        │     Starter      │ │
│  └──────┬───────┘        └────────┬─────────┘ │
│         │                         │            │
│  ┌──────▼───────────────────────▼──────────┐  │
│  │          MCP Java SDK                    │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  Transport: STDIO, HTTP SSE, WebFlux SSE      │
└────────────────────────────────────────────────┘
```

### 17.8.2 MCP Client 구성

```java
@Configuration
public class McpClientConfig {
    
    @Bean
    public McpSyncClient mcpClient() {
        // STDIO 전송 사용
        StdioClientTransport transport = StdioClientTransport.of(
            "python", "-m", "mcp_server"
        );
        
        return McpClient.sync(transport);
    }
    
    @Bean
    public McpToolProvider mcpToolProvider(McpSyncClient mcpClient) {
        return new McpToolProvider(mcpClient);
    }
}

// MCP 도구를 Spring AI 도구로 변환
@Component
public class McpToolAdapter {
    
    private final McpSyncClient mcpClient;
    
    public List<ToolCallback> getMcpTools() {
        InitializeResult init = mcpClient.initialize(
            new ClientInfo("spring-ai-agent", "1.0.0")
        );
        
        ListToolsResult tools = mcpClient.listTools();
        
        return tools.tools().stream()
            .map(this::convertToSpringAiTool)
            .collect(Collectors.toList());
    }
    
    private ToolCallback convertToSpringAiTool(Tool mcpTool) {
        return FunctionToolCallback.builder()
            .name(mcpTool.name())
            .description(mcpTool.description())
            .inputType(Map.class)
            .toolFunction(params -> {
                CallToolResult result = mcpClient.callTool(
                    new CallToolRequest(mcpTool.name(), params)
                );
                return result.content().stream()
                    .map(Content::text)
                    .collect(Collectors.joining("\n"));
            })
            .build();
    }
}
```

### 17.8.3 MCP Server 구현

```java
@Configuration
@EnableMcpServer
public class McpServerConfig {
    
    @Bean
    public McpServer<SseServerTransport> mcpServer() {
        return McpServer.builder()
            .serverInfo(new Implementation(
                "spring-ai-mcp-server",
                "1.0.0"
            ))
            .transport(SseServerTransport.create("/mcp"))
            .build();
    }
}

@Component
public class WeatherMcpTools {
    
    @McpTool(description = "Get current weather for a location")
    public ToolContent getWeather(
            @ToolParam(description = "City name", required = true) String city) {
        
        WeatherInfo weather = weatherService.getWeather(city);
        return ToolContent.text(
            String.format("Weather in %s: %s, Temperature: %d°C",
                city, weather.getCondition(), weather.getTemperature())
        );
    }
    
    @McpResource(uri = "weather://forecast/{city}")
    public ResourceContent getWeatherForecast(@PathVariable String city) {
        List<Forecast> forecasts = weatherService.getForecast(city, 7);
        return ResourceContent.text(
            forecasts.stream()
                .map(f -> String.format("%s: %s", f.getDate(), f.getCondition()))
                .collect(Collectors.joining("\n"))
        );
    }
}
```

### 17.8.4 MCP 기반 에이전트 구현

```java
@Service
public class McpEnabledAgent {
    
    private final ChatClient chatClient;
    private final McpToolAdapter mcpToolAdapter;
    
    public McpEnabledAgent(ChatModel chatModel, McpToolAdapter mcpToolAdapter) {
        this.mcpToolAdapter = mcpToolAdapter;
        this.chatClient = ChatClient.builder(chatModel)
            .defaultSystem("""
                당신은 MCP 도구를 활용할 수 있는 AI 어시스턴트입니다.
                사용 가능한 도구를 적절히 활용하여 사용자를 도와주세요.
                """)
            .build();
    }
    
    public String processWithMcpTools(String userQuery) {
        // MCP 도구 동적 로드
        List<ToolCallback> mcpTools = mcpToolAdapter.getMcpTools();
        
        return chatClient.prompt()
            .user(userQuery)
            .tools(mcpTools.toArray(new ToolCallback[0]))
            .call()
            .content();
    }
    
    public Flux<String> streamWithMcpTools(String userQuery) {
        List<ToolCallback> mcpTools = mcpToolAdapter.getMcpTools();
        
        return chatClient.prompt()
            .user(userQuery)
            .tools(mcpTools.toArray(new ToolCallback[0]))
            .stream()
            .content();
    }
}

// MCP 리소스 활용 예제
@Component
public class McpResourceAgent {
    
    private final McpSyncClient mcpClient;
    private final ChatClient chatClient;
    
    public String analyzeWithResources(String query) {
        // MCP 리소스 목록 조회
        ListResourcesResult resources = mcpClient.listResources();
        
        // 관련 리소스 읽기
        List<String> relevantData = resources.resources().stream()
            .filter(r -> isRelevantToQuery(r, query))
            .map(r -> mcpClient.readResource(
                new ReadResourceRequest(r.uri())
            ))
            .map(result -> result.contents().stream()
                .map(Content::text)
                .collect(Collectors.joining("\n")))
            .collect(Collectors.toList());
            
        // 리소스 데이터를 컨텍스트로 사용
        return chatClient.prompt()
            .system("""
                다음 데이터를 참고하여 질문에 답변하세요:
                {resourceData}
                """)
            .user(query)
            .call()
            .content();
    }
}
```

## 17.9 프로덕션 배포 고려사항

에이전트를 프로덕션 환경에 배포할 때 고려해야 할 주요 사항들을 살펴봅니다.

### 17.9.1 확장성과 성능 최적화

```java
@Configuration
public class AgentScalabilityConfig {
    
    @Bean
    public ChatClient scalableChatClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel)
            .defaultAdvisors(
                // 캐싱 Advisor
                new CachingAdvisor(cacheManager),
                // 레이트 리미팅 Advisor
                new RateLimitingAdvisor(rateLimiter),
                // 서킷 브레이커 Advisor
                new CircuitBreakerAdvisor(circuitBreaker)
            )
            .build();
    }
    
    @Bean
    public ThreadPoolTaskExecutor agentExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("agent-async-");
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
        executor.initialize();
        return executor;
    }
}

// 캐싱 Advisor 구현
public class CachingAdvisor implements CallAroundAdvisor {
    
    private final CacheManager cacheManager;
    private final Cache responseCache;
    
    public CachingAdvisor(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
        this.responseCache = cacheManager.getCache("ai-responses");
    }
    
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        String cacheKey = generateCacheKey(request);
        
        // 캐시 확인
        AdvisedResponse cached = responseCache.get(cacheKey, AdvisedResponse.class);
        if (cached != null && !isExpired(cached)) {
            logger.debug("캐시 히트: {}", cacheKey);
            return cached;
        }
        
        // 캐시 미스 - 실제 호출
        AdvisedResponse response = chain.nextAroundCall(request);
        
        // 캐시 저장 (성공적인 응답만)
        if (response.response() != null && !response.response().getResult().hasError()) {
            responseCache.put(cacheKey, response);
        }
        
        return response;
    }
}
```

### 17.9.2 보안 강화

```java
@Configuration
@EnableWebSecurity
public class AgentSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/agent/**").authenticated()
                .requestMatchers("/api/agent-metrics/**").hasRole("ADMIN")
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .build();
    }
    
    @Bean
    public PromptSanitizer promptSanitizer() {
        return new PromptSanitizer()
            .addPattern("(?i)ignore.*previous.*instructions", "")
            .addPattern("(?i)system.*prompt", "")
            .addValidator(prompt -> {
                if (prompt.contains("<script>")) {
                    throw new SecurityException("잠재적 보안 위협 감지");
                }
            });
    }
}

// 보안 Advisor
@Component
public class SecurityAdvisor implements CallAroundAdvisor {
    
    private final PromptSanitizer sanitizer;
    private final AuditService auditService;
    
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        // 입력 검증 및 살균
        String sanitizedPrompt = sanitizer.sanitize(request.userText());
        
        // 감사 로그
        auditService.logAiRequest(
            SecurityContextHolder.getContext().getAuthentication(),
            request
        );
        
        // PII 검출 및 마스킹
        String maskedPrompt = maskPII(sanitizedPrompt);
        
        AdvisedRequest sanitizedRequest = AdvisedRequest.from(request)
            .userText(maskedPrompt)
            .build();
            
        return chain.nextAroundCall(sanitizedRequest);
    }
}
```

### 17.9.3 오류 처리와 복원력

```java
@Component
public class ResilienceAdvisor implements CallAroundAdvisor {
    
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final TimeLimiter timeLimiter;
    
    public ResilienceAdvisor() {
        this.circuitBreaker = CircuitBreaker.ofDefaults("ai-service");
        this.retry = Retry.ofDefaults("ai-service");
        this.timeLimiter = TimeLimiter.ofDefaults("ai-service");
        
        // 서킷 브레이커 이벤트 리스너
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                logger.warn("서킷 브레이커 상태 변경: {}", event));
    }
    
    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        Supplier<AdvisedResponse> decoratedSupplier = 
            CircuitBreaker.decorateSupplier(circuitBreaker, 
                () -> chain.nextAroundCall(request));
                
        decoratedSupplier = Retry.decorateSupplier(retry, decoratedSupplier);
        decoratedSupplier = TimeLimiter.decorateSupplier(timeLimiter, decoratedSupplier);
        
        try {
            return decoratedSupplier.get();
        } catch (Exception e) {
            // 폴백 응답 생성
            return createFallbackResponse(request, e);
        }
    }
    
    private AdvisedResponse createFallbackResponse(AdvisedRequest request, Exception e) {
        String fallbackMessage = "죄송합니다. 일시적인 문제로 요청을 처리할 수 없습니다. " +
                               "잠시 후 다시 시도해 주세요.";
                               
        return AdvisedResponse.builder()
            .response(new ChatResponse(List.of(
                new Generation(fallbackMessage)
            )))
            .build();
    }
}
```

### 17.9.4 배포 체크리스트

```java
@Component
@ConditionalOnProperty("ai.agent.health.enabled")
public class AgentHealthIndicator implements HealthIndicator {
    
    private final ChatClient chatClient;
    private final List<HealthCheck> healthChecks = List.of(
        new ModelAvailabilityCheck(),
        new MemoryStoreCheck(),
        new ToolAvailabilityCheck(),
        new RateLimitCheck()
    );
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        for (HealthCheck check : healthChecks) {
            try {
                CheckResult result = check.check();
                details.put(check.getName(), result);
                
                if (!result.isHealthy()) {
                    return Health.down()
                        .withDetails(details)
                        .build();
                }
            } catch (Exception e) {
                return Health.down()
                    .withException(e)
                    .build();
            }
        }
        
        // 종합 테스트: 간단한 프롬프트 실행
        try {
            String response = chatClient.prompt()
                .user("Hello")
                .call()
                .content();
                
            details.put("testPrompt", "성공");
            details.put("lastCheck", LocalDateTime.now());
            
            return Health.up()
                .withDetails(details)
                .build();
                
        } catch (Exception e) {
            return Health.down()
                .withDetail("testPrompt", "실패")
                .withException(e)
                .build();
        }
    }
}
```

## 17.10 실전 프로젝트: 멀티모달 문서 처리 에이전트

최신 Spring AI의 기능을 활용하여 텍스트, 이미지, 문서를 처리할 수 있는 멀티모달 에이전트를 구현합니다.

### 17.10.1 프로젝트 설정

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pdf-document-reader</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-tika-document-reader</artifactId>
    </dependency>
</dependencies>
```

### 17.10.2 멀티모달 문서 처리 에이전트

```java
@Configuration
public class MultimodalAgentConfig {
    
    @Bean
    public ChatClient multimodalChatClient(
            ChatModel chatModel,
            VectorStore vectorStore,
            ChatMemory chatMemory) {
        
        return ChatClient.builder(chatModel)
            .defaultAdvisors(
                // 문서 RAG Advisor
                QuestionAnswerAdvisor.builder(vectorStore)
                    .searchRequest(SearchRequest.defaults()
                        .withTopK(10)
                        .withSimilarityThreshold(0.75))
                    .build(),
                    
                // 메모리 Advisor
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .build(),
                    
                // 관찰성 Advisor
                new ObservabilityAdvisor(meterRegistry, tracer)
            )
            .defaultSystem("""
                당신은 문서 분석 전문가입니다.
                텍스트, 이미지, PDF 등 다양한 형식의 문서를 분석하고
                사용자의 질문에 정확하게 답변할 수 있습니다.
                """)
            .build();
    }
}

@Service
public class MultimodalDocumentAgent {
    
    private final ChatClient chatClient;
    private final DocumentProcessor documentProcessor;
    private final ImageAnalyzer imageAnalyzer;
    
    public DocumentAnalysisResult analyzeDocument(
            MultipartFile file, 
            String analysisRequest,
            String conversationId) {
        
        // 1. 파일 타입에 따른 처리
        DocumentContent content = switch (getFileType(file)) {
            case PDF -> documentProcessor.processPdf(file);
            case IMAGE -> imageAnalyzer.analyzeImage(file);
            case DOCX, TXT -> documentProcessor.processText(file);
            default -> throw new UnsupportedOperationException(
                "지원하지 않는 파일 형식입니다: " + file.getContentType()
            );
        };
        
        // 2. 벡터 저장소에 문서 내용 저장
        List<Document> documents = content.toDocuments();
        vectorStore.add(documents);
        
        // 3. 멀티모달 분석 수행
        String analysis = chatClient.prompt()
            .advisors(advisor -> advisor
                .param(ChatMemory.CONVERSATION_ID, conversationId))
            .user(userText -> userText
                .text("""
                    다음 문서를 분석해주세요:
                    
                    문서 정보:
                    - 파일명: {fileName}
                    - 타입: {fileType}
                    - 크기: {fileSize}
                    
                    분석 요청: {analysisRequest}
                    
                    문서 내용:
                    {documentContent}
                    """)
                .media(content.getMediaList())
                .params(Map.of(
                    "fileName", file.getOriginalFilename(),
                    "fileType", content.getType(),
                    "fileSize", formatFileSize(file.getSize()),
                    "analysisRequest", analysisRequest,
                    "documentContent", content.getTextContent()
                )))
            .tools(new DocumentAnalysisTools())
            .call()
            .content();
            
        return DocumentAnalysisResult.builder()
            .fileName(file.getOriginalFilename())
            .fileType(content.getType())
            .analysis(analysis)
            .extractedEntities(content.getEntities())
            .metadata(content.getMetadata())
            .build();
    }
}

@Component
public class DocumentAnalysisTools {
    
    @Tool(description = "문서에서 특정 정보를 추출합니다")
    public ExtractionResult extractInformation(
            @ToolParam(description = "추출할 정보 유형") String infoType,
            @ToolParam(description = "추가 컨텍스트") String context) {
        
        return extractionService.extract(infoType, context);
    }
    
    @Tool(description = "문서를 다른 형식으로 변환합니다")
    public ConversionResult convertDocument(
            @ToolParam(description = "대상 형식") String targetFormat,
            @ToolParam(description = "변환 옵션") Map<String, Object> options) {
        
        return conversionService.convert(targetFormat, options);
    }
    
    @Tool(description = "문서 요약을 생성합니다", returnDirect = true)
    public String summarizeDocument(
            @ToolParam(description = "요약 길이") String length,
            @ToolParam(description = "요약 스타일") String style) {
        
        return summaryService.summarize(length, style);
    }
}

// 이미지 분석 컴포넌트
@Component
public class ImageAnalyzer {
    
    private final ChatClient imageAnalysisClient;
    
    public DocumentContent analyzeImage(MultipartFile imageFile) {
        String analysis = imageAnalysisClient.prompt()
            .user(userText -> userText
                .text("이 이미지를 상세히 분석하고 설명해주세요.")
                .media(MimeTypeUtils.parseMimeType(imageFile.getContentType()),
                       imageFile.getResource()))
            .call()
            .content();
            
        // OCR 수행 (필요한 경우)
        String extractedText = ocrService.extractText(imageFile);
        
        return DocumentContent.builder()
            .type(DocumentType.IMAGE)
            .textContent(analysis + "\n\nOCR 결과: " + extractedText)
            .mediaList(List.of(new Media(
                MimeTypeUtils.parseMimeType(imageFile.getContentType()),
                imageFile.getResource()
            )))
            .metadata(Map.of(
                "analysis", analysis,
                "ocrText", extractedText
            ))
            .build();
    }
}
```

### 17.10.3 웹 인터페이스와 API

```java
@RestController
@RequestMapping("/api/document-agent")
public class DocumentAgentController {
    
    private final MultimodalDocumentAgent documentAgent;
    private final ConversationService conversationService;
    
    @PostMapping("/analyze")
    public ResponseEntity<DocumentAnalysisResult> analyzeDocument(
            @RequestParam("file") MultipartFile file,
            @RequestParam("request") String analysisRequest,
            @RequestParam(value = "conversationId", required = false) String conversationId) {
        
        if (conversationId == null) {
            conversationId = UUID.randomUUID().toString();
        }
        
        DocumentAnalysisResult result = documentAgent.analyzeDocument(
            file, analysisRequest, conversationId
        );
        
        return ResponseEntity.ok()
            .header("X-Conversation-Id", conversationId)
            .body(result);
    }
    
    @PostMapping("/chat/{conversationId}")
    public ResponseEntity<String> continueConversation(
            @PathVariable String conversationId,
            @RequestBody ChatRequest request) {
        
        String response = documentAgent.chatAboutDocuments(
            conversationId, 
            request.getMessage()
        );
        
        return ResponseEntity.ok(response);
    }
    
    @GetMapping("/conversations/{conversationId}/summary")
    public ResponseEntity<ConversationSummary> getConversationSummary(
            @PathVariable String conversationId) {
        
        ConversationSummary summary = conversationService.summarize(conversationId);
        return ResponseEntity.ok(summary);
    }
}

// WebSocket 지원
@Controller
public class DocumentAgentWebSocketController {
    
    private final MultimodalDocumentAgent documentAgent;
    
    @MessageMapping("/document-analysis/{sessionId}")
    @SendTo("/topic/analysis/{sessionId}")
    public Flux<AnalysisUpdate> streamAnalysis(
            @DestinationVariable String sessionId,
            DocumentAnalysisRequest request) {
        
        return documentAgent.streamAnalysis(request)
            .map(update -> AnalysisUpdate.builder()
                .type(update.getType())
                .content(update.getContent())
                .progress(update.getProgress())
                .timestamp(LocalDateTime.now())
                .build());
    }
}
```

### 17.10.4 프론트엔드 통합 예제

```javascript
// React 컴포넌트 예제
const DocumentAnalyzer = () => {
    const [file, setFile] = useState(null);
    const [analysis, setAnalysis] = useState(null);
    const [conversationId, setConversationId] = useState(null);
    const [loading, setLoading] = useState(false);
    
    const analyzeDocument = async () => {
        setLoading(true);
        const formData = new FormData();
        formData.append('file', file);
        formData.append('request', '이 문서의 주요 내용을 분석해주세요.');
        
        try {
            const response = await fetch('/api/document-agent/analyze', {
                method: 'POST',
                body: formData
            });
            
            const result = await response.json();
            setAnalysis(result);
            setConversationId(response.headers.get('X-Conversation-Id'));
        } finally {
            setLoading(false);
        }
    };
    
    const askFollowUp = async (question) => {
        const response = await fetch(`/api/document-agent/chat/${conversationId}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ message: question })
        });
        
        const answer = await response.text();
        // 답변 표시 로직
    };
    
    // UI 렌더링...
};
```

## 17.11 결론

이 장에서는 Spring AI의 최신 에이전트 API에 대해 살펴보았습니다:

**핵심 내용 정리:**
- **Advisors API**: AI 상호작용을 가로채고 향상시키는 강력한 인터셉터 패턴
- **Tool Calling**: @Tool 어노테이션을 통한 선언적 도구 정의와 동적 실행
- **ChatMemory**: 대화 컨텍스트 유지를 위한 유연한 메모리 관리
- **에이전트 패턴**: Chain, Parallel, Routing, Orchestrator, Evaluator-Optimizer
- **MCP 통합**: 표준화된 도구 및 리소스 프로토콜 지원

Spring AI는 단순성과 구성 가능성을 중시하며, 복잡한 프레임워크보다는 명확한 패턴과 모듈식 접근을 통해 효과적인 AI 에이전트를 구축할 수 있도록 지원합니다.

다음 장에서는 이러한 기본 구성요소들을 활용하여 더 복잡한 멀티 에이전트 시스템과 워크플로를 구축하는 방법을 알아보겠습니다.

## 연습 문제

1. **Advisor 구현**: 사용자 입력을 검증하고 PII(개인식별정보)를 마스킹하는 SecurityAdvisor를 구현하세요. CallAroundAdvisor와 StreamAroundAdvisor 인터페이스를 모두 구현해야 합니다.

2. **복합 도구 구현**: 다음 기능을 가진 도구 세트를 구현하세요:
   - 웹 검색 도구 (@Tool 어노테이션 사용)
   - 검색 결과 요약 도구 (ToolContext 활용)
   - 요약 결과를 PDF로 저장하는 도구 (returnDirect = true)

3. **메모리 관리 확장**: VectorStoreChatMemoryAdvisor를 확장하여 대화 중요도에 따라 선택적으로 장기 메모리에 저장하는 SmartMemoryAdvisor를 구현하세요.

4. **에이전트 패턴 적용**: 주어진 비즈니스 요구사항을 분석하고, 적절한 에이전트 패턴(Chain, Parallel, Routing 등)을 선택하여 "온라인 쇼핑몰 고객 서비스" 에이전트를 설계하세요.

5. **MCP 서버 구현**: Spring AI MCP Server를 사용하여 다음 기능을 제공하는 MCP 서버를 구현하세요:
   - 날씨 정보 제공 도구
   - 환율 정보 제공 도구
   - 두 도구를 조합한 여행 정보 리소스