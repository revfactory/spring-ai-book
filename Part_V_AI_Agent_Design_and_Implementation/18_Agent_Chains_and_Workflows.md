# 18장: 효과적인 에이전트 구축: 워크플로와 패턴

## 18.1 개요

Anthropic의 "Building Effective Agents" 연구에서 제시된 원칙에 따르면, 효과적인 AI 에이전트를 구축하는 핵심은 복잡한 프레임워크보다 **단순성과 구성 가능성**에 있습니다. 이 장에서는 Spring AI를 사용하여 이러한 원칙을 구현하는 방법을 살펴봅니다.

### 18.1.1 에이전트 시스템의 두 가지 유형

**1. 워크플로 (Workflows)**
- LLM과 도구가 사전 정의된 코드 경로를 통해 오케스트레이션되는 시스템
- 예측 가능하고 일관된 동작
- 명확한 단계와 흐름을 가진 작업에 적합

**2. 에이전트 (Agents)**
- LLM이 동적으로 자신의 프로세스와 도구 사용을 지시하는 시스템
- 더 유연하지만 예측하기 어려움
- 탐색적이고 적응적인 작업에 적합

### 18.1.2 Spring AI의 접근 방식

Spring AI는 다음과 같은 장점을 제공합니다:

- **모델 이식성**: 다양한 LLM 제공자 간 일관된 API
- **구조화된 출력**: 타입 안전한 응답 처리
- **단순한 패턴**: 복잡한 프레임워크 대신 명확한 패턴 사용
- **구성 가능성**: 작은 구성 요소를 조합하여 복잡한 시스템 구축

### 18.1.3 5가지 기본 워크플로 패턴

1. **Chain Workflow**: 순차적 작업 처리
2. **Parallelization Workflow**: 병렬 작업 실행
3. **Routing Workflow**: 조건부 작업 분기
4. **Orchestrator-Workers**: 동적 작업 분해
5. **Evaluator-Optimizer**: 반복적 개선

## 18.2 Chain Workflow 패턴

**Chain Workflow**는 복잡한 작업을 더 단순하고 관리 가능한 단계로 분해하는 패턴입니다.

### 18.2.1 언제 사용하나?

- 명확한 순차적 단계가 있는 작업
- 각 단계가 이전 단계의 출력을 기반으로 할 때
- 정확성을 위해 지연 시간을 감수할 수 있을 때

### 18.2.2 기본 구현

```java
@Component
public class ChainWorkflow {
    
    private final ChatClient chatClient;
    
    public ChainWorkflow(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String chain(String userInput, String... systemPrompts) {
        String response = userInput;
        
        for (String prompt : systemPrompts) {
            response = chatClient.prompt()
                .system(prompt)
                .user(response)
                .call()
                .content();
        }
        
        return response;
    }
}
```

### 18.2.3 실제 예제: 연구 보고서 생성

```java
@Service
public class ResearchReportService {
    
    private final ChatClient chatClient;
    
    public String generateResearchReport(String topic) {
        ChainWorkflow workflow = new ChainWorkflow(chatClient);
        
        return workflow.chain(
            topic,
            // 단계 1: 연구 주제 분석
            "주어진 주제를 분석하고 핵심 연구 질문을 도출하세요. " +
            "각 질문은 구체적이고 답변 가능해야 합니다.",
            
            // 단계 2: 정보 수집 계획
            "연구 질문을 바탕으로 필요한 정보 소스와 " +
            "데이터 수집 방법을 상세히 기술하세요.",
            
            // 단계 3: 분석 프레임워크 개발
            "수집된 정보를 분석할 프레임워크를 개발하고 " +
            "주요 평가 기준을 설정하세요.",
            
            // 단계 4: 보고서 작성
            "분석 결과를 바탕으로 체계적인 연구 보고서를 작성하세요. " +
            "서론, 방법론, 결과, 결론 섹션을 포함해야 합니다."
        );
    }
}
```

### 18.2.4 고급 Chain 구현: 컨텍스트 전파

```java
@Component
public class ContextAwareChainWorkflow {
    
    private final ChatClient chatClient;
    
    public ChainResult executeWithContext(String initialInput, List<ChainStep> steps) {
        Map<String, Object> context = new HashMap<>();
        String currentOutput = initialInput;
        List<StepResult> stepResults = new ArrayList<>();
        
        for (ChainStep step : steps) {
            // 이전 단계의 출력과 컨텍스트를 사용
            String prompt = step.buildPrompt(currentOutput, context);
            
            String response = chatClient.prompt()
                .system(step.getSystemPrompt())
                .user(prompt)
                .call()
                .content();
            
            // 결과 저장 및 컨텍스트 업데이트
            StepResult result = new StepResult(step.getName(), response);
            stepResults.add(result);
            
            // 다음 단계를 위한 컨텍스트 업데이트
            context.putAll(step.extractContext(response));
            currentOutput = response;
        }
        
        return new ChainResult(currentOutput, stepResults, context);
    }
    
    @Data
    @Builder
    public static class ChainStep {
        private String name;
        private String systemPrompt;
        private Function<String, Map<String, Object>> contextExtractor;
        
        public String buildPrompt(String previousOutput, Map<String, Object> context) {
            // 이전 출력과 컨텍스트를 기반으로 프롬프트 구성
            return String.format(
                "Previous output: %s\nContext: %s\nPlease proceed with: %s",
                previousOutput, context, systemPrompt
            );
        }
        
        public Map<String, Object> extractContext(String output) {
            return contextExtractor != null ? 
                contextExtractor.apply(output) : 
                Collections.emptyMap();
        }
    }
}
```

### 18.2.5 Chain Workflow의 장점

1. **명확한 구조**: 각 단계가 명확히 정의됨
2. **디버깅 용이**: 각 단계의 입출력을 추적 가능
3. **재사용성**: 개별 단계를 다른 체인에서 재사용 가능
4. **점진적 개선**: 특정 단계만 수정 가능

## 18.3 Parallelization Workflow 패턴

**Parallelization Workflow**는 LLM이 여러 작업을 동시에 처리하고 결과를 프로그래밍 방식으로 집계할 수 있게 합니다.

### 18.3.1 언제 사용하나?

- 대량의 유사하지만 독립적인 항목 처리
- 여러 독립적인 관점이 필요한 작업
- 처리 시간이 중요하고 작업이 병렬화 가능할 때

### 18.3.2 기본 구현

```java
@Component
public class ParallelizationWorkflow {
    
    private final ChatClient chatClient;
    private final ExecutorService executorService;
    
    public ParallelizationWorkflow(ChatClient chatClient) {
        this.chatClient = chatClient;
        this.executorService = Executors.newFixedThreadPool(10);
    }
    
    public List<String> parallel(String task, List<String> items, int maxConcurrency) {
        List<CompletableFuture<String>> futures = items.stream()
            .map(item -> CompletableFuture.supplyAsync(() -> 
                processItem(task, item), executorService))
            .collect(Collectors.toList());
        
        return futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
    }
    
    private String processItem(String task, String item) {
        return chatClient.prompt()
            .system(task)
            .user(item)
            .call()
            .content();
    }
}
```

### 18.3.3 실제 예제: 이해관계자 영향 분석

```java
@Service
public class StakeholderAnalysisService {
    
    private final ParallelizationWorkflow workflow;
    
    public StakeholderAnalysisReport analyzeBusinessChange(String businessChange) {
        List<String> stakeholders = Arrays.asList(
            "고객: 제품 사용자와 구매자",
            "직원: 내부 팀과 관리자",
            "투자자: 주주와 이사회",
            "공급업체: 파트너와 벤더"
        );
        
        String analysisPrompt = 
            "다음 비즈니스 변화가 이해관계자 그룹에 미치는 영향을 분석하세요: " +
            businessChange + "\n" +
            "긍정적/부정적 영향, 단기/장기 효과, 권장 조치를 포함하세요.";
        
        List<String> analyses = workflow.parallel(
            analysisPrompt,
            stakeholders,
            4
        );
        
        return StakeholderAnalysisReport.builder()
            .businessChange(businessChange)
            .stakeholderAnalyses(mapToStakeholderAnalyses(stakeholders, analyses))
            .summary(generateSummary(analyses))
            .build();
    }
    
    private String generateSummary(List<String> analyses) {
        // 모든 분석을 종합하여 요약 생성
        String combinedAnalyses = String.join("\n\n", analyses);
        
        return chatClient.prompt()
            .system("다음 이해관계자 분석들을 종합하여 전체 요약을 작성하세요.")
            .user(combinedAnalyses)
            .call()
            .content();
    }
}

```

### 18.3.4 고급: 결과 집계 전략

```java
@Component
public class AdvancedParallelizationWorkflow {
    
    private final ChatClient chatClient;
    
    public <T> ParallelResult<T> executeWithAggregation(
            String task,
            List<String> items,
            Function<String, T> resultParser,
            BiFunction<List<T>, ChatClient, T> aggregator) {
        
        // 병렬 처리
        List<CompletableFuture<T>> futures = items.stream()
            .map(item -> CompletableFuture.supplyAsync(() -> {
                String response = chatClient.prompt()
                    .system(task)
                    .user(item)
                    .call()
                    .content();
                return resultParser.apply(response);
            }))
            .collect(Collectors.toList());
        
        // 결과 수집
        List<T> results = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        // 집계
        T aggregatedResult = aggregator.apply(results, chatClient);
        
        return new ParallelResult<>(results, aggregatedResult);
    }
    
    @Data
    @AllArgsConstructor
    public static class ParallelResult<T> {
        private List<T> individualResults;
        private T aggregatedResult;
    }
}
```

### 18.3.5 Parallelization의 장점

1. **성능 향상**: 독립적인 작업을 동시에 처리
2. **확장성**: 작업 수에 따라 동적으로 확장 가능
3. **다양한 관점**: 여러 분석을 병렬로 수행하여 종합적인 결과 도출

## 18.4 Routing Workflow 패턴

**Routing Workflow**는 입력을 분류하고 적절한 전문 처리기로 라우팅하는 지능적인 작업 분배를 구현합니다.

### 18.4.1 언제 사용하나?

- 입력이 명확한 카테고리로 분류될 때
- 각 카테고리가 특화된 처리를 필요로 할 때
- 정확한 분류가 가능할 때

### 18.4.2 기본 구현

```java
@Component
public class RoutingWorkflow {
    
    private final ChatClient chatClient;
    
    public String route(String input, Map<String, String> routes) {
        // 입력 분류
        String classification = classify(input, routes.keySet());
        
        // 적절한 프롬프트 선택
        String systemPrompt = routes.getOrDefault(classification, 
            "You are a helpful assistant. Please help with the user's request.");
        
        // 전문 처리
        return chatClient.prompt()
            .system(systemPrompt)
            .user(input)
            .call()
            .content();
    }
    
    private String classify(String input, Set<String> categories) {
        String categoriesStr = String.join(", ", categories);
        
        return chatClient.prompt()
            .system("Classify the following input into one of these categories: " + 
                   categoriesStr + ". Respond with only the category name.")
            .user(input)
            .call()
            .content()
            .toLowerCase()
            .trim();
    }
}
```

### 18.4.3 실제 예제: 고객 지원 라우터

```java
@Service
public class CustomerSupportRouter {
    
    private final RoutingWorkflow routingWorkflow;
    
    public CustomerSupportResponse handleCustomerQuery(String query) {
        Map<String, String> routes = Map.of(
            "billing", "You are a billing specialist. Help resolve billing issues, " +
                       "payment problems, and subscription questions. Be precise about amounts and dates.",
            
            "technical", "You are a technical support engineer. Help solve technical problems, " +
                         "troubleshoot errors, and guide through technical configurations.",
            
            "product", "You are a product specialist. Provide detailed information about " +
                       "product features, comparisons, and recommendations.",
            
            "general", "You are a customer service representative. Help with general " +
                       "inquiries and direct customers to appropriate resources."
        );
        
        String response = routingWorkflow.route(query, routes);
        
        return CustomerSupportResponse.builder()
            .query(query)
            .response(response)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

### 18.4.4 고급: 다단계 라우팅

```java
@Component
public class HierarchicalRoutingWorkflow {
    
    private final ChatClient chatClient;
    
    public String routeHierarchically(String input, RoutingTree routingTree) {
        RoutingNode currentNode = routingTree.getRoot();
        String currentInput = input;
        
        while (!currentNode.isLeaf()) {
            // 현재 레벨에서 분류
            String category = classify(currentInput, currentNode.getChildCategories());
            currentNode = currentNode.getChild(category);
            
            if (currentNode == null) {
                currentNode = routingTree.getDefaultNode();
                break;
            }
        }
        
        // 최종 노드의 프롬프트로 처리
        return chatClient.prompt()
            .system(currentNode.getSystemPrompt())
            .user(input)
            .call()
            .content();
    }
    
    @Data
    @Builder
    public static class RoutingNode {
        private String category;
        private String systemPrompt;
        private Map<String, RoutingNode> children;
        
        public boolean isLeaf() {
            return children == null || children.isEmpty();
        }
        
        public Set<String> getChildCategories() {
            return children != null ? children.keySet() : Collections.emptySet();
        }
        
        public RoutingNode getChild(String category) {
            return children != null ? children.get(category) : null;
        }
    }
}

```

### 18.4.5 Routing의 장점

1. **전문화**: 각 카테고리에 최적화된 처리
2. **확장성**: 새로운 카테고리 추가가 용이
3. **정확성**: 특화된 프롬프트로 더 나은 결과
4. **유지보수성**: 각 라우트를 독립적으로 관리

## 18.5 Orchestrator-Workers 패턴

**Orchestrator-Workers**는 복잡한 작업을 동적으로 하위 작업으로 분해하고 조정하는 패턴입니다.

### 18.5.1 언제 사용하나?

- 하위 작업을 미리 예측할 수 없는 복잡한 작업
- 다양한 접근 방식이나 관점이 필요한 작업
- 적응적 문제 해결이 필요한 상황

### 18.5.2 기본 구현

```java
@Component
public class OrchestratorWorkersWorkflow {
    
    private final ChatClient chatClient;
    
    public WorkerResponse process(String taskDescription) {
        // 1. Orchestrator가 작업을 분석하고 하위 작업 결정
        OrchestratorResponse orchestration = analyzeAndDecompose(taskDescription);
        
        // 2. Workers가 병렬로 하위 작업 처리
        List<String> workerResponses = executeWorkers(orchestration.getSubtasks());
        
        // 3. 결과를 최종 응답으로 결합
        String finalResponse = combineResults(orchestration, workerResponses);
        
        return WorkerResponse.builder()
            .analysis(orchestration.getAnalysis())
            .workerResponses(workerResponses)
            .finalResponse(finalResponse)
            .build();
    }
    
    private OrchestratorResponse analyzeAndDecompose(String taskDescription) {
        String response = chatClient.prompt()
            .system("""
                You are an orchestrator. Analyze the given task and break it down into subtasks.
                Return a JSON with:
                - analysis: your understanding of the task
                - subtasks: array of specific subtasks for workers
                - integration_strategy: how to combine worker results
                """)
            .user(taskDescription)
            .call()
            .entity(OrchestratorResponse.class);
        
        return response;
    }
    
    private List<String> executeWorkers(List<String> subtasks) {
        return subtasks.parallelStream()
            .map(subtask -> chatClient.prompt()
                .system("You are a specialized worker. Complete the assigned subtask.")
                .user(subtask)
                .call()
                .content())
            .collect(Collectors.toList());
    }
    
    private String combineResults(OrchestratorResponse orchestration, 
                                 List<String> workerResponses) {
        String combinedContext = String.format(
            "Original task analysis: %s\n" +
            "Integration strategy: %s\n" +
            "Worker results:\n%s",
            orchestration.getAnalysis(),
            orchestration.getIntegrationStrategy(),
            String.join("\n---\n", workerResponses)
        );
        
        return chatClient.prompt()
            .system("Combine the worker results into a cohesive final response.")
            .user(combinedContext)
            .call()
            .content();
    }
    
    @Data
    public static class OrchestratorResponse {
        private String analysis;
        private List<String> subtasks;
        private String integrationStrategy;
    }
    
    @Data
    @Builder
    public static class WorkerResponse {
        private String analysis;
        private List<String> workerResponses;
        private String finalResponse;
    }
}

```

### 18.5.3 실제 예제: API 문서 생성

```java
@Service
public class APIDocumentationService {
    
    private final OrchestratorWorkersWorkflow workflow;
    
    public APIDocumentation generateDocumentation(String apiEndpointDescription) {
        WorkerResponse response = workflow.process(
            "Generate comprehensive documentation for this REST API endpoint: " +
            apiEndpointDescription + "\n" +
            "Include technical documentation for developers and " +
            "user-friendly documentation for end users."
        );
        
        // 워커 응답을 구조화된 문서로 변환
        return APIDocumentation.builder()
            .endpoint(apiEndpointDescription)
            .technicalDocs(extractTechnicalDocs(response))
            .userGuide(extractUserGuide(response))
            .examples(extractExamples(response))
            .generatedAt(LocalDateTime.now())
            .build();
    }
}

```

### 18.5.4 Orchestrator-Workers의 장점

1. **유연성**: 작업에 따라 동적으로 접근 방식 결정
2. **전문화**: 각 워커가 특정 측면에 집중
3. **확장성**: 필요에 따라 워커 추가/제거 가능
4. **적응성**: 복잡하고 예측 불가능한 작업 처리

## 18.6 Evaluator-Optimizer 패턴

**Evaluator-Optimizer**는 반복적인 평가와 개선을 통해 결과의 품질을 향상시키는 패턴입니다.

### 18.6.1 언제 사용하나?

- 명확한 평가 기준이 존재할 때
- 반복적 개선이 측정 가능한 가치를 제공할 때
- 여러 라운드의 비평이 도움이 되는 작업

### 18.6.2 기본 구현

```java
@Component
public class EvaluatorOptimizerWorkflow {
    
    private final ChatClient chatClient;
    private static final int MAX_ITERATIONS = 3;
    
    public RefinedResponse loop(String task) {
        List<Generation> generations = new ArrayList<>();
        Generation currentGeneration = generate(task, null);
        generations.add(currentGeneration);
        
        for (int i = 0; i < MAX_ITERATIONS; i++) {
            EvaluationResponse evaluation = evaluate(currentGeneration.getResponse(), task);
            
            if (evaluation.getScore() >= 0.9) {
                break; // 충분히 좋은 결과
            }
            
            currentGeneration = generate(task, evaluation.getFeedback());
            generations.add(currentGeneration);
        }
        
        return RefinedResponse.builder()
            .finalSolution(currentGeneration.getResponse())
            .chainOfThought(generations)
            .build();
    }
    
    private Generation generate(String task, String feedback) {
        String prompt = feedback == null ?
            "Complete this task: " + task :
            "Improve your previous solution based on feedback:\n" +
            "Task: " + task + "\n" +
            "Feedback: " + feedback;
        
        String response = chatClient.prompt()
            .system("You are an expert problem solver.")
            .user(prompt)
            .call()
            .content();
        
        return new Generation(response, feedback);
    }
    
    private EvaluationResponse evaluate(String solution, String task) {
        return chatClient.prompt()
            .system("""
                Evaluate the solution for the given task.
                Provide:
                - score: 0.0 to 1.0
                - issues: list of problems
                - suggestions: specific improvements
                """)
            .user("Task: " + task + "\nSolution: " + solution)
            .call()
            .entity(EvaluationResponse.class);
    }
    
    @Data
    @AllArgsConstructor
    public static class Generation {
        private String response;
        private String feedback;
    }
    
    @Data
    public static class EvaluationResponse {
        private double score;
        private List<String> issues;
        private List<String> suggestions;
        
        public String getFeedback() {
            return "Issues: " + String.join(", ", issues) + "\n" +
                   "Suggestions: " + String.join(", ", suggestions);
        }
    }
    
    @Data
    @Builder
    public static class RefinedResponse {
        private String finalSolution;
        private List<Generation> chainOfThought;
    }
}

```

### 18.6.3 실제 예제: 코드 품질 개선

```java
@Service
public class CodeQualityService {
    
    private final EvaluatorOptimizerWorkflow workflow;
    
    public ImprovedCode improveCode(String code, String requirements) {
        String task = "Improve this code to meet the requirements:\n" +
                     "Code:\n" + code + "\n" +
                     "Requirements:\n" + requirements;
        
        RefinedResponse response = workflow.loop(task);
        
        return ImprovedCode.builder()
            .originalCode(code)
            .improvedCode(response.getFinalSolution())
            .iterations(response.getChainOfThought().size())
            .improvementSteps(extractImprovementSteps(response.getChainOfThought()))
            .build();
    }
    
    private List<String> extractImprovementSteps(List<Generation> generations) {
        return generations.stream()
            .filter(g -> g.getFeedback() != null)
            .map(Generation::getFeedback)
            .collect(Collectors.toList());
    }
}
```

### 18.6.4 Evaluator-Optimizer의 장점

1. **품질 향상**: 반복적 개선으로 더 나은 결과
2. **투명성**: 개선 과정을 추적 가능
3. **객관적 평가**: 명확한 기준에 따른 평가
4. **학습 가능**: 평가 피드백을 통한 개선

## 18.5 반복 워크플로 구현

일정한 조건이 충족될 때까지 반복적으로 작업을 수행해야 하는 경우 반복 워크플로를 구현할 수 있습니다.

### 18.5.1 IterativeAgentWorkflow 구현

```java
public class IterativeAgentWorkflow implements AgentWorkflow<AgentRequest, AgentResponse> {
    
    private final Agent agent;
    private final Predicate<AgentResponse> continuationCondition;
    private final Function<AgentResponse, AgentRequest> feedbackTransformer;
    private final int maxIterations;
    
    public IterativeAgentWorkflow(
            Agent agent,
            Predicate<AgentResponse> continuationCondition,
            Function<AgentResponse, AgentRequest> feedbackTransformer,
            int maxIterations) {
        this.agent = agent;
        this.continuationCondition = continuationCondition;
        this.feedbackTransformer = feedbackTransformer;
        this.maxIterations = maxIterations;
    }
    
    @Override
    public AgentResponse execute(AgentRequest initialRequest) {
        AgentRequest currentRequest = initialRequest;
        AgentResponse response = null;
        int iterationCount = 0;
        
        do {
            // 현재 반복 횟수 확인
            if (iterationCount >= maxIterations) {
                break;
            }
            
            // 에이전트 실행
            response = agent.execute(currentRequest);
            iterationCount++;
            
            // 계속 진행 여부 확인
            if (!continuationCondition.test(response)) {
                break;
            }
            
            // 다음 반복을 위한 요청 생성
            currentRequest = feedbackTransformer.apply(response);
            
        } while (true);
        
        return response;
    }
    
    @Override
    public List<Agent> getAgents() {
        return Collections.singletonList(agent);
    }
    
    @Override
    public WorkflowMetadata getMetadata() {
        return WorkflowMetadata.builder()
            .name("IterativeAgentWorkflow")
            .description("Iterative execution of an agent until a condition is met")
            .build();
    }
}
```

### 18.5.2 반복 조건 및 피드백 루프 구현

```java
@Service
public class ContentRefinementService {
    
    private final Agent contentRefinementAgent;
    
    public String refineContent(String initialContent, List<String> refinementCriteria) {
        // 반복 조건 정의: 모든 기준이 충족될 때까지 반복
        Predicate<AgentResponse> continuationCondition = response -> {
            String content = response.getContent();
            return refinementCriteria.stream()
                .anyMatch(criteria -> !contentMeetsCriteria(content, criteria));
        };
        
        // 피드백 변환기 정의: 충족되지 않은 기준을 피드백으로 제공
        Function<AgentResponse, AgentRequest> feedbackTransformer = response -> {
            String content = response.getContent();
            
            // 충족되지 않은 기준 찾기
            List<String> unmetCriteria = refinementCriteria.stream()
                .filter(criteria -> !contentMeetsCriteria(content, criteria))
                .collect(Collectors.toList());
            
            // 피드백 메시지 생성
            String feedback = "Please improve the content to meet the following criteria: " +
                String.join(", ", unmetCriteria);
            
            return AgentRequest.builder()
                .prompt(content)
                .metadata(Map.of("feedback", feedback))
                .build();
        };
        
        // 반복 워크플로 생성
        IterativeAgentWorkflow workflow = new IterativeAgentWorkflow(
            contentRefinementAgent,
            continuationCondition,
            feedbackTransformer,
            5 // 최대 5회 반복
        );
        
        // 워크플로 실행
        AgentResponse result = workflow.execute(
            AgentRequest.builder()
                .prompt(initialContent)
                .metadata(Map.of("criteria", refinementCriteria))
                .build()
        );
        
        return result.getContent();
    }
    
    private boolean contentMeetsCriteria(String content, String criteria) {
        // 콘텐츠가 특정 기준을 충족하는지 평가하는 로직
    }
}
```

### 18.5.3 반복 진행 상황 추적

```java
@Component
public class IterationProgressTracker {
    
    public <T> T trackIterativeProgress(
            Supplier<T> iterationStep,
            Consumer<IterationProgress> progressHandler,
            int currentIteration,
            int maxIterations) {
        
        // 진행 상황 객체 생성
        IterationProgress progress = new IterationProgress(currentIteration, maxIterations);
        progressHandler.accept(progress);
        
        // 시작 시간 기록
        long startTime = System.currentTimeMillis();
        
        try {
            // 반복 단계 실행
            T result = iterationStep.get();
            
            // 소요 시간 계산
            long elapsedTime = System.currentTimeMillis() - startTime;
            
            // 진행 상황 업데이트
            progress.setStatus("COMPLETED");
            progress.setElapsedTimeMs(elapsedTime);
            progressHandler.accept(progress);
            
            return result;
            
        } catch (Exception e) {
            // 오류 처리
            progress.setStatus("FAILED");
            progress.setErrorMessage(e.getMessage());
            progressHandler.accept(progress);
            
            throw e;
        }
    }
    
    public static class IterationProgress {
        private final int currentIteration;
        private final int maxIterations;
        private String status;
        private long elapsedTimeMs;
        private String errorMessage;
        
        // 생성자, 게터, 세터 등...
    }
}
```

## 18.6 이벤트 기반 워크플로 구현

비동기 처리와 이벤트 기반 아키텍처를 활용하여 유연한 워크플로를 구현할 수 있습니다.

### 18.6.1 EventDrivenAgentWorkflow 구현

```java
public class EventDrivenAgentWorkflow {
    
    private final Map<String, Agent> agentsByEventType;
    private final ApplicationEventPublisher eventPublisher;
    
    public EventDrivenAgentWorkflow(
            Map<String, Agent> agentsByEventType,
            ApplicationEventPublisher eventPublisher) {
        this.agentsByEventType = new HashMap<>(agentsByEventType);
        this.eventPublisher = eventPublisher;
    }
    
    public void handleEvent(AgentEvent event) {
        String eventType = event.getEventType();
        Agent agent = agentsByEventType.get(eventType);
        
        if (agent == null) {
            log.warn("No agent registered for event type: {}", eventType);
            return;
        }
        
        try {
            // 이벤트에서 에이전트 요청 생성
            AgentRequest request = createRequestFromEvent(event);
            
            // 에이전트 실행
            AgentResponse response = agent.execute(request);
            
            // 응답 처리 및 후속 이벤트 발행
            processResponseAndPublishEvents(response, event);
            
        } catch (Exception e) {
            // 오류 처리 및 오류 이벤트 발행
            log.error("Error processing event: {}", e.getMessage(), e);
            eventPublisher.publishEvent(
                new AgentErrorEvent(this, event, e.getMessage())
            );
        }
    }
    
    private AgentRequest createRequestFromEvent(AgentEvent event) {
        // 이벤트에서 에이전트 요청 생성
    }
    
    private void processResponseAndPublishEvents(AgentResponse response, AgentEvent sourceEvent) {
        // 응답 처리 및 후속 이벤트 발행
    }
}
```

### 18.6.2 이벤트 리스너 및 핸들러 구현

```java
@Component
public class AgentEventHandler {
    
    private final EventDrivenAgentWorkflow workflow;
    
    @EventListener
    public void handleUserQueryEvent(UserQueryEvent event) {
        workflow.handleEvent(event);
    }
    
    @EventListener
    public void handleDocumentAnalysisEvent(DocumentAnalysisEvent event) {
        workflow.handleEvent(event);
    }
    
    @EventListener
    public void handleAlertEvent(AlertEvent event) {
        workflow.handleEvent(event);
    }
    
    @EventListener
    public void handleAgentErrorEvent(AgentErrorEvent event) {
        // 오류 이벤트 처리
    }
}
```

### 18.6.3 분산 이벤트 처리 시스템

```java
@Configuration
public class DistributedAgentWorkflowConfig {
    
    @Bean
    public EventDrivenAgentWorkflow eventDrivenWorkflow(
            Map<String, Agent> agents,
            ApplicationEventPublisher eventPublisher) {
        
        Map<String, Agent> agentMapping = Map.of(
            "user.query", agents.get("queryRouter"),
            "document.analysis", agents.get("documentAnalyzer"),
            "alert.generated", agents.get("alertHandler"),
            "data.processed", agents.get("dataReporter")
        );
        
        return new EventDrivenAgentWorkflow(agentMapping, eventPublisher);
    }
    
    @Bean
    public KafkaTemplate<String, AgentEvent> kafkaTemplate() {
        // Kafka 템플릿 구성
    }
    
    @Bean
    public KafkaListener kafkaAgentEventListener(EventDrivenAgentWorkflow workflow) {
        return new KafkaListener(workflow);
    }
    
    public static class KafkaListener {
        private final EventDrivenAgentWorkflow workflow;
        
        @KafkaListener(topics = "agent-events")
        public void listen(AgentEvent event) {
            workflow.handleEvent(event);
        }
    }
}
```

## 18.7 패턴 조합 전략

개별 워크플로 패턴은 강력하지만, 실제 복잡한 문제를 해결하려면 여러 패턴을 조합해야 합니다.

### 18.7.1 하이브리드 워크플로 구현

```java
@Service
public class HybridWorkflowService {
    
    private final ChatClient chatClient;
    
    public ResearchReport conductResearch(String topic) {
        // 1단계: Chain을 사용하여 연구 계획 수립
        String researchPlan = new ChainWorkflow(chatClient).chain(
            topic,
            "Analyze the research topic and identify key areas to investigate.",
            "Create a structured research plan with specific questions."
        );
        
        // 2단계: Parallelization을 사용하여 여러 소스에서 정보 수집
        List<String> sources = Arrays.asList(
            "Academic papers",
            "Industry reports",
            "Expert opinions",
            "Case studies"
        );
        
        ParallelizationWorkflow parallelWorkflow = new ParallelizationWorkflow(chatClient);
        List<String> researchData = parallelWorkflow.parallel(
            "Research " + topic + " from the perspective of: ",
            sources,
            4
        );
        
        // 3단계: Evaluator-Optimizer로 결과 개선
        String task = "Synthesize the research findings into a comprehensive report";
        RefinedResponse refinedReport = new EvaluatorOptimizerWorkflow(chatClient)
            .loop(task + "\nData: " + String.join("\n", researchData));
        
        return ResearchReport.builder()
            .topic(topic)
            .plan(researchPlan)
            .sources(researchData)
            .finalReport(refinedReport.getFinalSolution())
            .iterations(refinedReport.getChainOfThought().size())
            .build();
    }
}
```

### 18.7.2 적응형 워크플로 선택

```java
@Component
public class AdaptiveWorkflowSelector {
    
    private final ChatClient chatClient;
    
    public String selectAndExecuteWorkflow(String task) {
        // 작업 분석
        String taskAnalysis = chatClient.prompt()
            .system("Analyze the task and determine the best workflow pattern.")
            .user(task)
            .call()
            .entity(TaskAnalysis.class);
        
        // 분석 결과에 따라 적절한 워크플로 선택
        return switch (taskAnalysis.getRecommendedPattern()) {
            case CHAIN -> executeChainWorkflow(task);
            case PARALLEL -> executeParallelWorkflow(task);
            case ROUTING -> executeRoutingWorkflow(task);
            case ORCHESTRATOR -> executeOrchestratorWorkflow(task);
            case EVALUATOR -> executeEvaluatorWorkflow(task);
            default -> executeDefaultWorkflow(task);
        };
    }
    
    @Data
    static class TaskAnalysis {
        enum Pattern { CHAIN, PARALLEL, ROUTING, ORCHESTRATOR, EVALUATOR }
        private Pattern recommendedPattern;
        private String reasoning;
        private List<String> subtasks;
    }
}
```

### 18.7.3 중첩 워크플로 구현

```java
@Service
public class NestedWorkflowService {
    
    private final ChatClient chatClient;
    
    public ProductAnalysis analyzeProduct(String productDescription) {
        // 최상위 Orchestrator
        OrchestratorWorkersWorkflow orchestrator = new OrchestratorWorkersWorkflow(chatClient);
        
        WorkerResponse mainAnalysis = orchestrator.process(
            "Analyze this product from multiple perspectives: " + productDescription
        );
        
        // 각 워커 응답에 대해 추가 분석 수행
        List<DetailedAnalysis> detailedAnalyses = mainAnalysis.getWorkerResponses()
            .parallelStream()
            .map(workerResponse -> {
                // 중첩된 Chain Workflow
                String refined = new ChainWorkflow(chatClient).chain(
                    workerResponse,
                    "Extract key insights from this analysis.",
                    "Identify actionable recommendations.",
                    "Prioritize recommendations by impact."
                );
                
                return new DetailedAnalysis(workerResponse, refined);
            })
            .collect(Collectors.toList());
        
        // 최종 통합 (Evaluator-Optimizer 사용)
        String consolidationTask = "Consolidate all analyses into executive summary";
        RefinedResponse summary = new EvaluatorOptimizerWorkflow(chatClient)
            .loop(consolidationTask + "\n" + detailedAnalyses);
        
        return ProductAnalysis.builder()
            .product(productDescription)
            .mainAnalysis(mainAnalysis)
            .detailedAnalyses(detailedAnalyses)
            .executiveSummary(summary.getFinalSolution())
            .build();
    }
}
```

## 18.8 성능 최적화와 모니터링

효과적인 워크플로를 구축하려면 성능 최적화와 적절한 모니터링이 필수적입니다.

### 18.8.1 성능 최적화 전략

```java
@Component
public class WorkflowOptimizer {
    
    private final ChatClient chatClient;
    private final MeterRegistry meterRegistry;
    
    // 1. 캐싱을 통한 중복 호출 최소화
    private final Cache<String, String> responseCache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build();
    
    public String optimizedChain(String input, String... prompts) {
        String response = input;
        
        for (String prompt : prompts) {
            String cacheKey = prompt + "::" + response.hashCode();
            
            response = responseCache.get(cacheKey, key -> {
                Timer.Sample sample = Timer.start(meterRegistry);
                
                String result = chatClient.prompt()
                    .system(prompt)
                    .user(response)
                    .call()
                    .content();
                
                sample.stop(Timer.builder("workflow.step.duration")
                    .tag("prompt", prompt.substring(0, Math.min(prompt.length(), 50)))
                    .register(meterRegistry));
                
                return result;
            });
        }
        
        return response;
    }
    
    // 2. 배치 처리를 통한 처리량 향상
    public List<String> batchProcess(List<String> items, String prompt, int batchSize) {
        return Lists.partition(items, batchSize).stream()
            .flatMap(batch -> {
                String batchPrompt = String.format(
                    "%s\n\nProcess these items:\n%s",
                    prompt,
                    String.join("\n", batch)
                );
                
                String response = chatClient.prompt()
                    .user(batchPrompt)
                    .call()
                    .content();
                
                // Parse batch response
                return parseBatchResponse(response).stream();
            })
            .collect(Collectors.toList());
    }
    
    // 3. 동적 타임아웃 조정
    public String adaptiveTimeout(String prompt, Duration initialTimeout) {
        AtomicReference<Duration> timeout = new AtomicReference<>(initialTimeout);
        
        return Retry.decorateSupplier(
            Retry.of("adaptive-retry", RetryConfig.custom()
                .maxAttempts(3)
                .waitDuration(Duration.ofSeconds(1))
                .build()),
            () -> {
                try {
                    return chatClient.prompt()
                        .user(prompt)
                        .options(ChatOptions.builder()
                            .timeout(timeout.get())
                            .build())
                        .call()
                        .content();
                } catch (TimeoutException e) {
                    // 타임아웃 발생 시 시간 증가
                    timeout.updateAndGet(d -> d.multipliedBy(2));
                    throw new RuntimeException(e);
                }
            }
        ).get();
    }
}
```

### 18.8.2 워크플로 모니터링

```java
@Component
public class WorkflowMonitor {
    
    private final MeterRegistry meterRegistry;
    private final ApplicationEventPublisher eventPublisher;
    
    @EventListener
    public void handleWorkflowEvent(WorkflowEvent event) {
        // 메트릭 수집
        meterRegistry.counter("workflow.executions",
            "type", event.getWorkflowType(),
            "status", event.getStatus()
        ).increment();
        
        if (event instanceof WorkflowCompletedEvent) {
            WorkflowCompletedEvent completed = (WorkflowCompletedEvent) event;
            
            meterRegistry.timer("workflow.duration",
                "type", completed.getWorkflowType()
            ).record(completed.getDuration());
            
            // 토큰 사용량 추적
            meterRegistry.gauge("workflow.tokens.used",
                Tags.of("type", completed.getWorkflowType()),
                completed.getTokensUsed()
            );
        }
    }
    
    // 실시간 모니터링 대시보드용 엔드포인트
    @RestController
    @RequestMapping("/workflow/metrics")
    public class MetricsController {
        
        @GetMapping("/current")
        public WorkflowMetrics getCurrentMetrics() {
            return WorkflowMetrics.builder()
                .activeWorkflows(getActiveWorkflowCount())
                .averageResponseTime(getAverageResponseTime())
                .errorRate(getErrorRate())
                .tokenUsageRate(getTokenUsageRate())
                .build();
        }
        
        @GetMapping("/health")
        public WorkflowHealth checkHealth() {
            return WorkflowHealth.builder()
                .status(determineHealthStatus())
                .latency(measureLatency())
                .throughput(calculateThroughput())
                .errorThreshold(checkErrorThreshold())
                .build();
        }
    }
}
```

### 18.8.3 비용 최적화

```java
@Service
public class CostOptimizationService {
    
    private final ChatClient chatClient;
    private final TokenCounter tokenCounter;
    
    public OptimizedResponse executeWithBudget(String task, double maxCost) {
        // 1. 작업 복잡도 평가
        TaskComplexity complexity = evaluateComplexity(task);
        
        // 2. 모델 선택 최적화
        String model = selectOptimalModel(complexity, maxCost);
        
        // 3. 프롬프트 압축
        String compressedPrompt = compressPrompt(task);
        
        // 4. 실행 및 비용 추적
        CostTracker costTracker = new CostTracker(maxCost);
        
        String response = chatClient.prompt()
            .user(compressedPrompt)
            .options(ChatOptions.builder()
                .model(model)
                .maxTokens(calculateMaxTokens(costTracker.getRemainingBudget()))
                .build())
            .call()
            .content();
        
        return OptimizedResponse.builder()
            .response(response)
            .actualCost(costTracker.getCurrentCost())
            .tokensUsed(tokenCounter.count(compressedPrompt + response))
            .modelUsed(model)
            .build();
    }
    
    private String compressPrompt(String original) {
        // 프롬프트 압축 기법 적용
        return chatClient.prompt()
            .system("Compress this prompt while preserving all essential information.")
            .user(original)
            .options(ChatOptions.builder()
                .model("gpt-3.5-turbo") // 저렴한 모델 사용
                .build())
            .call()
            .content();
    }
}
```

## 18.9 테스트 전략

워크플로 패턴을 효과적으로 테스트하려면 단위 테스트와 통합 테스트 전략이 필요합니다.

### 18.9.1 워크플로 단위 테스트

```java
@ExtendWith(MockitoExtension.class)
class WorkflowPatternTest {
    
    @Mock
    private ChatClient chatClient;
    
    @Mock
    private ChatClient.ChatClientRequest chatClientRequest;
    
    @Mock
    private ChatClient.CallResponseSpec callResponseSpec;
    
    @BeforeEach
    void setUp() {
        // ChatClient 모킹 체인 설정
        when(chatClient.prompt()).thenReturn(chatClientRequest);
        when(chatClientRequest.system(anyString())).thenReturn(chatClientRequest);
        when(chatClientRequest.user(anyString())).thenReturn(chatClientRequest);
        when(chatClientRequest.call()).thenReturn(callResponseSpec);
    }
    
    @Test
    void testChainWorkflow() {
        // Given
        String[] responses = {"Step 1 complete", "Step 2 complete", "Final result"};
        AtomicInteger callCount = new AtomicInteger(0);
        
        when(callResponseSpec.content()).thenAnswer(invocation -> 
            responses[callCount.getAndIncrement()]
        );
        
        // When
        ChainWorkflow workflow = new ChainWorkflow(chatClient);
        String result = workflow.chain(
            "Initial input",
            "Process step 1",
            "Process step 2",
            "Finalize"
        );
        
        // Then
        assertEquals("Final result", result);
        verify(chatClient, times(3)).prompt();
    }
    
    @Test
    void testParallelizationWorkflow() {
        // Given
        List<String> items = Arrays.asList("Item1", "Item2", "Item3");
        when(callResponseSpec.content()).thenReturn("Processed");
        
        // When
        ParallelizationWorkflow workflow = new ParallelizationWorkflow(chatClient);
        List<String> results = workflow.parallel(
            "Process each item",
            items,
            3
        );
        
        // Then
        assertEquals(3, results.size());
        results.forEach(result -> assertEquals("Processed", result));
    }
}
```

### 18.9.2 통합 테스트

```java
@SpringBootTest
@AutoConfigureMockMvc
class WorkflowIntegrationTest {
    
    @Autowired
    private ChatClient.Builder chatClientBuilder;
    
    @MockBean
    private ChatModel chatModel;
    
    @Test
    void testCompleteWorkflowExecution() {
        // Given
        ChatResponse mockResponse = new ChatResponse(
            List.of(new Generation(new AssistantMessage("Test response")))
        );
        
        when(chatModel.call(any(Prompt.class))).thenReturn(mockResponse);
        
        ChatClient chatClient = chatClientBuilder.build();
        
        // When - 복합 워크플로 실행
        OrchestratorWorkersWorkflow workflow = new OrchestratorWorkersWorkflow(chatClient);
        WorkerResponse response = workflow.process(
            "Analyze the impact of climate change on global agriculture"
        );
        
        // Then
        assertNotNull(response);
        assertNotNull(response.getAnalysis());
        assertFalse(response.getWorkerResponses().isEmpty());
        assertNotNull(response.getFinalResponse());
    }
}
```

### 18.9.3 성능 테스트

```java
@Test
class WorkflowPerformanceTest {
    
    @Test
    void measureWorkflowLatency() {
        ChatClient chatClient = createTestChatClient();
        
        // 각 패턴의 지연 시간 측정
        Map<String, Duration> latencies = new HashMap<>();
        
        // Chain Workflow
        Instant start = Instant.now();
        new ChainWorkflow(chatClient).chain(
            "Test input",
            "Step 1", "Step 2", "Step 3"
        );
        latencies.put("Chain", Duration.between(start, Instant.now()));
        
        // Parallelization Workflow
        start = Instant.now();
        new ParallelizationWorkflow(chatClient).parallel(
            "Process",
            Arrays.asList("A", "B", "C"),
            3
        );
        latencies.put("Parallel", Duration.between(start, Instant.now()));
        
        // 결과 분석
        latencies.forEach((pattern, duration) -> {
            System.out.printf("%s: %d ms%n", pattern, duration.toMillis());
            assertTrue(duration.toMillis() < 5000, 
                pattern + " should complete within 5 seconds");
        });
    }
}

## 18.10 결론

이 장에서는 Anthropic의 "Building Effective Agents" 연구를 바탕으로 Spring AI를 사용하여 효과적인 에이전트 워크플로를 구축하는 방법을 살펴보았습니다. 

### 핵심 요점

1. **단순성과 구성 가능성**: 복잡한 프레임워크보다 단순하고 구성 가능한 패턴이 더 효과적입니다.

2. **5가지 기본 패턴**:
   - Chain: 순차적 작업 처리
   - Parallelization: 병렬 작업 실행
   - Routing: 지능적 작업 분배
   - Orchestrator-Workers: 동적 작업 분해
   - Evaluator-Optimizer: 반복적 개선

3. **Spring AI의 장점**: 모델 이식성, 구조화된 출력, 일관된 API를 통해 패턴 구현이 용이합니다.

4. **실전 활용**: 개별 패턴을 조합하여 복잡한 비즈니스 요구사항을 해결할 수 있습니다.

### 향후 발전 방향

- 고급 메모리 관리 기법
- 도구 통합 및 MCP(Model Context Protocol) 활용
- 더 정교한 패턴 조합 전략
- 실시간 적응형 워크플로

효과적인 에이전트 시스템을 구축하는 핵심은 복잡성을 추가하는 것이 아니라, 적절한 패턴을 선택하고 조합하는 것입니다.

## 18.11 연습 문제

1. **Chain Workflow 실습**: 뉴스 기사를 분석하는 3단계 체인을 구현하세요 (요약 → 감정 분석 → 핵심 인사이트 추출).

2. **Parallelization 활용**: 제품 리뷰를 여러 관점(품질, 가격, 디자인, 서비스)에서 동시에 분석하는 워크플로를 구현하세요.

3. **Routing Pattern**: 고객 문의를 카테고리별로 분류하고 적절한 전문 프롬프트로 라우팅하는 시스템을 구축하세요.

4. **Orchestrator-Workers**: 복잡한 연구 주제를 하위 질문으로 분해하고 병렬로 조사한 후 통합하는 워크플로를 구현하세요.

5. **Evaluator-Optimizer**: 마케팅 카피를 반복적으로 개선하는 시스템을 구축하세요 (명확성, 설득력, 간결성 기준).

6. **패턴 조합**: 위의 패턴 중 2개 이상을 조합하여 실제 비즈니스 문제를 해결하는 복합 워크플로를 설계하고 구현하세요.