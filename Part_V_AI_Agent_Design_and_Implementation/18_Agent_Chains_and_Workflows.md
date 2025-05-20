# 18장: 에이전트 체인 & 워크플로 구축

## 18.1 에이전트 체인과 워크플로 개요

AI 에이전트 체인과 워크플로는 여러 전문 에이전트를 조합하여 복잡한 작업을 단계별로 처리하는 방법을 제공합니다. 이 장에서는 Spring AI에서 에이전트 체인과 워크플로를 구축하고 관리하는 방법을 살펴보겠습니다.

### 18.1.1 에이전트 체인의 필요성

- 복잡한 문제를 작은 단계로 분해
- 전문화된 에이전트의 장점 활용
- 확장성과 재사용성 증대
- 실패 처리와 오류 복구 강화

### 18.1.2 에이전트 워크플로의 종류

- 선형 체인 (Linear Chain)
- 분기 워크플로 (Branching Workflow)
- 병렬 처리 (Parallel Processing)
- 반복 작업 (Iterative Processing)
- 이벤트 기반 워크플로 (Event-driven Workflow)

### 18.1.3 Spring AI의 워크플로 지원

```java
public interface AgentWorkflow<I, O> {
    O execute(I input);
    List<Agent> getAgents();
    WorkflowMetadata getMetadata();
}
```

## 18.2 기본 에이전트 체인 구현

가장 단순한 형태의 에이전트 체인은 여러 에이전트를 순차적으로 실행하는 선형 체인입니다. 이 섹션에서는 기본적인 에이전트 체인을 구현하는 방법을 알아봅니다.

### 18.2.1 LinearAgentChain 구현

```java
public class LinearAgentChain implements AgentWorkflow<AgentRequest, AgentResponse> {
    
    private final List<Agent> agents;
    private final boolean stopOnError;
    
    public LinearAgentChain(List<Agent> agents, boolean stopOnError) {
        this.agents = new ArrayList<>(agents);
        this.stopOnError = stopOnError;
    }
    
    @Override
    public AgentResponse execute(AgentRequest initialRequest) {
        AgentRequest currentRequest = initialRequest;
        AgentResponse finalResponse = null;
        
        for (Agent agent : agents) {
            try {
                AgentResponse response = agent.execute(currentRequest);
                finalResponse = response;
                
                // 다음 에이전트의 입력으로 현재 응답을 변환
                currentRequest = transformResponseToRequest(response);
                
            } catch (Exception e) {
                if (stopOnError) {
                    throw new AgentWorkflowException("Chain execution failed at agent: " + agent, e);
                }
                // 오류 발생 시에도 계속 진행하는 경우 로깅 및 처리
                log.error("Error in agent chain execution: {}", e.getMessage(), e);
            }
        }
        
        return finalResponse;
    }
    
    private AgentRequest transformResponseToRequest(AgentResponse response) {
        // 이전 에이전트의 응답을 다음 에이전트의 요청으로 변환
        return AgentRequest.builder()
            .prompt(response.getContent())
            .messages(response.getMessages())
            .metadata(response.getMetadata())
            .build();
    }
    
    @Override
    public List<Agent> getAgents() {
        return Collections.unmodifiableList(agents);
    }
    
    @Override
    public WorkflowMetadata getMetadata() {
        return WorkflowMetadata.builder()
            .name("LinearAgentChain")
            .description("Sequential execution of multiple agents")
            .build();
    }
}
```

### 18.2.2 에이전트 체인 생성 및 사용

```java
@Service
public class SimpleChainService {
    
    private final Agent researchAgent;
    private final Agent analysisAgent;
    private final Agent reportingAgent;
    
    public String processResearchTopic(String topic) {
        // 선형 체인 생성
        LinearAgentChain researchChain = new LinearAgentChain(
            Arrays.asList(researchAgent, analysisAgent, reportingAgent),
            true // 오류 발생 시 중단
        );
        
        // 초기 요청 생성
        AgentRequest initialRequest = AgentRequest.builder()
            .prompt("Research the following topic: " + topic)
            .build();
        
        // 체인 실행
        AgentResponse result = researchChain.execute(initialRequest);
        
        return result.getContent();
    }
}
```

### 18.2.3 체인 결과 처리 및 포맷팅

```java
public class ChainResultProcessor {
    
    public ReportDocument processChainResult(AgentResponse chainResponse, String format) {
        String content = chainResponse.getContent();
        List<ToolExecution> toolExecutions = chainResponse.getToolExecutions();
        
        // 결과 처리 로직
        Map<String, Object> processedData = extractDataFromResponse(content);
        List<Attachment> attachments = generateAttachmentsFromToolExecutions(toolExecutions);
        
        // 요청된 형식으로 변환
        return formatReport(processedData, attachments, format);
    }
    
    private Map<String, Object> extractDataFromResponse(String content) {
        // 응답 내용에서 구조화된 데이터 추출
    }
    
    private List<Attachment> generateAttachmentsFromToolExecutions(List<ToolExecution> toolExecutions) {
        // 도구 실행 결과에서 첨부 파일 생성
    }
    
    private ReportDocument formatReport(Map<String, Object> data, List<Attachment> attachments, String format) {
        // 지정된 형식으로 보고서 생성 (HTML, PDF, Markdown 등)
    }
}
```

## 18.3 분기 워크플로 구현

복잡한 의사 결정이 필요한 경우 조건에 따라 다른 에이전트로 처리를 전달하는 분기 워크플로를 구현할 수 있습니다.

### 18.3.1 BranchingAgentWorkflow 구현

```java
public class BranchingAgentWorkflow implements AgentWorkflow<AgentRequest, AgentResponse> {
    
    private final Agent initialAgent;
    private final Map<String, Agent> branchAgents;
    private final Function<AgentResponse, String> branchSelector;
    private final Agent defaultAgent;
    
    public BranchingAgentWorkflow(
            Agent initialAgent,
            Map<String, Agent> branchAgents,
            Function<AgentResponse, String> branchSelector,
            Agent defaultAgent) {
        this.initialAgent = initialAgent;
        this.branchAgents = new HashMap<>(branchAgents);
        this.branchSelector = branchSelector;
        this.defaultAgent = defaultAgent;
    }
    
    @Override
    public AgentResponse execute(AgentRequest request) {
        // 초기 에이전트 실행
        AgentResponse initialResponse = initialAgent.execute(request);
        
        // 분기 결정
        String branch = branchSelector.apply(initialResponse);
        
        // 선택된 분기에 따른 에이전트 실행
        Agent nextAgent = branchAgents.getOrDefault(branch, defaultAgent);
        AgentRequest nextRequest = transformResponseToRequest(initialResponse);
        
        return nextAgent.execute(nextRequest);
    }
    
    private AgentRequest transformResponseToRequest(AgentResponse response) {
        // 이전 에이전트의 응답을 다음 에이전트의 요청으로 변환
    }
    
    @Override
    public List<Agent> getAgents() {
        List<Agent> allAgents = new ArrayList<>();
        allAgents.add(initialAgent);
        allAgents.addAll(branchAgents.values());
        if (defaultAgent != null && !branchAgents.containsValue(defaultAgent)) {
            allAgents.add(defaultAgent);
        }
        return Collections.unmodifiableList(allAgents);
    }
    
    @Override
    public WorkflowMetadata getMetadata() {
        return WorkflowMetadata.builder()
            .name("BranchingAgentWorkflow")
            .description("Conditional branching workflow based on agent response")
            .build();
    }
}
```

### 18.3.2 분기 선택 로직 구현

```java
@Service
public class CustomerQueryRouter {
    
    private final Agent classifierAgent;
    private final Agent productSupportAgent;
    private final Agent billingAgent;
    private final Agent technicalSupportAgent;
    private final Agent generalInquiryAgent;
    
    public AgentResponse routeCustomerQuery(String query) {
        // 분기 워크플로 생성
        Function<AgentResponse, String> branchSelector = response -> {
            // 분류 에이전트의 응답에서 카테고리 추출
            String content = response.getContent().toLowerCase();
            if (content.contains("product") || content.contains("item")) {
                return "product";
            } else if (content.contains("bill") || content.contains("payment")) {
                return "billing";
            } else if (content.contains("tech") || content.contains("error")) {
                return "technical";
            } else {
                return "general";
            }
        };
        
        Map<String, Agent> branches = Map.of(
            "product", productSupportAgent,
            "billing", billingAgent,
            "technical", technicalSupportAgent
        );
        
        BranchingAgentWorkflow workflow = new BranchingAgentWorkflow(
            classifierAgent,
            branches,
            branchSelector,
            generalInquiryAgent
        );
        
        // 워크플로 실행
        return workflow.execute(AgentRequest.builder().prompt(query).build());
    }
}
```

### 18.3.3 복잡한 분기 패턴 구현

```java
@Component
public class DecisionTreeWorkflow implements AgentWorkflow<AgentRequest, AgentResponse> {
    
    private final Agent rootAgent;
    private final Map<String, DecisionNode> decisionTree;
    
    @Override
    public AgentResponse execute(AgentRequest request) {
        DecisionNode currentNode = decisionTree.get("root");
        AgentResponse finalResponse = null;
        
        while (currentNode != null) {
            // 현재 노드의 에이전트 실행
            AgentRequest nodeRequest = currentNode.transformRequest(request, finalResponse);
            finalResponse = currentNode.getAgent().execute(nodeRequest);
            
            // 다음 노드 결정
            String nextNodeKey = currentNode.determineNextNode(finalResponse);
            currentNode = nextNodeKey != null ? decisionTree.get(nextNodeKey) : null;
        }
        
        return finalResponse;
    }
    
    // Decision node 클래스 (중첩 클래스 또는 별도 클래스로 구현)
    public static class DecisionNode {
        private final Agent agent;
        private final Function<AgentResponse, String> nextNodeSelector;
        
        public Agent getAgent() {
            return agent;
        }
        
        public String determineNextNode(AgentResponse response) {
            return nextNodeSelector.apply(response);
        }
        
        public AgentRequest transformRequest(AgentRequest originalRequest, AgentResponse previousResponse) {
            // 이전 응답을 고려하여 현재 노드에 대한 요청 변환
        }
    }
}
```

## 18.4 병렬 에이전트 워크플로 구현

독립적인 작업을 동시에 처리해야 하는 경우 병렬 에이전트 워크플로를 사용할 수 있습니다.

### 18.4.1 ParallelAgentWorkflow 구현

```java
public class ParallelAgentWorkflow implements AgentWorkflow<AgentRequest, List<AgentResponse>> {
    
    private final List<Agent> agents;
    private final Function<AgentRequest, List<AgentRequest>> requestSplitter;
    private final Executor executor;
    
    public ParallelAgentWorkflow(
            List<Agent> agents,
            Function<AgentRequest, List<AgentRequest>> requestSplitter,
            Executor executor) {
        this.agents = new ArrayList<>(agents);
        this.requestSplitter = requestSplitter;
        this.executor = executor;
    }
    
    @Override
    public List<AgentResponse> execute(AgentRequest request) {
        // 요청 분할
        List<AgentRequest> splitRequests = requestSplitter.apply(request);
        
        // 에이전트 수와 요청 수가 일치하는지 확인
        if (splitRequests.size() != agents.size()) {
            throw new IllegalStateException("Number of split requests must match number of agents");
        }
        
        // 병렬 처리를 위한 CompletableFuture 생성
        List<CompletableFuture<AgentResponse>> futures = new ArrayList<>();
        
        for (int i = 0; i < agents.size(); i++) {
            final int index = i;
            CompletableFuture<AgentResponse> future = CompletableFuture.supplyAsync(
                () -> agents.get(index).execute(splitRequests.get(index)),
                executor
            );
            futures.add(future);
        }
        
        // 모든 작업 완료 대기
        CompletableFuture<Void> allFutures = CompletableFuture.allOf(
            futures.toArray(new CompletableFuture[0])
        );
        
        // 결과 수집
        try {
            allFutures.get();
            return futures.stream()
                .map(f -> {
                    try {
                        return f.get();
                    } catch (Exception e) {
                        throw new AgentWorkflowException("Error getting parallel execution result", e);
                    }
                })
                .collect(Collectors.toList());
        } catch (Exception e) {
            throw new AgentWorkflowException("Error in parallel workflow execution", e);
        }
    }
    
    @Override
    public List<Agent> getAgents() {
        return Collections.unmodifiableList(agents);
    }
    
    @Override
    public WorkflowMetadata getMetadata() {
        return WorkflowMetadata.builder()
            .name("ParallelAgentWorkflow")
            .description("Parallel execution of multiple agents")
            .build();
    }
}
```

### 18.4.2 병렬 작업 분배 및 결과 병합

```java
@Service
public class ComprehensiveAnalysisService {
    
    private final Agent financialAnalysisAgent;
    private final Agent marketAnalysisAgent;
    private final Agent competitorAnalysisAgent;
    private final Agent trendAnalysisAgent;
    
    public AnalysisReport analyzeCompany(String companyName) {
        // 병렬 요청 분배기 정의
        Function<AgentRequest, List<AgentRequest>> requestSplitter = request -> {
            String basePrompt = "Analyze " + companyName + " ";
            
            return Arrays.asList(
                AgentRequest.builder().prompt(basePrompt + "financial data and performance").build(),
                AgentRequest.builder().prompt(basePrompt + "market position and share").build(),
                AgentRequest.builder().prompt(basePrompt + "competitors and competitive advantage").build(),
                AgentRequest.builder().prompt(basePrompt + "industry trends and future outlook").build()
            );
        };
        
        // 병렬 워크플로 생성
        ParallelAgentWorkflow workflow = new ParallelAgentWorkflow(
            Arrays.asList(
                financialAnalysisAgent,
                marketAnalysisAgent,
                competitorAnalysisAgent,
                trendAnalysisAgent
            ),
            requestSplitter,
            Executors.newFixedThreadPool(4)
        );
        
        // 워크플로 실행
        List<AgentResponse> results = workflow.execute(
            AgentRequest.builder().prompt("Comprehensive analysis of " + companyName).build()
        );
        
        // 결과 병합
        return mergeAnalysisResults(results, companyName);
    }
    
    private AnalysisReport mergeAnalysisResults(List<AgentResponse> results, String companyName) {
        // 각 에이전트의 결과를 종합하여 최종 보고서 생성
        AnalysisReport report = new AnalysisReport();
        report.setCompanyName(companyName);
        report.setGeneratedDate(LocalDate.now());
        
        // 결과 처리 로직...
        
        return report;
    }
}
```

### 18.4.3 오류 처리 및 결과 시각화

```java
@Component
public class ParallelWorkflowMonitor {
    
    public <T> CompletableFuture<List<T>> executeWithMonitoring(
            List<CompletableFuture<T>> futures,
            Consumer<Map<Integer, WorkflowTaskStatus>> statusUpdateHandler) {
        
        // 각 작업의 상태를 추적하는 맵
        Map<Integer, WorkflowTaskStatus> taskStatusMap = new ConcurrentHashMap<>();
        for (int i = 0; i < futures.size(); i++) {
            taskStatusMap.put(i, new WorkflowTaskStatus(i, "PENDING"));
        }
        
        // 상태 업데이트 핸들러 호출
        statusUpdateHandler.accept(Collections.unmodifiableMap(taskStatusMap));
        
        // 각 작업에 완료 및 오류 핸들러 추가
        List<CompletableFuture<T>> monitoredFutures = new ArrayList<>();
        for (int i = 0; i < futures.size(); i++) {
            final int index = i;
            CompletableFuture<T> monitoredFuture = futures.get(i)
                .thenApply(result -> {
                    taskStatusMap.put(index, new WorkflowTaskStatus(index, "COMPLETED"));
                    statusUpdateHandler.accept(Collections.unmodifiableMap(taskStatusMap));
                    return result;
                })
                .exceptionally(ex -> {
                    taskStatusMap.put(index, new WorkflowTaskStatus(index, "FAILED", ex.getMessage()));
                    statusUpdateHandler.accept(Collections.unmodifiableMap(taskStatusMap));
                    throw new CompletionException(ex);
                });
            
            monitoredFutures.add(monitoredFuture);
        }
        
        // 모든 작업이 완료되면 결과 반환
        return CompletableFuture.allOf(monitoredFutures.toArray(new CompletableFuture[0]))
            .thenApply(v -> monitoredFutures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList()));
    }
    
    public static class WorkflowTaskStatus {
        private final int taskId;
        private final String status;
        private final String errorMessage;
        
        // 생성자, 게터 등...
    }
}
```

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

## 18.7 워크플로 오케스트레이션 및 관리

복잡한 워크플로를 관리하고 모니터링하기 위한 오케스트레이션 도구와 관리 시스템을 구현할 수 있습니다.

### 18.7.1 WorkflowOrchestrator 구현

```java
@Service
public class WorkflowOrchestrator {
    
    private final Map<String, AgentWorkflow<?, ?>> registeredWorkflows;
    private final WorkflowStateRepository stateRepository;
    private final WorkflowMonitor monitor;
    
    public <I, O> String startWorkflow(String workflowName, I input) {
        // 워크플로 검색
        AgentWorkflow<I, O> workflow = (AgentWorkflow<I, O>) registeredWorkflows.get(workflowName);
        if (workflow == null) {
            throw new IllegalArgumentException("Workflow not found: " + workflowName);
        }
        
        // 워크플로 인스턴스 ID 생성
        String instanceId = UUID.randomUUID().toString();
        
        // 워크플로 상태 초기화
        WorkflowState state = new WorkflowState();
        state.setInstanceId(instanceId);
        state.setWorkflowName(workflowName);
        state.setStatus("RUNNING");
        state.setStartTime(LocalDateTime.now());
        stateRepository.save(state);
        
        // 비동기 실행
        CompletableFuture.runAsync(() -> {
            try {
                // 워크플로 실행
                O result = workflow.execute(input);
                
                // 성공 처리
                state.setStatus("COMPLETED");
                state.setEndTime(LocalDateTime.now());
                state.setResult(result);
                stateRepository.save(state);
                
                // 모니터링 이벤트 발행
                monitor.workflowCompleted(instanceId, workflow, result);
                
            } catch (Exception e) {
                // 실패 처리
                state.setStatus("FAILED");
                state.setEndTime(LocalDateTime.now());
                state.setErrorMessage(e.getMessage());
                stateRepository.save(state);
                
                // 모니터링 이벤트 발행
                monitor.workflowFailed(instanceId, workflow, e);
            }
        });
        
        return instanceId;
    }
    
    public WorkflowState getWorkflowState(String instanceId) {
        return stateRepository.findById(instanceId)
            .orElseThrow(() -> new IllegalArgumentException("Workflow instance not found: " + instanceId));
    }
    
    public void cancelWorkflow(String instanceId) {
        // 워크플로 취소 로직
    }
    
    public void registerWorkflow(String name, AgentWorkflow<?, ?> workflow) {
        registeredWorkflows.put(name, workflow);
    }
}
```

### 18.7.2 워크플로 상태 관리 및 모니터링

```java
@Component
public class WorkflowMonitor {
    
    private final ApplicationEventPublisher eventPublisher;
    private final WorkflowMetricsCollector metricsCollector;
    
    public <O> void workflowCompleted(String instanceId, AgentWorkflow<?, O> workflow, O result) {
        // 완료 이벤트 발행
        eventPublisher.publishEvent(
            new WorkflowCompletedEvent(this, instanceId, workflow.getMetadata(), result)
        );
        
        // 메트릭 기록
        metricsCollector.recordWorkflowExecution(
            workflow.getMetadata().getName(),
            "COMPLETED",
            calculateDuration(instanceId)
        );
    }
    
    public void workflowFailed(String instanceId, AgentWorkflow<?, ?> workflow, Exception exception) {
        // 실패 이벤트 발행
        eventPublisher.publishEvent(
            new WorkflowFailedEvent(this, instanceId, workflow.getMetadata(), exception)
        );
        
        // 메트릭 기록
        metricsCollector.recordWorkflowExecution(
            workflow.getMetadata().getName(),
            "FAILED",
            calculateDuration(instanceId)
        );
    }
    
    private Duration calculateDuration(String instanceId) {
        // 워크플로 실행 시간 계산
    }
}
```

### 18.7.3 워크플로 시각화 및 대시보드

```java
@RestController
@RequestMapping("/workflow")
public class WorkflowDashboardController {
    
    private final WorkflowOrchestrator orchestrator;
    private final WorkflowMetricsCollector metricsCollector;
    private final WorkflowVisualizer visualizer;
    
    @GetMapping("/instances")
    public List<WorkflowInstanceDto> getAllInstances() {
        // 모든 워크플로 인스턴스 조회
    }
    
    @GetMapping("/instances/{id}")
    public WorkflowDetailsDto getWorkflowDetails(@PathVariable String id) {
        // 특정 워크플로 인스턴스 상세 정보 조회
    }
    
    @GetMapping("/metrics")
    public WorkflowMetricsDto getWorkflowMetrics() {
        // 워크플로 메트릭 조회
    }
    
    @GetMapping("/instances/{id}/visualize")
    public WorkflowVisualizationDto visualizeWorkflow(@PathVariable String id) {
        // 워크플로 시각화 데이터 생성
        return visualizer.generateVisualization(id);
    }
}
```

## 18.8 실전 패턴: 복합 워크플로 구현

여러 워크플로 패턴을 조합하여 복잡한 비즈니스 프로세스를 구현하는 방법을 알아봅니다.

### 18.8.1 마케팅 캠페인 분석 워크플로

다양한 데이터 소스를 수집하고 분석하여 마케팅 캠페인의 효과를 평가하는 복합 워크플로를 구현합니다.

```java
@Service
public class MarketingCampaignAnalysisService {
    
    private final Agent dataCollectionAgent;
    private final Agent dataPreprocessingAgent;
    private final Agent segmentationAgent;
    private final Agent performanceAnalysisAgent;
    private final Agent insightGenerationAgent;
    private final Agent recommendationAgent;
    
    public CampaignAnalysisReport analyzeCampaign(String campaignId) {
        // 1. 데이터 수집 단계
        LinearAgentChain dataCollectionChain = new LinearAgentChain(
            Arrays.asList(dataCollectionAgent, dataPreprocessingAgent),
            true
        );
        
        AgentRequest initialRequest = AgentRequest.builder()
            .prompt("Collect and preprocess data for campaign: " + campaignId)
            .build();
        
        AgentResponse preprocessedData = dataCollectionChain.execute(initialRequest);
        
        // 2. 병렬 분석 단계
        ParallelAgentWorkflow analysisWorkflow = createAnalysisWorkflow(preprocessedData);
        List<AgentResponse> analysisResults = analysisWorkflow.execute(
            AgentRequest.builder()
                .prompt("Run parallel analysis on campaign data")
                .metadata(Map.of("preprocessedData", preprocessedData.getContent()))
                .build()
        );
        
        // 3. 최종 권장사항 및 보고서 생성 단계
        BranchingAgentWorkflow recommendationWorkflow = createRecommendationWorkflow(analysisResults);
        AgentResponse finalRecommendation = recommendationWorkflow.execute(
            AgentRequest.builder()
                .prompt("Generate final insights and recommendations")
                .metadata(Map.of("analysisResults", analysisResults))
                .build()
        );
        
        // 결과 변환 및 반환
        return generateFinalReport(campaignId, preprocessedData, analysisResults, finalRecommendation);
    }
    
    private ParallelAgentWorkflow createAnalysisWorkflow(AgentResponse preprocessedData) {
        // 병렬 분석 워크플로 구성
    }
    
    private BranchingAgentWorkflow createRecommendationWorkflow(List<AgentResponse> analysisResults) {
        // 분기 권장사항 워크플로 구성
    }
    
    private CampaignAnalysisReport generateFinalReport(
            String campaignId,
            AgentResponse preprocessedData,
            List<AgentResponse> analysisResults,
            AgentResponse recommendation) {
        // 최종 보고서 생성
    }
}
```

### 18.8.2 문서 처리 및 지식 추출 워크플로

대량의 문서에서 지식을 추출하고 구조화하는 복합 워크플로를 구현합니다.

```java
@Service
public class DocumentProcessingService {
    
    private final Agent documentClassifierAgent;
    private final Agent textExtractionAgent;
    private final Agent structuringAgent;
    private final Agent entityExtractionAgent;
    private final Agent relationExtractionAgent;
    private final Agent knowledgeBaseUpdaterAgent;
    
    public void processDocumentBatch(List<Document> documents) {
        // 문서 분류 워크플로 구성
        BranchingAgentWorkflow classificationWorkflow = new BranchingAgentWorkflow(
            documentClassifierAgent,
            classifyDocumentsByType(),
            Map.of(
                "invoice", createInvoiceProcessingChain(),
                "contract", createContractProcessingChain(),
                "report", createReportProcessingChain()
            ),
            createGeneralDocumentProcessingChain()
        );
        
        // 각 문서 처리
        for (Document document : documents) {
            AgentRequest request = AgentRequest.builder()
                .prompt("Process document: " + document.getTitle())
                .metadata(Map.of("document", document))
                .build();
            
            try {
                // 분류 및 처리 워크플로 실행
                AgentResponse result = classificationWorkflow.execute(request);
                
                // 추출된 지식을 지식 베이스에 업데이트
                updateKnowledgeBase(document, result);
                
            } catch (Exception e) {
                // 문서 처리 실패 처리
                log.error("Failed to process document: {}", document.getTitle(), e);
            }
        }
    }
    
    private Function<AgentResponse, String> classifyDocumentsByType() {
        // 문서 유형에 따른 분류 로직
    }
    
    private Agent createInvoiceProcessingChain() {
        // 청구서 처리 체인 구성
    }
    
    private Agent createContractProcessingChain() {
        // 계약서 처리 체인 구성
    }
    
    private Agent createReportProcessingChain() {
        // 보고서 처리 체인 구성
    }
    
    private Agent createGeneralDocumentProcessingChain() {
        // 일반 문서 처리 체인 구성
    }
    
    private void updateKnowledgeBase(Document document, AgentResponse processingResult) {
        // 지식 베이스 업데이트 로직
    }
}
```

### 18.8.3 고객 여정 최적화 워크플로

고객 데이터를 분석하고 개인화된 여정을 최적화하는 복합 워크플로를 구현합니다.

```java
@Service
public class CustomerJourneyOptimizationService {
    
    private final Agent customerProfilerAgent;
    private final Agent behaviorAnalysisAgent;
    private final Agent segmentationAgent;
    private final Agent journeyMappingAgent;
    private final Agent touchpointOptimizationAgent;
    private final Agent personalizedContentAgent;
    
    public OptimizedJourneyPlan optimizeCustomerJourney(String customerId) {
        // 고객 프로필링 및 행동 분석 (선형 체인)
        LinearAgentChain profilingChain = new LinearAgentChain(
            Arrays.asList(customerProfilerAgent, behaviorAnalysisAgent),
            true
        );
        
        AgentResponse profileData = profilingChain.execute(
            AgentRequest.builder()
                .prompt("Profile customer: " + customerId)
                .build()
        );
        
        // 세분화 및 여정 매핑 (반복적 개선)
        IterativeAgentWorkflow segmentationWorkflow = new IterativeAgentWorkflow(
            segmentationAgent,
            response -> needsRefinement(response),
            response -> createRefinementRequest(response),
            3 // 최대 3회 반복
        );
        
        AgentResponse segmentationResult = segmentationWorkflow.execute(
            AgentRequest.builder()
                .prompt("Segment customer based on profile data")
                .metadata(Map.of("profileData", profileData.getContent()))
                .build()
        );
        
        // 접점 최적화 및 개인화 콘텐츠 추천 (병렬 처리)
        ParallelAgentWorkflow optimizationWorkflow = new ParallelAgentWorkflow(
            Arrays.asList(journeyMappingAgent, touchpointOptimizationAgent, personalizedContentAgent),
            request -> createOptimizationRequests(request, segmentationResult),
            Executors.newFixedThreadPool(3)
        );
        
        List<AgentResponse> optimizationResults = optimizationWorkflow.execute(
            AgentRequest.builder()
                .prompt("Optimize customer journey")
                .metadata(Map.of(
                    "profileData", profileData.getContent(),
                    "segmentationResult", segmentationResult.getContent()
                ))
                .build()
        );
        
        // 최종 최적화된 여정 계획 생성
        return generateOptimizedJourneyPlan(customerId, profileData, segmentationResult, optimizationResults);
    }
    
    private boolean needsRefinement(AgentResponse response) {
        // 세분화 결과가 충분히 정교한지 평가
    }
    
    private AgentRequest createRefinementRequest(AgentResponse response) {
        // 세분화 개선을 위한 요청 생성
    }
    
    private List<AgentRequest> createOptimizationRequests(
            AgentRequest originalRequest,
            AgentResponse segmentationResult) {
        // 최적화 작업을 위한 병렬 요청 생성
    }
    
    private OptimizedJourneyPlan generateOptimizedJourneyPlan(
            String customerId,
            AgentResponse profileData,
            AgentResponse segmentationResult,
            List<AgentResponse> optimizationResults) {
        // 최적화된 여정 계획 생성
    }
}
```

## 18.9 단위 테스트 및 통합 테스트

에이전트 체인과 워크플로를 효과적으로 테스트하는 방법을 알아봅니다.

### 18.9.1 워크플로 단위 테스트 구현

```java
@ExtendWith(MockitoExtension.class)
public class LinearAgentChainTest {
    
    @Mock
    private Agent firstAgent;
    
    @Mock
    private Agent secondAgent;
    
    @Test
    public void testLinearChainExecution() {
        // 테스트 요청 및 응답 설정
        AgentRequest initialRequest = AgentRequest.builder()
            .prompt("Initial request")
            .build();
        
        AgentResponse firstResponse = AgentResponse.builder()
            .content("First agent response")
            .build();
        
        AgentRequest expectedSecondRequest = AgentRequest.builder()
            .prompt("First agent response")
            .build();
        
        AgentResponse secondResponse = AgentResponse.builder()
            .content("Second agent response")
            .build();
        
        // Mock 설정
        when(firstAgent.execute(initialRequest)).thenReturn(firstResponse);
        when(secondAgent.execute(any())).thenReturn(secondResponse);
        
        // 체인 생성 및 실행
        LinearAgentChain chain = new LinearAgentChain(
            Arrays.asList(firstAgent, secondAgent),
            true
        );
        
        AgentResponse result = chain.execute(initialRequest);
        
        // 검증
        verify(firstAgent).execute(initialRequest);
        verify(secondAgent).execute(argThat(request ->
            request.getPrompt().equals(expectedSecondRequest.getPrompt())
        ));
        
        assertEquals("Second agent response", result.getContent());
    }
    
    @Test
    public void testErrorHandlingWithStopOnError() {
        // Mock 설정 - 첫 번째 에이전트에서 오류 발생
        when(firstAgent.execute(any())).thenThrow(new RuntimeException("Test error"));
        
        // 체인 생성 및 실행
        LinearAgentChain chain = new LinearAgentChain(
            Arrays.asList(firstAgent, secondAgent),
            true // 오류 발생 시 중단
        );
        
        // 검증 - 예외 발생 확인
        assertThrows(AgentWorkflowException.class, () ->
            chain.execute(AgentRequest.builder().prompt("Test").build())
        );
        
        // 두 번째 에이전트는 호출되지 않음
        verify(secondAgent, never()).execute(any());
    }
}
```

### 18.9.2 모의 에이전트를 이용한 워크플로 테스트

```java
@SpringBootTest
public class AgentWorkflowIntegrationTest {
    
    @Autowired
    private WorkflowOrchestrator orchestrator;
    
    @MockBean
    private Agent researchAgent;
    
    @MockBean
    private Agent analysisAgent;
    
    @MockBean
    private Agent reportingAgent;
    
    @Test
    public void testResearchWorkflowEndToEnd() {
        // 모의 에이전트 동작 설정
        when(researchAgent.execute(any())).thenReturn(
            AgentResponse.builder()
                .content("Research data about topic")
                .build()
        );
        
        when(analysisAgent.execute(any())).thenReturn(
            AgentResponse.builder()
                .content("Analysis of research data")
                .build()
        );
        
        when(reportingAgent.execute(any())).thenReturn(
            AgentResponse.builder()
                .content("Final report")
                .build()
        );
        
        // 워크플로 실행
        String instanceId = orchestrator.startWorkflow(
            "researchWorkflow",
            AgentRequest.builder().prompt("Research topic").build()
        );
        
        // 완료 대기
        await().atMost(5, TimeUnit.SECONDS).until(
            () -> orchestrator.getWorkflowState(instanceId).getStatus().equals("COMPLETED")
        );
        
        // 결과 검증
        WorkflowState state = orchestrator.getWorkflowState(instanceId);
        assertEquals("COMPLETED", state.getStatus());
        
        AgentResponse result = (AgentResponse) state.getResult();
        assertEquals("Final report", result.getContent());
        
        // 에이전트 호출 순서 검증
        InOrder inOrder = inOrder(researchAgent, analysisAgent, reportingAgent);
        inOrder.verify(researchAgent).execute(any());
        inOrder.verify(analysisAgent).execute(any());
        inOrder.verify(reportingAgent).execute(any());
    }
}
```

### 18.9.3 성능 및 부하 테스트

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class WorkflowPerformanceTest {
    
    private WorkflowOrchestrator orchestrator;
    private WorkflowMetricsCollector metricsCollector;
    
    @BeforeEach
    public void setup() {
        // 테스트용 오케스트레이터 및 에이전트 설정
    }
    
    @Test
    @Order(1)
    public void testSingleWorkflowPerformance() {
        // 단일 워크플로 성능 측정
        long startTime = System.currentTimeMillis();
        
        orchestrator.startWorkflow("testWorkflow", createTestRequest());
        
        await().atMost(10, TimeUnit.SECONDS).until(
            () -> metricsCollector.getCompletedWorkflows("testWorkflow") == 1
        );
        
        long endTime = System.currentTimeMillis();
        long executionTime = endTime - startTime;
        
        // 성능 기준 검증
        assertTrue(executionTime < 5000, "Workflow execution should complete in less than 5 seconds");
    }
    
    @Test
    @Order(2)
    public void testConcurrentWorkflowPerformance() {
        // 동시 워크플로 부하 테스트
        int concurrentWorkflows = 10;
        CountDownLatch latch = new CountDownLatch(concurrentWorkflows);
        
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < concurrentWorkflows; i++) {
            int index = i;
            new Thread(() -> {
                try {
                    orchestrator.startWorkflow("testWorkflow", createTestRequest(index));
                } finally {
                    latch.countDown();
                }
            }).start();
        }
        
        try {
            latch.await(30, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            fail("Test interrupted");
        }
        
        await().atMost(30, TimeUnit.SECONDS).until(
            () -> metricsCollector.getCompletedWorkflows("testWorkflow") >= concurrentWorkflows
        );
        
        long endTime = System.currentTimeMillis();
        long totalExecutionTime = endTime - startTime;
        
        // 성능 기준 검증
        double avgExecutionTime = totalExecutionTime / (double) concurrentWorkflows;
        assertTrue(avgExecutionTime < 8000, "Average workflow execution should complete in less than 8 seconds under load");
    }
}
```

## 18.10 결론

이 장에서는 Spring AI를 사용하여 다양한 유형의 에이전트 체인과 워크플로를 구축하는 방법을 살펴보았습니다. 선형 체인, 분기 워크플로, 병렬 처리, 반복 워크플로, 이벤트 기반 워크플로 등 다양한 패턴을 사용하여 복잡한 AI 애플리케이션을 구조화하고 관리하는 방법을 알아보았습니다.

에이전트 체인과 워크플로를 효과적으로 구현함으로써 복잡한 문제를 해결하는 강력한 AI 애플리케이션을 구축할 수 있습니다. 다음 장에서는 이러한 에이전트와 워크플로를 평가, 디버깅, 모니터링하는 방법에 대해 알아보겠습니다.

## 연습 문제

1. 세 개의 에이전트(질문 분류, 정보 검색, 응답 생성)를 연결하는 선형 체인을 구현하고, 사용자 질문에 대한 답변을 생성하는 시스템을 구축해 보세요.
2. 사용자 입력을 분석하여 기술적 질문, 일반적 질문, 불만 사항 중 하나로 분류한 후 각각 다른 에이전트로 라우팅하는 분기 워크플로를 구현해 보세요.
3. 문서 요약, 감정 분석, 키워드 추출, 주제 분류를 병렬로 수행하는 문서 분석 워크플로를 구현해 보세요.
4. 사용자 피드백에 따라 반복적으로 콘텐츠를 개선하는 반복 워크플로를 설계하고 구현해 보세요.
5. 이벤트 기반 아키텍처를 사용하여 새로운 문서가 업로드되면 자동으로 처리되는 워크플로를 구현해 보세요.