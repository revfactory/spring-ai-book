# 14. AI 에이전트 기본 개념

## 14.1 AI 에이전트의 정의와 중요성

AI 에이전트는 사용자를 대신하여 특정 작업을 수행하거나 문제를 해결하도록 설계된 자율적인 시스템입니다. Spring AI 1.0.0 GA는 Anthropic의 "Building Effective Agents" 연구를 기반으로 실용적이고 효과적인 에이전트 구축을 위한 패턴과 도구를 제공합니다.

### 에이전틱 시스템의 분류

Spring AI는 에이전틱 시스템을 두 가지 유형으로 구분합니다:

1. **워크플로우(Workflows)**: LLM과 도구가 미리 정의된 코드 경로를 통해 조율되는 시스템
2. **에이전트(Agents)**: LLM이 동적으로 자신의 프로세스와 도구 사용을 결정하는 시스템

핵심 인사이트는 완전 자율 에이전트가 매력적으로 보일 수 있지만, 잘 정의된 작업에서는 워크플로우가 더 나은 예측 가능성과 일관성을 제공한다는 것입니다.

### 14.1.1 효과적인 에이전트의 핵심 원칙

Anthropic의 연구와 Spring AI의 구현을 바탕으로 한 효과적인 에이전트의 특성:

- **단순성 우선**: 복잡한 프레임워크보다 단순하고 구성 가능한 접근 방식을 선호합니다.
- **신뢰성**: 예측 가능하고 일관된 결과를 제공합니다.
- **조합 가능성**: 작은 구성 요소를 조합하여 복잡한 기능을 구현합니다.
- **도구 통합**: 외부 시스템과 API를 효과적으로 활용합니다.
- **구조화된 출력**: 타입 안전한 응답과 명확한 데이터 구조를 사용합니다.

### 14.1.2 Spring AI 에이전트의 장점

Spring AI는 다음과 같은 이점을 제공합니다:

- **모델 이식성**: 다양한 LLM 제공업체 간 일관된 API 제공
- **구조화된 출력**: 타입 안전한 응답 처리
- **일관된 API**: 통합된 인터페이스와 내장된 오류 처리
- **Spring 생태계 통합**: Spring Boot, Spring Security 등과의 원활한 통합
- **확장 가능한 아키텍처**: 엔터프라이즈급 요구사항 지원

## 14.2 Spring AI 에이전틱 패턴

Spring AI는 5가지 기본 에이전틱 패턴을 제공하며, 각각 특정 사용 사례에 맞게 설계되었습니다.

### 14.2.1 체인 워크플로우 (Chain Workflow)

복잡한 작업을 관리 가능한 단계로 분해하는 패턴입니다.

![Chain Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F7418719e3dab222dccb379b8879e1dc08ad34c78-2401x1000.png&w=3840&q=75)

**사용 시점:**
- 명확한 순차적 단계가 있는 작업
- 지연 시간을 감수하고 높은 정확도를 원할 때
- 각 단계가 이전 단계의 출력을 기반으로 할 때

```java
@Component
public class ChainWorkflow {
    private final ChatClient chatClient;
    private final String[] systemPrompts;

    public ChainWorkflow(ChatClient chatClient) {
        this.chatClient = chatClient;
        this.systemPrompts = new String[]{
            "1단계: 입력 텍스트를 분석하고 주요 개념을 추출하세요.",
            "2단계: 추출된 개념들 간의 관계를 파악하세요.",
            "3단계: 최종 요약을 생성하세요."
        };
    }

    public String process(String userInput) {
        String response = userInput;
        for (String prompt : systemPrompts) {
            String input = String.format("%s\n\n%s", prompt, response);
            response = chatClient.prompt(input).call().content();
        }
        return response;
    }
}
```

### 14.2.2 병렬화 워크플로우 (Parallelization Workflow)

LLM이 작업을 동시에 처리하고 결과를 프로그래밍적으로 집계하는 패턴입니다.

![Parallelization Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F406bb032ca007fd1624f261af717d70e6ca86286-2401x1000.png&w=3840&q=75)

**사용 시점:**
- 유사하지만 독립적인 항목을 대량 처리할 때
- 여러 독립적인 관점이 필요한 작업
- 처리 시간이 중요하고 작업이 병렬화 가능할 때

```java
@Component
public class ParallelizationWorkflow {
    private final ChatClient chatClient;
    private final Executor executor;

    public ParallelizationWorkflow(ChatClient chatClient) {
        this.chatClient = chatClient;
        this.executor = Executors.newVirtualThreadPerTaskExecutor();
    }

    public List<String> parallel(String basePrompt, List<String> items, int maxConcurrency) {
        Semaphore semaphore = new Semaphore(maxConcurrency);
        
        List<CompletableFuture<String>> futures = items.stream()
            .map(item -> CompletableFuture.supplyAsync(() -> {
                try {
                    semaphore.acquire();
                    String prompt = String.format("%s\n\n%s", basePrompt, item);
                    return chatClient.prompt(prompt).call().content();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                } finally {
                    semaphore.release();
                }
            }, executor))
            .toList();

        return futures.stream()
            .map(CompletableFuture::join)
            .toList();
    }
}
```

### 14.2.3 라우팅 워크플로우 (Routing Workflow)

지능적인 작업 분배를 구현하여 다양한 유형의 입력에 대해 전문화된 처리를 가능하게 하는 패턴입니다.

![Routing Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F5c0c0e9fe4def0b584c04d37849941da55e5e71c-2401x1000.png&w=3840&q=75)

**사용 시점:**
- 입력의 명확한 카테고리가 있는 복잡한 작업
- 다른 입력이 전문화된 처리를 요구할 때
- 분류가 정확하게 처리될 수 있을 때

```java
@Component
public class RoutingWorkflow {
    private final ChatClient chatClient;

    public RoutingWorkflow(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String route(String userInput, Map<String, String> routes) {
        // 1. 입력 분류
        String classificationPrompt = String.format(
            "다음 사용자 입력을 분류하세요. 가능한 카테고리: %s\n\n사용자 입력: %s\n\n카테고리만 응답하세요:",
            String.join(", ", routes.keySet()),
            userInput
        );
        
        String category = chatClient.prompt(classificationPrompt).call().content().trim().toLowerCase();
        
        // 2. 해당 카테고리의 전문 프롬프트로 처리
        String specializedPrompt = routes.getOrDefault(category, routes.get("general"));
        if (specializedPrompt == null) {
            throw new IllegalArgumentException("No handler found for category: " + category);
        }
        
        String fullPrompt = String.format("%s\n\n사용자 입력: %s", specializedPrompt, userInput);
        return chatClient.prompt(fullPrompt).call().content();
    }
}

// 사용 예시
@Service
public class CustomerServiceAgent {
    private final RoutingWorkflow routingWorkflow;
    
    public CustomerServiceAgent(RoutingWorkflow routingWorkflow) {
        this.routingWorkflow = routingWorkflow;
    }
    
    public String handleCustomerQuery(String query) {
        Map<String, String> routes = Map.of(
            "billing", "당신은 청구 전문가입니다. 청구 문제 해결을 도와주세요...",
            "technical", "당신은 기술 지원 엔지니어입니다. 기술적 문제를 해결해주세요...",
            "general", "당신은 고객 서비스 담당자입니다. 일반적인 문의를 도와주세요..."
        );
        
        return routingWorkflow.route(query, routes);
    }
}
```

### 14.2.4 오케스트레이터-워커 패턴 (Orchestrator-Workers)

복잡한 작업을 동적으로 하위 작업으로 분해하고 전문화된 워커들이 처리하는 패턴입니다.

![Orchestration Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F8985fc683fae4780fb34eab1365ab78c7e51bc8e-2401x1000.png&w=3840&q=75)

**사용 시점:**
- 하위 작업을 미리 예측할 수 없는 복잡한 작업
- 다른 접근 방식이나 관점이 필요한 작업
- 적응적 문제 해결이 필요한 상황

```java
@Component
public class OrchestratorWorkersWorkflow {
    private final ChatClient chatClient;
    
    public OrchestratorWorkersWorkflow(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public WorkerResponse process(String taskDescription) {
        // 1. 오케스트레이터가 작업 분석 및 하위 작업 결정
        OrchestratorResponse orchestratorResponse = analyzeTask(taskDescription);
        
        // 2. 워커들이 하위 작업을 병렬 처리
        List<String> workerResponses = processSubtasks(orchestratorResponse.subtasks());
        
        // 3. 결과를 최종 응답으로 결합
        String finalResult = combineResults(taskDescription, workerResponses);
        
        return new WorkerResponse(orchestratorResponse.analysis(), workerResponses, finalResult);
    }
    
    private OrchestratorResponse analyzeTask(String taskDescription) {
        String prompt = String.format("""
            작업을 분석하고 3-5개의 구체적인 하위 작업으로 분해하세요.
            각 하위 작업은 독립적으로 처리 가능해야 합니다.
            
            작업: %s
            
            응답 형식:
            분석: [작업에 대한 분석]
            하위작업:
            1. [하위작업 1]
            2. [하위작업 2]
            ...
            """, taskDescription);
        
        String response = chatClient.prompt(prompt).call().content();
        return parseOrchestratorResponse(response);
    }
    
    private List<String> processSubtasks(List<String> subtasks) {
        return subtasks.parallelStream()
            .map(subtask -> {
                String prompt = String.format("다음 작업을 수행하세요: %s", subtask);
                return chatClient.prompt(prompt).call().content();
            })
            .toList();
    }
    
    private String combineResults(String originalTask, List<String> workerResponses) {
        String prompt = String.format("""
            원래 작업: %s
            
            워커 결과들:
            %s
            
            위 결과들을 종합하여 원래 작업에 대한 완전한 답변을 제공하세요.
            """, originalTask, String.join("\n\n", workerResponses));
        
        return chatClient.prompt(prompt).call().content();
    }
    
    private OrchestratorResponse parseOrchestratorResponse(String response) {
        // 응답 파싱 로직
        String[] lines = response.split("\n");
        String analysis = "";
        List<String> subtasks = new ArrayList<>();
        
        boolean inSubtasks = false;
        for (String line : lines) {
            if (line.startsWith("분석:")) {
                analysis = line.substring(3).trim();
            } else if (line.startsWith("하위작업:")) {
                inSubtasks = true;
            } else if (inSubtasks && line.matches("\\d+\\. .*")) {
                subtasks.add(line.substring(line.indexOf('.') + 1).trim());
            }
        }
        
        return new OrchestratorResponse(analysis, subtasks);
    }
    
    record OrchestratorResponse(String analysis, List<String> subtasks) {}
    record WorkerResponse(String analysis, List<String> workerResponses, String finalResult) {}
}
```

### 14.2.5 평가자-최적화자 패턴 (Evaluator-Optimizer)

반복적인 개선을 통해 응답 품질을 향상시키는 패턴입니다.

![Evaluator-Optimizer Workflow](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F14f51e6406ccb29e695da48b17017e899a6119c7-2401x1000.png&w=3840&q=75)

**사용 시점:**
- 명확한 평가 기준이 존재할 때
- 반복적인 개선이 측정 가능한 가치를 제공할 때
- 여러 번의 비평을 통해 개선되는 작업

```java
@Component
public class EvaluatorOptimizerWorkflow {
    private final ChatClient chatClient;
    private final int maxIterations;
    
    public EvaluatorOptimizerWorkflow(ChatClient chatClient) {
        this.chatClient = chatClient;
        this.maxIterations = 3;
    }
    
    public RefinedResponse refine(String task) {
        String currentSolution = generateInitialSolution(task);
        List<String> evolutionSteps = new ArrayList<>();
        evolutionSteps.add("초기 솔루션: " + currentSolution);
        
        for (int i = 0; i < maxIterations; i++) {
            EvaluationResponse evaluation = evaluate(currentSolution, task);
            evolutionSteps.add(String.format("평가 %d: %s", i + 1, evaluation.feedback()));
            
            if (evaluation.score() >= 8.0) {
                evolutionSteps.add("목표 품질 달성, 개선 종료");
                break;
            }
            
            String improvedSolution = improve(currentSolution, evaluation.feedback(), task);
            evolutionSteps.add(String.format("개선 %d: %s", i + 1, improvedSolution));
            currentSolution = improvedSolution;
        }
        
        return new RefinedResponse(currentSolution, evolutionSteps);
    }
    
    private String generateInitialSolution(String task) {
        String prompt = String.format("다음 작업에 대한 솔루션을 제공하세요: %s", task);
        return chatClient.prompt(prompt).call().content();
    }
    
    private EvaluationResponse evaluate(String solution, String originalTask) {
        String prompt = String.format("""
            다음 솔루션을 평가하세요:
            
            원래 작업: %s
            솔루션: %s
            
            평가 기준:
            - 정확성 (1-10)
            - 완전성 (1-10)
            - 명확성 (1-10)
            
            응답 형식:
            점수: [평균 점수]
            피드백: [구체적인 개선 사항]
            """, originalTask, solution);
        
        String response = chatClient.prompt(prompt).call().content();
        return parseEvaluationResponse(response);
    }
    
    private String improve(String currentSolution, String feedback, String originalTask) {
        String prompt = String.format("""
            다음 피드백을 바탕으로 솔루션을 개선하세요:
            
            원래 작업: %s
            현재 솔루션: %s
            피드백: %s
            
            개선된 솔루션을 제공하세요:
            """, originalTask, currentSolution, feedback);
        
        return chatClient.prompt(prompt).call().content();
    }
    
    private EvaluationResponse parseEvaluationResponse(String response) {
        String[] lines = response.split("\n");
        double score = 0.0;
        String feedback = "";
        
        for (String line : lines) {
            if (line.startsWith("점수:")) {
                try {
                    score = Double.parseDouble(line.substring(3).trim());
                } catch (NumberFormatException e) {
                    score = 5.0; // 기본값
                }
            } else if (line.startsWith("피드백:")) {
                feedback = line.substring(4).trim();
            }
        }
        
        return new EvaluationResponse(score, feedback);
    }
    
    record EvaluationResponse(double score, String feedback) {}
    record RefinedResponse(String finalSolution, List<String> evolutionSteps) {}
}
```

## 14.3 Spring AI 에이전트 구현의 장점

### 14.3.1 모델 이식성 (Model Portability)

Spring AI는 다양한 LLM 제공업체 간 일관된 API를 제공합니다:

```xml
<!-- OpenAI 사용 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>

<!-- Anthropic 사용 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-anthropic</artifactId>
</dependency>

<!-- Ollama 사용 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
```

동일한 ChatClient 인터페이스로 모든 모델을 사용할 수 있습니다:

```java
@Service
public class AgnosticAgentService {
    private final ChatClient chatClient;
    
    // 어떤 모델이든 동일한 방식으로 사용
    public String process(String input) {
        return chatClient.prompt(input).call().content();
    }
}
```

### 14.3.2 구조화된 출력 (Structured Output)

Spring AI는 타입 안전한 응답 처리를 지원합니다:

```java
public class AnalysisAgent {
    private final ChatClient chatClient;
    
    public AnalysisResult analyzeDocument(String document) {
        String prompt = String.format("""
            다음 문서를 분석하고 구조화된 결과를 제공하세요:
            
            %s
            
            결과는 다음 형식으로 제공하세요:
            - summary: 요약
            - keyTopics: 주요 주제 목록
            - sentiment: 감정 (POSITIVE, NEGATIVE, NEUTRAL)
            - confidence: 신뢰도 (0.0-1.0)
            """, document);
        
        return chatClient.prompt(prompt)
            .call()
            .entity(AnalysisResult.class);
    }
    
    record AnalysisResult(
        String summary,
        List<String> keyTopics,
        Sentiment sentiment,
        double confidence
    ) {}
    
    enum Sentiment { POSITIVE, NEGATIVE, NEUTRAL }
}
```

### 14.3.3 일관된 API와 통합 기능

Spring AI는 다음과 같은 통합된 기능을 제공합니다:

```java
@Component
public class ComprehensiveAgent {
    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;
    
    public ComprehensiveAgent(
        ChatClient chatClient,
        VectorStore vectorStore,
        EmbeddingModel embeddingModel
    ) {
        this.chatClient = chatClient;
        this.vectorStore = vectorStore;
        this.embeddingModel = embeddingModel;
    }
    
    public String processWithContext(String query) {
        // 1. 관련 문서 검색
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(3)
        );
        
        // 2. 컨텍스트와 함께 응답 생성
        String context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));
        
        return chatClient.prompt()
            .system("다음 컨텍스트를 바탕으로 답변하세요: " + context)
            .user(query)
            .call()
            .content();
    }
    
    @Retryable(value = {Exception.class}, maxAttempts = 3)
    public String processWithRetry(String input) {
        return chatClient.prompt(input).call().content();
    }
    
    @Observed(name = "agent.processing")
    public String processWithObservability(String input) {
        return chatClient.prompt(input).call().content();
    }
}
```

## 14.4 실제 사용 사례와 구현

### 14.4.1 고객 서비스 에이전트

다양한 패턴을 조합한 종합적인 고객 서비스 에이전트:

```java
@Service
public class CustomerServiceAgent {
    private final RoutingWorkflow routingWorkflow;
    private final ChainWorkflow chainWorkflow;
    private final ChatClient chatClient;
    
    public CustomerServiceAgent(
        RoutingWorkflow routingWorkflow,
        ChainWorkflow chainWorkflow,
        ChatClient chatClient
    ) {
        this.routingWorkflow = routingWorkflow;
        this.chainWorkflow = chainWorkflow;
        this.chatClient = chatClient;
    }
    
    public CustomerServiceResponse handleInquiry(String customerQuery, String customerId) {
        try {
            // 1. 라우팅을 통한 초기 분류
            String category = classifyInquiry(customerQuery);
            
            // 2. 카테고리별 전문 처리
            String response = switch (category) {
                case "complex" -> handleComplexInquiry(customerQuery);
                case "billing" -> handleBillingInquiry(customerQuery, customerId);
                case "technical" -> handleTechnicalInquiry(customerQuery);
                default -> handleGeneralInquiry(customerQuery);
            };
            
            return new CustomerServiceResponse(category, response, "SUCCESS");
            
        } catch (Exception e) {
            return new CustomerServiceResponse("error", 
                "죄송합니다. 일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요.", 
                "ERROR");
        }
    }
    
    private String handleComplexInquiry(String query) {
        // 복잡한 문의는 체인 워크플로우로 처리
        return chainWorkflow.process(query);
    }
    
    private String handleBillingInquiry(String query, String customerId) {
        // 고객 정보와 함께 청구 관련 처리
        String prompt = String.format("""
            고객 ID: %s
            청구 관련 문의: %s
            
            청구 전문가로서 정확하고 도움이 되는 답변을 제공하세요.
            필요시 관련 부서 연결을 안내하세요.
            """, customerId, query);
        
        return chatClient.prompt(prompt).call().content();
    }
    
    private String classifyInquiry(String query) {
        Map<String, String> routes = Map.of(
            "complex", "복잡하고 여러 단계가 필요한 문의",
            "billing", "청구, 결제, 요금 관련 문의",
            "technical", "기술적 문제, 오류, 사용법 관련 문의",
            "general", "일반적인 정보 문의"
        );
        
        return routingWorkflow.route(query, routes);
    }
    
    record CustomerServiceResponse(String category, String response, String status) {}
}
```

### 14.4.2 코드 리뷰 에이전트

평가자-최적화자 패턴을 활용한 코드 리뷰 에이전트:

```java
@Service
public class CodeReviewAgent {
    private final EvaluatorOptimizerWorkflow evaluatorOptimizer;
    private final ChatClient chatClient;
    
    public CodeReviewAgent(
        EvaluatorOptimizerWorkflow evaluatorOptimizer,
        ChatClient chatClient
    ) {
        this.evaluatorOptimizer = evaluatorOptimizer;
        this.chatClient = chatClient;
    }
    
    public CodeReviewResult reviewCode(String code, String requirements) {
        // 1. 초기 코드 분석
        CodeAnalysis analysis = analyzeCode(code);
        
        // 2. 개선사항 식별 및 최적화
        String task = String.format("""
            다음 요구사항에 맞게 코드를 검토하고 개선하세요:
            
            요구사항: %s
            
            코드:
            %s
            
            보안, 성능, 가독성, 모범 사례 관점에서 검토하세요.
            """, requirements, code);
        
        RefinedResponse refinedResult = evaluatorOptimizer.refine(task);
        
        // 3. 최종 권장사항 생성
        List<String> recommendations = generateRecommendations(
            analysis, refinedResult.finalSolution()
        );
        
        return new CodeReviewResult(
            analysis,
            recommendations,
            refinedResult.finalSolution(),
            refinedResult.evolutionSteps()
        );
    }
    
    private CodeAnalysis analyzeCode(String code) {
        String prompt = String.format("""
            다음 코드를 분석하세요:
            
            %s
            
            분석 결과를 다음 형식으로 제공하세요:
            - 복잡도: [HIGH/MEDIUM/LOW]
            - 주요이슈: [발견된 이슈들]
            - 강점: [코드의 좋은 점들]
            - 위험도: [CRITICAL/HIGH/MEDIUM/LOW]
            """, code);
        
        return chatClient.prompt(prompt)
            .call()
            .entity(CodeAnalysis.class);
    }
    
    private List<String> generateRecommendations(CodeAnalysis analysis, String refinedFeedback) {
        String prompt = String.format("""
            코드 분석 결과와 개선된 피드백을 바탕으로 실행 가능한 권장사항을 생성하세요:
            
            분석 결과: %s
            개선된 피드백: %s
            
            우선순위별로 정렬된 권장사항 목록을 제공하세요.
            """, analysis, refinedFeedback);
        
        String response = chatClient.prompt(prompt).call().content();
        return parseRecommendations(response);
    }
    
    private List<String> parseRecommendations(String response) {
        return Arrays.stream(response.split("\n"))
            .filter(line -> line.matches("\\d+\\. .*"))
            .map(line -> line.substring(line.indexOf('.') + 1).trim())
            .toList();
    }
    
    record CodeAnalysis(String complexity, List<String> issues, List<String> strengths, String riskLevel) {}
    record CodeReviewResult(CodeAnalysis analysis, List<String> recommendations, String improvedCode, List<String> reviewProcess) {}
}
```

## 14.5 에이전트 개발 모범 사례

### 14.5.1 단순함에서 시작

- **기본 워크플로우 우선**: 복잡성을 추가하기 전에 기본 패턴으로 시작
- **요구사항에 맞는 가장 간단한 패턴 사용**: 과도한 엔지니어링 방지
- **필요시에만 정교함 추가**: 명확한 개선이 있을 때만 복잡성 추가

```java
// 좋은 예: 간단한 시작
@Service
public class SimpleDocumentSummarizer {
    private final ChatClient chatClient;
    
    public String summarize(String document) {
        return chatClient.prompt()
            .system("다음 문서를 3-5문장으로 요약하세요.")
            .user(document)
            .call()
            .content();
    }
}

// 복잡함이 필요한 경우에만 추가
@Service
public class AdvancedDocumentProcessor {
    private final ChainWorkflow chainWorkflow;
    private final EvaluatorOptimizerWorkflow optimizer;
    
    public ProcessedDocument process(String document) {
        // 복잡한 처리 로직
        return optimizer.refine(chainWorkflow.process(document));
    }
}
```

### 14.5.2 신뢰성을 위한 설계

- **명확한 오류 처리**: 각 단계에서 발생할 수 있는 오류 처리
- **타입 안전 응답**: 가능한 곳에서 구조화된 출력 사용
- **각 단계에서 검증**: 중간 결과의 유효성 검증

```java
@Service
public class ReliableAgent {
    private final ChatClient chatClient;
    
    public Result processWithValidation(String input) {
        try {
            // 입력 검증
            validateInput(input);
            
            // 처리
            ProcessedData result = chatClient.prompt(input)
                .call()
                .entity(ProcessedData.class);
            
            // 결과 검증
            validateResult(result);
            
            return Result.success(result);
            
        } catch (ValidationException e) {
            return Result.error("입력 검증 실패: " + e.getMessage());
        } catch (Exception e) {
            return Result.error("처리 중 오류 발생: " + e.getMessage());
        }
    }
    
    private void validateInput(String input) {
        if (input == null || input.trim().isEmpty()) {
            throw new ValidationException("입력이 비어있습니다");
        }
        if (input.length() > 10000) {
            throw new ValidationException("입력이 너무 깁니다");
        }
    }
    
    private void validateResult(ProcessedData result) {
        if (result == null || result.content() == null) {
            throw new ValidationException("처리 결과가 유효하지 않습니다");
        }
    }
    
    record ProcessedData(String content, double confidence) {}
    
    sealed interface Result permits Result.Success, Result.Error {
        record Success(ProcessedData data) implements Result {}
        record Error(String message) implements Result {}
        
        static Result success(ProcessedData data) { return new Success(data); }
        static Result error(String message) { return new Error(message); }
    }
}
```

### 14.5.3 절충점 고려

- **지연 시간 vs 정확도**: 체인 패턴은 정확도는 높이지만 지연 시간이 증가
- **병렬 처리 활용**: 독립적인 작업은 병렬 처리로 성능 향상
- **고정 워크플로우 vs 동적 에이전트**: 예측 가능성과 유연성 간 균형

```java
@Service
public class OptimizedAgent {
    private final ChatClient chatClient;
    
    public String processOptimized(String input, ProcessingMode mode) {
        return switch (mode) {
            case FAST -> processFast(input);        // 단일 호출, 빠름
            case ACCURATE -> processAccurate(input); // 체인 처리, 정확함
            case BALANCED -> processBalanced(input); // 2단계 처리, 균형
        };
    }
    
    private String processFast(String input) {
        return chatClient.prompt(input).call().content();
    }
    
    private String processAccurate(String input) {
        // 3단계 체인 처리
        String step1 = chatClient.prompt("1단계: " + input).call().content();
        String step2 = chatClient.prompt("2단계: " + step1).call().content();
        return chatClient.prompt("3단계: " + step2).call().content();
    }
    
    private String processBalanced(String input) {
        // 분석 후 처리
        String analysis = chatClient.prompt("분석: " + input).call().content();
        return chatClient.prompt("처리: " + analysis).call().content();
    }
    
    enum ProcessingMode { FAST, ACCURATE, BALANCED }
}
```

### 14.5.4 패턴 조합 전략

실제 애플리케이션에서는 여러 패턴을 조합하여 사용합니다:

```java
@Service
public class HybridAgent {
    private final RoutingWorkflow routing;
    private final ChainWorkflow chain;
    private final ParallelizationWorkflow parallel;
    private final OrchestratorWorkersWorkflow orchestrator;
    
    public ProcessingResult processComplex(String input, String context) {
        // 1. 라우팅으로 작업 유형 결정
        String taskType = classifyTask(input);
        
        // 2. 작업 유형에 따른 적절한 패턴 적용
        return switch (taskType) {
            case "analytical" -> {
                // 분석적 작업: 체인 워크플로우
                String result = chain.process(input);
                yield new ProcessingResult(taskType, result, "chain");
            }
            case "distributed" -> {
                // 분산 처리: 병렬화 워크플로우
                List<String> parts = splitInput(input);
                List<String> results = parallel.parallel("처리하세요:", parts, 4);
                String combined = String.join("\n", results);
                yield new ProcessingResult(taskType, combined, "parallel");
            }
            case "complex" -> {
                // 복잡한 작업: 오케스트레이터-워커
                WorkerResponse result = orchestrator.process(input);
                yield new ProcessingResult(taskType, result.finalResult(), "orchestrator");
            }
            default -> {
                // 기본 처리
                String result = chatClient.prompt(input).call().content();
                yield new ProcessingResult(taskType, result, "simple");
            }
        };
    }
    
    private String classifyTask(String input) {
        Map<String, String> routes = Map.of(
            "analytical", "단계적 분석이 필요한 작업",
            "distributed", "병렬 처리가 가능한 작업",
            "complex", "동적 하위 작업 분해가 필요한 작업",
            "simple", "단순한 작업"
        );
        return routing.route(input, routes);
    }
    
    private List<String> splitInput(String input) {
        // 입력을 병렬 처리 가능한 부분으로 분할
        return Arrays.asList(input.split("\n\n"));
    }
    
    record ProcessingResult(String taskType, String result, String patternUsed) {}
}
```

## 14.6 Spring Boot 통합

### 14.6.1 의존성 및 구성 설정

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4
          temperature: 0.7
          max-tokens: 2000

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

### 14.6.2 RESTful API 엔드포인트

```java
@RestController
@RequestMapping("/api/agent")
public class AgentController {
    private final CustomerServiceAgent customerService;
    private final CodeReviewAgent codeReview;
    private final HybridAgent hybridAgent;
    
    public AgentController(
        CustomerServiceAgent customerService,
        CodeReviewAgent codeReview,
        HybridAgent hybridAgent
    ) {
        this.customerService = customerService;
        this.codeReview = codeReview;
        this.hybridAgent = hybridAgent;
    }
    
    @PostMapping("/customer-service")
    public ResponseEntity<CustomerServiceResponse> handleCustomerInquiry(
        @RequestBody CustomerInquiryRequest request
    ) {
        try {
            CustomerServiceResponse response = customerService.handleInquiry(
                request.query(), request.customerId()
            );
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new CustomerServiceResponse("error", 
                    "서비스 오류가 발생했습니다", "ERROR"));
        }
    }
    
    @PostMapping("/code-review")
    public ResponseEntity<CodeReviewResult> reviewCode(
        @RequestBody CodeReviewRequest request
    ) {
        CodeReviewResult result = codeReview.reviewCode(
            request.code(), request.requirements()
        );
        return ResponseEntity.ok(result);
    }
    
    @PostMapping("/process")
    public ResponseEntity<ProcessingResult> processComplex(
        @RequestBody ProcessingRequest request
    ) {
        ProcessingResult result = hybridAgent.processComplex(
            request.input(), request.context()
        );
        return ResponseEntity.ok(result);
    }
    
    record CustomerInquiryRequest(String query, String customerId) {}
    record CodeReviewRequest(String code, String requirements) {}
    record ProcessingRequest(String input, String context) {}
}
```

### 14.6.3 구성 클래스

```java
@Configuration
@EnableConfigurationProperties
public class AgentConfiguration {
    
    @Bean
    public ChainWorkflow chainWorkflow(ChatClient chatClient) {
        return new ChainWorkflow(chatClient);
    }
    
    @Bean
    public ParallelizationWorkflow parallelizationWorkflow(ChatClient chatClient) {
        return new ParallelizationWorkflow(chatClient);
    }
    
    @Bean
    public RoutingWorkflow routingWorkflow(ChatClient chatClient) {
        return new RoutingWorkflow(chatClient);
    }
    
    @Bean
    public OrchestratorWorkersWorkflow orchestratorWorkflow(ChatClient chatClient) {
        return new OrchestratorWorkersWorkflow(chatClient);
    }
    
    @Bean
    public EvaluatorOptimizerWorkflow evaluatorOptimizerWorkflow(ChatClient chatClient) {
        return new EvaluatorOptimizerWorkflow(chatClient);
    }
    
    @Bean
    public CustomerServiceAgent customerServiceAgent(
        RoutingWorkflow routingWorkflow,
        ChainWorkflow chainWorkflow,
        ChatClient chatClient
    ) {
        return new CustomerServiceAgent(routingWorkflow, chainWorkflow, chatClient);
    }
    
    @Bean
    public CodeReviewAgent codeReviewAgent(
        EvaluatorOptimizerWorkflow evaluatorOptimizer,
        ChatClient chatClient
    ) {
        return new CodeReviewAgent(evaluatorOptimizer, chatClient);
    }
    
    @Bean
    public HybridAgent hybridAgent(
        RoutingWorkflow routing,
        ChainWorkflow chain,
        ParallelizationWorkflow parallel,
        OrchestratorWorkersWorkflow orchestrator,
        ChatClient chatClient
    ) {
        return new HybridAgent(routing, chain, parallel, orchestrator, chatClient);
    }
}
```

## 14.7 미래 발전 방향

에이전트의 품질과 성능을 평가하기 위한 주요 지표:

- **작업 완료율**: 에이전트가 주어진 작업을 성공적으로 완료하는 비율
- **정확성**: 제공된 정보나 수행된 작업의 정확도
- **응답 시간**: 사용자 요청에 대한 에이전트의 응답 속도
- **자원 사용**: CPU, 메모리, API 호출 등 자원 소비량
- **사용자 만족도**: 사용자 피드백 및 평가 점수

### 14.5.2 품질 평가 방법론

에이전트의 품질을 종합적으로 평가하는 방법:

- **인간 평가**: 전문가나 사용자에 의한 직접 평가
- **자동화된 테스트**: 사전 정의된 시나리오와 기대 결과를 기반으로 한 테스트
- **A/B 테스트**: 다른 버전의 에이전트를 비교하는 실험
- **행동 분석**: 에이전트의 결정과 행동 패턴 분석
- **오류 분석**: 실패 사례의 체계적 분석 및 분류

### 14.5.3 평가 시나리오 설계

효과적인 에이전트 평가를 위한 시나리오 설계 전략:

- **현실적 사용 사례**: 실제 사용 환경과 유사한 시나리오 설계
- **경계 조건 테스트**: 에이전트의 한계를 시험하는 극단적 상황 포함
- **복잡도 수준**: 다양한 복잡도의 작업 포함
- **다양성 확보**: 다양한 주제, 상황, 입력 형식 포함
- **장기적 상호작용**: 단일 요청뿐만 아니라 연속적인 대화나 작업 시퀀스 평가

## 14.6 Spring AI와 에이전트 개발

### 14.6.1 Spring AI의 에이전트 지원 기능

Spring AI는 다음과 같은 에이전트 개발 지원 기능을 제공합니다:

- **AI 모델 통합**: 다양한 LLM과의 쉬운 통합을 지원합니다.
- **프롬프트 엔지니어링 도구**: 효과적인 프롬프트 설계와 관리를 위한 도구를 제공합니다.
- **도구 사용 인프라**: 외부 도구와 시스템을 쉽게 통합할 수 있는 인프라를 제공합니다.
- **메모리 관리 시스템**: 대화 및 상태 관리를 위한 메모리 시스템을 제공합니다.
- **평가 및 모니터링**: 에이전트 성능을 모니터링하고 평가하는 도구를 제공합니다.

### 14.6.2 Spring AI 에이전트 프레임워크 소개

Spring AI의 에이전트 프레임워크는 다음과 같은 구성 요소로 이루어집니다:

- **AgentTemplate**: 에이전트 구현을 위한 기본 템플릿을 제공합니다.
- **ToolRegistry**: 에이전트가 사용할 수 있는 도구를 등록하고 관리합니다.
- **MemoryStore**: 대화 및 상태 정보를 저장하고 관리합니다.
- **PromptTemplate**: 효과적인 프롬프트 설계를 위한 템플릿을 제공합니다.
- **AIServices**: 모델 호출과 응답 처리를 위한 서비스를 제공합니다.

### 14.6.3 Spring Boot와의 통합

Spring AI 에이전트를 Spring Boot 애플리케이션에 통합하는 방법:

- **의존성 관리**: 필요한 Spring AI 의존성을 프로젝트에 추가합니다.
- **구성 설정**: application.properties 또는 application.yml에서 AI 모델 및 에이전트 설정을 구성합니다.
- **빈 등록**: 에이전트 및 관련 구성 요소를 Spring 빈으로 등록합니다.
- **API 엔드포인트**: 에이전트와 상호작용하기 위한 REST API 엔드포인트를 설계합니다.
- **서비스 계층 통합**: 기존 서비스 계층과 에이전트의 통합 방법을 설계합니다.

## 14.7 요약 및 다음 단계

이 장에서는 AI 에이전트의 기본 개념, 아키텍처, 유형, 개발 고려사항 및 Spring AI와의 통합에 대해 알아보았습니다. 효과적인 AI 에이전트 개발을 위해서는 명확한 목표 설정, 적절한 아키텍처 선택, 효과적인 프롬프트 설계, 도구 통합, 메모리 관리, 안전성 확보, 그리고 체계적인 평가가 필요합니다.

다음 장에서는 Spring AI를 사용하여 실제 에이전트를 설계하고 구현하는 과정을 단계별로 살펴보겠습니다. 이를 통해 이론적 개념을 실제 코드로 구현하는 방법을 배우게 될 것입니다.