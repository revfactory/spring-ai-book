# 19장: 에이전트 평가·디버깅·모니터링

## 19.1 에이전트 평가 개요

AI 에이전트 시스템을 개발하고 운영할 때 성능과 품질을 지속적으로 평가하는 것이 중요합니다. 이 장에서는 Spring AI 에이전트를 체계적으로 평가하고, 디버깅하며, 모니터링하는 방법을 살펴보겠습니다.

### 19.1.1 에이전트 평가의 중요성

- 품질 보증과 지속적 개선
- 사용자 만족도 향상
- 리소스 사용 최적화
- 윤리적, 법적 규정 준수

### 19.1.2 평가 프레임워크 구성요소

```java
public interface AgentEvaluationFramework {
    EvaluationResult evaluateAgent(Agent agent, EvaluationDataset dataset);
    List<EvaluationMetric> getSupportedMetrics();
    void registerMetric(EvaluationMetric metric);
    EvaluationReport generateReport(List<EvaluationResult> results);
}
```

### 19.1.3 평가 지표 유형

- 정확성 지표 (Accuracy Metrics)
- 일관성 지표 (Consistency Metrics)
- 안전성 지표 (Safety Metrics)
- 효율성 지표 (Efficiency Metrics)
- 사용자 만족도 지표 (User Satisfaction Metrics)

## 19.2 평가 데이터셋 구성

에이전트를 평가하기 위한 데이터셋을 구성하고 관리하는 방법을 알아봅니다.

### 19.2.1 평가 데이터셋 인터페이스

```java
public interface EvaluationDataset {
    List<EvaluationSample> getSamples();
    int size();
    String getName();
    DatasetMetadata getMetadata();
}

public interface EvaluationSample {
    String getId();
    AgentRequest getInput();
    List<ExpectedOutput> getExpectedOutputs();
    Map<String, Object> getMetadata();
}

public interface ExpectedOutput {
    String getContent();
    double getScore(String actualOutput);
    boolean isAcceptable(String actualOutput);
}
```

### 19.2.2 테스트 케이스 구성

```java
@Component
public class TestCaseBuilder {
    
    public EvaluationSample createTestCase(
            String id,
            String input,
            List<String> expectedOutputs,
            OutputMatchType matchType) {
        
        AgentRequest agentRequest = AgentRequest.builder()
            .prompt(input)
            .build();
        
        List<ExpectedOutput> expectedOutputList = expectedOutputs.stream()
            .map(output -> createExpectedOutput(output, matchType))
            .collect(Collectors.toList());
        
        return new DefaultEvaluationSample(id, agentRequest, expectedOutputList);
    }
    
    private ExpectedOutput createExpectedOutput(String output, OutputMatchType matchType) {
        switch (matchType) {
            case EXACT:
                return new ExactMatchOutput(output);
            case CONTAINS:
                return new ContainsMatchOutput(output);
            case SEMANTIC:
                return new SemanticMatchOutput(output, embeddingService);
            case REGEX:
                return new RegexMatchOutput(output);
            default:
                throw new IllegalArgumentException("Unsupported match type: " + matchType);
        }
    }
    
    public enum OutputMatchType {
        EXACT, CONTAINS, SEMANTIC, REGEX
    }
}
```

### 19.2.3 데이터셋 저장 및 관리

```java
@Service
public class EvaluationDatasetService {
    
    private final EvaluationDatasetRepository repository;
    
    public EvaluationDataset createDataset(String name, String description, List<EvaluationSample> samples) {
        // 데이터셋 생성 및 저장
        DatasetMetadata metadata = new DatasetMetadata(description, LocalDateTime.now());
        EvaluationDataset dataset = new DefaultEvaluationDataset(name, samples, metadata);
        return repository.save(dataset);
    }
    
    public EvaluationDataset getDataset(String name) {
        return repository.findByName(name)
            .orElseThrow(() -> new IllegalArgumentException("Dataset not found: " + name));
    }
    
    public List<EvaluationDataset> getAllDatasets() {
        return repository.findAll();
    }
    
    public EvaluationDataset addSamplesToDataset(String datasetName, List<EvaluationSample> newSamples) {
        // 기존 데이터셋에 샘플 추가
        EvaluationDataset dataset = getDataset(datasetName);
        List<EvaluationSample> allSamples = new ArrayList<>(dataset.getSamples());
        allSamples.addAll(newSamples);
        
        DatasetMetadata updatedMetadata = new DatasetMetadata(
            dataset.getMetadata().getDescription(),
            LocalDateTime.now()
        );
        
        EvaluationDataset updatedDataset = new DefaultEvaluationDataset(
            dataset.getName(),
            allSamples,
            updatedMetadata
        );
        
        return repository.save(updatedDataset);
    }
}
```

## 19.3 정확성 및 품질 평가

에이전트의 응답 정확성과 품질을 평가하는 방법을 알아봅니다.

### 19.3.1 정확성 평가 메트릭

```java
@Component
public class AccuracyMetric implements EvaluationMetric {
    
    @Override
    public String getName() {
        return "accuracy";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 기대 출력과 실제 출력 비교
        for (ExpectedOutput expectedOutput : sample.getExpectedOutputs()) {
            if (expectedOutput.isAcceptable(response.getContent())) {
                return 1.0;
            }
        }
        return 0.0;
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures if the agent response matches any of the expected outputs",
            "range", "0.0 to 1.0"
        );
    }
}
```

### 19.3.2 의미적 유사성 평가

```java
@Component
public class SemanticSimilarityMetric implements EvaluationMetric {
    
    private final TextEmbedding embeddingService;
    
    @Override
    public String getName() {
        return "semantic_similarity";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 기대 출력과 실제 출력의 의미적 유사도 계산
        String actualContent = response.getContent();
        
        double maxSimilarity = 0.0;
        for (ExpectedOutput expectedOutput : sample.getExpectedOutputs()) {
            String expectedContent = expectedOutput.getContent();
            
            // 임베딩 생성
            float[] actualEmbedding = embeddingService.embed(actualContent).getVector();
            float[] expectedEmbedding = embeddingService.embed(expectedContent).getVector();
            
            // 코사인 유사도 계산
            double similarity = calculateCosineSimilarity(actualEmbedding, expectedEmbedding);
            maxSimilarity = Math.max(maxSimilarity, similarity);
        }
        
        return maxSimilarity;
    }
    
    private double calculateCosineSimilarity(float[] vector1, float[] vector2) {
        // 코사인 유사도 계산 로직
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures semantic similarity between agent response and expected outputs",
            "range", "0.0 to 1.0"
        );
    }
}
```

### 19.3.3 도구 사용 정확성 평가

```java
@Component
public class ToolUsageMetric implements EvaluationMetric {
    
    @Override
    public String getName() {
        return "tool_usage";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 예상된 도구 사용과 실제 도구 사용 비교
        List<ToolExecution> actualToolExecutions = response.getToolExecutions();
        List<ExpectedToolUsage> expectedToolUsages = getExpectedToolUsages(sample);
        
        if (expectedToolUsages.isEmpty() && actualToolExecutions.isEmpty()) {
            return 1.0; // 도구를 사용하지 않아야 하는 경우
        }
        
        if (expectedToolUsages.isEmpty() || actualToolExecutions.isEmpty()) {
            return 0.0; // 한쪽만 도구를 사용한 경우
        }
        
        // 도구 사용 일치 확인
        return calculateToolUsageMatch(expectedToolUsages, actualToolExecutions);
    }
    
    private List<ExpectedToolUsage> getExpectedToolUsages(EvaluationSample sample) {
        // 샘플 메타데이터에서 예상 도구 사용 정보 추출
    }
    
    private double calculateToolUsageMatch(
            List<ExpectedToolUsage> expectedUsages,
            List<ToolExecution> actualExecutions) {
        // 도구 사용 일치도 계산 로직
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Evaluates if the agent correctly used the expected tools",
            "range", "0.0 to 1.0"
        );
    }
}
```

## 19.4 안전성 및 윤리적 준수 평가

에이전트의 안전성과 윤리적 준수를 평가하는 방법을 알아봅니다.

### 19.4.1 유해 콘텐츠 감지

```java
@Component
public class HarmfulContentMetric implements EvaluationMetric {
    
    private final ModerationService moderationService;
    
    @Override
    public String getName() {
        return "harmful_content";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 응답에서 유해 콘텐츠 감지
        String content = response.getContent();
        ModerationResult result = moderationService.moderate(content);
        
        // 안전성 점수 계산 (높을수록 안전)
        double safetyScore = 1.0 - result.getHarmScore();
        return safetyScore;
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures the absence of harmful content in agent responses",
            "range", "0.0 to 1.0 (higher is safer)"
        );
    }
}
```

### 19.4.2 편향성 감지

```java
@Component
public class BiasDetectionMetric implements EvaluationMetric {
    
    private final BiasAnalysisService biasAnalysisService;
    
    @Override
    public String getName() {
        return "bias_detection";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 응답에서 편향성 분석
        String content = response.getContent();
        BiasAnalysisResult result = biasAnalysisService.analyzeText(content);
        
        // 편향성 점수 계산 (높을수록 중립적)
        double fairnessScore = 1.0 - result.getBiasScore();
        return fairnessScore;
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures the neutrality and absence of bias in agent responses",
            "range", "0.0 to 1.0 (higher is more neutral)"
        );
    }
}
```

### 19.4.3 개인정보 누출 감지

```java
@Component
public class PrivacyProtectionMetric implements EvaluationMetric {
    
    private final PrivacyScanner privacyScanner;
    
    @Override
    public String getName() {
        return "privacy_protection";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 응답에서 개인정보 누출 검사
        String content = response.getContent();
        PrivacyScanResult result = privacyScanner.scanForPII(content);
        
        // 개인정보 보호 점수 계산 (높을수록 안전)
        int piiCount = result.getDetectedPIICount();
        return piiCount == 0 ? 1.0 : Math.max(0.0, 1.0 - (piiCount * 0.2));
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Evaluates if the agent properly protects personal information",
            "range", "0.0 to 1.0 (higher is better protected)"
        );
    }
}
```

## 19.5 효율성 및 성능 평가

에이전트의 효율성과 성능을 평가하는 방법을 알아봅니다.

### 19.5.1 응답 시간 측정

```java
@Component
public class ResponseTimeMetric implements EvaluationMetric {
    
    @Override
    public String getName() {
        return "response_time";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 응답 시간을 메타데이터에서 추출
        Map<String, Object> metadata = response.getMetadata();
        if (!metadata.containsKey("responseTimeMs")) {
            return 0.0; // 응답 시간 정보가 없음
        }
        
        long responseTimeMs = (long) metadata.get("responseTimeMs");
        
        // 응답 시간 점수 계산 (낮을수록 좋음)
        // 예: 1초 이내는 1.0, 10초 이상은 0.1
        if (responseTimeMs <= 1000) {
            return 1.0;
        } else if (responseTimeMs >= 10000) {
            return 0.1;
        } else {
            return 1.0 - ((responseTimeMs - 1000) / 9000.0 * 0.9);
        }
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures the response time of the agent",
            "range", "0.0 to 1.0 (higher is faster)"
        );
    }
}
```

### 19.5.2 토큰 사용량 측정

```java
@Component
public class TokenUsageMetric implements EvaluationMetric {
    
    @Override
    public String getName() {
        return "token_efficiency";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 토큰 사용량을 메타데이터에서 추출
        Map<String, Object> metadata = response.getMetadata();
        if (!metadata.containsKey("totalTokens")) {
            return 0.0; // 토큰 정보가 없음
        }
        
        int totalTokens = (int) metadata.get("totalTokens");
        int expectedMaxTokens = getExpectedMaxTokens(sample);
        
        // 토큰 효율성 점수 계산
        if (totalTokens <= expectedMaxTokens) {
            return 1.0;
        } else {
            double overageRatio = (double) totalTokens / expectedMaxTokens;
            return Math.max(0.0, 1.0 - ((overageRatio - 1.0) * 0.5));
        }
    }
    
    private int getExpectedMaxTokens(EvaluationSample sample) {
        // 샘플 메타데이터에서 예상 최대 토큰 수 추출
        Map<String, Object> metadata = sample.getMetadata();
        return metadata.containsKey("expectedMaxTokens")
            ? (int) metadata.get("expectedMaxTokens")
            : 1000; // 기본값
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures the token usage efficiency of the agent",
            "range", "0.0 to 1.0 (higher is more efficient)"
        );
    }
}
```

### 19.5.3 리소스 사용량 측정

```java
@Component
public class ResourceUsageMetric implements EvaluationMetric {
    
    @Override
    public String getName() {
        return "resource_usage";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 리소스 사용량을 메타데이터에서 추출
        Map<String, Object> metadata = response.getMetadata();
        
        double cpuScore = calculateCpuScore(metadata);
        double memoryScore = calculateMemoryScore(metadata);
        double apiCallScore = calculateApiCallScore(metadata);
        
        // 종합 리소스 점수 계산
        return (cpuScore + memoryScore + apiCallScore) / 3.0;
    }
    
    private double calculateCpuScore(Map<String, Object> metadata) {
        // CPU 사용량 점수 계산
        if (!metadata.containsKey("cpuTimeMs")) {
            return 1.0; // 정보가 없으면 최대 점수
        }
        
        long cpuTimeMs = (long) metadata.get("cpuTimeMs");
        return cpuTimeMs < 500 ? 1.0 : Math.max(0.0, 1.0 - ((cpuTimeMs - 500) / 9500.0));
    }
    
    private double calculateMemoryScore(Map<String, Object> metadata) {
        // 메모리 사용량 점수 계산
    }
    
    private double calculateApiCallScore(Map<String, Object> metadata) {
        // API 호출 횟수 점수 계산
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Evaluates the computational resource efficiency of the agent",
            "range", "0.0 to 1.0 (higher is more efficient)"
        );
    }
}
```

## 19.6 사용자 관점 평가

사용자 경험과 만족도 측면에서 에이전트를 평가하는 방법을 알아봅니다.

### 19.6.1 사용자 만족도 측정

```java
@Component
public class UserSatisfactionMetric implements EvaluationMetric {
    
    private final UserFeedbackRepository feedbackRepository;
    
    @Override
    public String getName() {
        return "user_satisfaction";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 테스트 ID에 대한 사용자 피드백 검색
        String testId = sample.getId();
        List<UserFeedback> feedbacks = feedbackRepository.findByTestId(testId);
        
        if (feedbacks.isEmpty()) {
            return 0.0; // 피드백이 없음
        }
        
        // 평균 사용자 만족도 계산
        double totalScore = 0.0;
        for (UserFeedback feedback : feedbacks) {
            totalScore += feedback.getRating();
        }
        
        return totalScore / (feedbacks.size() * 5.0); // 5점 만점 기준으로 정규화
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures user satisfaction based on feedback",
            "range", "0.0 to 1.0 (higher is more satisfied)"
        );
    }
}
```

### 19.6.2 상호작용 품질 평가

```java
@Component
public class InteractionQualityMetric implements EvaluationMetric {
    
    private final ConversationAnalyzer conversationAnalyzer;
    
    @Override
    public String getName() {
        return "interaction_quality";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 대화 이력 추출
        List<Message> conversationHistory = extractConversationHistory(sample, response);
        
        // 대화 품질 분석
        ConversationQuality quality = conversationAnalyzer.analyzeQuality(conversationHistory);
        
        // 품질 점수 계산
        double coherenceScore = quality.getCoherenceScore();
        double engagementScore = quality.getEngagementScore();
        double helpfulnessScore = quality.getHelpfulnessScore();
        
        return (coherenceScore + engagementScore + helpfulnessScore) / 3.0;
    }
    
    private List<Message> extractConversationHistory(EvaluationSample sample, AgentResponse response) {
        // 대화 이력 구성
        List<Message> history = new ArrayList<>();
        
        // 요청 메시지 추가
        history.add(new UserMessage(sample.getInput().getPrompt()));
        
        // 응답 메시지 추가
        history.add(new AssistantMessage(response.getContent()));
        
        return history;
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Evaluates the quality of agent-user interaction",
            "range", "0.0 to 1.0 (higher is better quality)"
        );
    }
}
```

### 19.6.3 태스크 완료율 평가

```java
@Component
public class TaskCompletionMetric implements EvaluationMetric {
    
    @Override
    public String getName() {
        return "task_completion";
    }
    
    @Override
    public double evaluate(EvaluationSample sample, AgentResponse response) {
        // 작업 요구사항 추출
        List<String> taskRequirements = extractTaskRequirements(sample);
        
        // 각 요구사항 충족 여부 확인
        int completedTasks = 0;
        for (String requirement : taskRequirements) {
            if (isRequirementSatisfied(requirement, response)) {
                completedTasks++;
            }
        }
        
        // 작업 완료율 계산
        return taskRequirements.isEmpty() ? 1.0 : (double) completedTasks / taskRequirements.size();
    }
    
    private List<String> extractTaskRequirements(EvaluationSample sample) {
        // 샘플 메타데이터에서 작업 요구사항 추출
        Map<String, Object> metadata = sample.getMetadata();
        if (!metadata.containsKey("taskRequirements")) {
            return Collections.emptyList();
        }
        
        return (List<String>) metadata.get("taskRequirements");
    }
    
    private boolean isRequirementSatisfied(String requirement, AgentResponse response) {
        // 특정 요구사항 충족 여부 확인 로직
    }
    
    @Override
    public Map<String, Object> getMetadata() {
        return Map.of(
            "description", "Measures the rate of task completion by the agent",
            "range", "0.0 to 1.0 (higher is more complete)"
        );
    }
}
```

## 19.7 에이전트 디버깅 도구 구현

에이전트 동작을 디버깅하고 문제를 진단하는 도구를 구현하는 방법을 알아봅니다.

### 19.7.1 세부 로깅 및 추적

```java
@Aspect
@Component
public class AgentExecutionTracer {
    
    private final Logger logger = LoggerFactory.getLogger(AgentExecutionTracer.class);
    private final TraceRepository traceRepository;
    
    @Around("execution(* org.springframework.ai.agent.Agent.execute(..))")
    public Object traceAgentExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        Agent agent = (Agent) joinPoint.getTarget();
        AgentRequest request = (AgentRequest) joinPoint.getArgs()[0];
        
        // 추적 ID 생성
        String traceId = UUID.randomUUID().toString();
        
        // 실행 시작 로깅
        logger.debug("Agent execution started - TraceID: {}, Agent: {}", traceId, agent.getClass().getSimpleName());
        
        // 요청 상세 정보 로깅
        logRequest(traceId, request);
        
        // 실행 시간 측정 시작
        long startTime = System.currentTimeMillis();
        
        ExecutionTrace trace = new ExecutionTrace();
        trace.setTraceId(traceId);
        trace.setAgentType(agent.getClass().getName());
        trace.setRequest(request);
        trace.setStartTime(LocalDateTime.now());
        
        try {
            // 에이전트 실행
            AgentResponse response = (AgentResponse) joinPoint.proceed();
            
            // 실행 완료 시간 및 소요 시간 계산
            long endTime = System.currentTimeMillis();
            long executionTime = endTime - startTime;
            
            // 응답 상세 정보 로깅
            logResponse(traceId, response, executionTime);
            
            // 추적 정보 저장
            trace.setResponse(response);
            trace.setEndTime(LocalDateTime.now());
            trace.setExecutionTimeMs(executionTime);
            trace.setStatus("SUCCESS");
            traceRepository.save(trace);
            
            return response;
            
        } catch (Throwable e) {
            // 오류 정보 로깅
            long endTime = System.currentTimeMillis();
            long executionTime = endTime - startTime;
            
            logger.error("Agent execution failed - TraceID: {}, Error: {}", traceId, e.getMessage(), e);
            
            // 추적 정보 저장 (오류 포함)
            trace.setEndTime(LocalDateTime.now());
            trace.setExecutionTimeMs(executionTime);
            trace.setStatus("ERROR");
            trace.setErrorMessage(e.getMessage());
            trace.setStackTrace(ExceptionUtils.getStackTrace(e));
            traceRepository.save(trace);
            
            throw e;
        }
    }
    
    private void logRequest(String traceId, AgentRequest request) {
        // 요청 상세 정보 로깅
        logger.debug("Request - TraceID: {}, Prompt: {}", traceId, request.getPrompt());
        logger.trace("Request details - TraceID: {}, Full request: {}", traceId, request);
    }
    
    private void logResponse(String traceId, AgentResponse response, long executionTime) {
        // 응답 상세 정보 로깅
        logger.debug("Response - TraceID: {}, Execution time: {}ms, Content length: {}", 
            traceId, executionTime, response.getContent().length());
        logger.trace("Response details - TraceID: {}, Full response: {}", traceId, response);
        
        // 도구 실행 정보 로깅
        List<ToolExecution> toolExecutions = response.getToolExecutions();
        if (!toolExecutions.isEmpty()) {
            logger.debug("Tool executions - TraceID: {}, Count: {}", traceId, toolExecutions.size());
            for (ToolExecution execution : toolExecutions) {
                logger.trace("Tool execution - TraceID: {}, Tool: {}, Parameters: {}", 
                    traceId, execution.getToolName(), execution.getParameters());
            }
        }
    }
}
```

### 19.7.2 단계별 실행 분석

```java
@Service
public class AgentExecutionAnalyzer {
    
    private final TraceRepository traceRepository;
    
    public ExecutionAnalysisReport analyzeExecution(String traceId) {
        // 추적 정보 조회
        ExecutionTrace trace = traceRepository.findByTraceId(traceId)
            .orElseThrow(() -> new IllegalArgumentException("Trace not found: " + traceId));
        
        // 실행 단계 분석
        List<ExecutionStep> steps = analyzeExecutionSteps(trace);
        
        // 병목 구간 식별
        List<PerformanceBottleneck> bottlenecks = identifyBottlenecks(steps);
        
        // 에러 및 경고 분석
        List<ExecutionIssue> issues = analyzeIssues(trace, steps);
        
        // 분석 보고서 생성
        return new ExecutionAnalysisReport(trace, steps, bottlenecks, issues);
    }
    
    private List<ExecutionStep> analyzeExecutionSteps(ExecutionTrace trace) {
        // 실행 단계 분석 로직
        List<ExecutionStep> steps = new ArrayList<>();
        
        // 1. 입력 처리 단계
        steps.add(new ExecutionStep("input_processing", "Input processing", 
            trace.getStartTime(), null, extractInputProcessingDuration(trace)));
        
        // 2. 모델 호출 단계
        LocalDateTime modelCallStart = calculateModelCallStart(trace);
        steps.add(new ExecutionStep("model_call", "Model API call",
            modelCallStart, null, extractModelCallDuration(trace)));
        
        // 3. 도구 실행 단계
        if (trace.getResponse() != null && !trace.getResponse().getToolExecutions().isEmpty()) {
            List<ToolExecution> toolExecutions = trace.getResponse().getToolExecutions();
            for (int i = 0; i < toolExecutions.size(); i++) {
                ToolExecution execution = toolExecutions.get(i);
                LocalDateTime toolStart = calculateToolExecutionStart(trace, i);
                steps.add(new ExecutionStep("tool_execution_" + i, "Tool: " + execution.getToolName(),
                    toolStart, null, extractToolExecutionDuration(trace, i)));
            }
        }
        
        // 4. 응답 생성 단계
        LocalDateTime responseGenStart = calculateResponseGenerationStart(trace);
        steps.add(new ExecutionStep("response_generation", "Response generation",
            responseGenStart, trace.getEndTime(), extractResponseGenerationDuration(trace)));
        
        return steps;
    }
    
    private List<PerformanceBottleneck> identifyBottlenecks(List<ExecutionStep> steps) {
        // 병목 구간 식별 로직
    }
    
    private List<ExecutionIssue> analyzeIssues(ExecutionTrace trace, List<ExecutionStep> steps) {
        // 실행 중 발생한 이슈 분석 로직
    }
    
    // 단계별 시간 계산 메서드들...
}
```

### 19.7.3 프롬프트와 응답 디버깅

```java
@Component
public class PromptDebugger {
    
    private final ModelClient debugModel;
    
    public PromptAnalysisResult analyzePrompt(String prompt, Map<String, Object> parameters) {
        // 프롬프트 구조 분석
        PromptStructure structure = analyzePromptStructure(prompt);
        
        // 변수 치환 시뮬레이션
        String renderedPrompt = simulateVariableSubstitution(prompt, parameters);
        
        // 토큰 수 예측
        int estimatedTokens = estimateTokenCount(renderedPrompt);
        
        // 프롬프트 품질 평가
        PromptQualityMetrics qualityMetrics = evaluatePromptQuality(renderedPrompt);
        
        // 잠재적 이슈 감지
        List<PromptIssue> issues = detectPotentialIssues(renderedPrompt, structure, qualityMetrics);
        
        return new PromptAnalysisResult(
            structure,
            renderedPrompt,
            estimatedTokens,
            qualityMetrics,
            issues
        );
    }
    
    private PromptStructure analyzePromptStructure(String prompt) {
        // 프롬프트 구조 분석 로직
    }
    
    private String simulateVariableSubstitution(String prompt, Map<String, Object> parameters) {
        // 변수 치환 시뮬레이션 로직
    }
    
    private int estimateTokenCount(String text) {
        // 토큰 수 예측 로직
    }
    
    private PromptQualityMetrics evaluatePromptQuality(String prompt) {
        // 프롬프트 품질 평가 로직
    }
    
    private List<PromptIssue> detectPotentialIssues(
            String prompt,
            PromptStructure structure,
            PromptQualityMetrics metrics) {
        // 잠재적 이슈 감지 로직
    }
}
```

## 19.8 에이전트 모니터링 시스템 구현

운영 환경에서 에이전트 성능과 품질을 모니터링하는 시스템을 구현하는 방법을 알아봅니다.

### 19.8.1 실시간 메트릭 수집

```java
@Component
public class AgentMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    
    @Around("execution(* org.springframework.ai.agent.Agent.execute(..))")
    public Object collectMetrics(ProceedingJoinPoint joinPoint) throws Throwable {
        Agent agent = (Agent) joinPoint.getTarget();
        String agentName = agent.getClass().getSimpleName();
        
        // 호출 횟수 카운터 증가
        meterRegistry.counter("agent.calls", "agent", agentName).increment();
        
        // 타이머 시작
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            // 에이전트 실행
            AgentResponse response = (AgentResponse) joinPoint.proceed();
            
            // 성공 횟수 카운터 증가
            meterRegistry.counter("agent.success", "agent", agentName).increment();
            
            // 토큰 사용량 기록
            recordTokenUsage(agentName, response);
            
            // 도구 사용 기록
            recordToolUsage(agentName, response);
            
            return response;
            
        } catch (Exception e) {
            // 실패 횟수 카운터 증가
            meterRegistry.counter("agent.errors", "agent", agentName, "error", e.getClass().getSimpleName()).increment();
            throw e;
            
        } finally {
            // 응답 시간 기록
            sample.stop(meterRegistry.timer("agent.response.time", "agent", agentName));
        }
    }
    
    private void recordTokenUsage(String agentName, AgentResponse response) {
        // 토큰 사용량 메트릭 기록
        Map<String, Object> metadata = response.getMetadata();
        if (metadata.containsKey("totalTokens")) {
            int totalTokens = (int) metadata.get("totalTokens");
            meterRegistry.gauge("agent.tokens.total", Tags.of("agent", agentName), totalTokens);
        }
        
        if (metadata.containsKey("promptTokens")) {
            int promptTokens = (int) metadata.get("promptTokens");
            meterRegistry.gauge("agent.tokens.prompt", Tags.of("agent", agentName), promptTokens);
        }
        
        if (metadata.containsKey("completionTokens")) {
            int completionTokens = (int) metadata.get("completionTokens");
            meterRegistry.gauge("agent.tokens.completion", Tags.of("agent", agentName), completionTokens);
        }
    }
    
    private void recordToolUsage(String agentName, AgentResponse response) {
        // 도구 사용 메트릭 기록
        List<ToolExecution> toolExecutions = response.getToolExecutions();
        for (ToolExecution execution : toolExecutions) {
            String toolName = execution.getToolName();
            meterRegistry.counter("agent.tool.usage", 
                "agent", agentName, 
                "tool", toolName
            ).increment();
        }
    }
}
```

### 19.8.2 알림 시스템 구현

```java
@Service
public class AgentAlertingService {
    
    private final MeterRegistry meterRegistry;
    private final NotificationService notificationService;
    private final List<AlertRule> alertRules;
    
    @Scheduled(fixedRate = 60000) // 1분마다 실행
    public void checkAlerts() {
        for (AlertRule rule : alertRules) {
            // 규칙에 따른 메트릭 검사
            boolean isTriggered = checkMetricAgainstRule(rule);
            
            if (isTriggered) {
                // 알림 생성 및 전송
                Alert alert = createAlert(rule);
                notificationService.sendAlert(alert);
            }
        }
    }
    
    private boolean checkMetricAgainstRule(AlertRule rule) {
        // 규칙에 따른 메트릭 평가
        String metricName = rule.getMetricName();
        String agentName = rule.getAgentName();
        
        double currentValue = fetchMetricValue(metricName, agentName);
        
        switch (rule.getOperator()) {
            case GREATER_THAN:
                return currentValue > rule.getThreshold();
            case LESS_THAN:
                return currentValue < rule.getThreshold();
            case EQUALS:
                return Math.abs(currentValue - rule.getThreshold()) < 0.0001;
            default:
                return false;
        }
    }
    
    private double fetchMetricValue(String metricName, String agentName) {
        // 메트릭 값 조회
        Search search = Search.in(meterRegistry).name(metricName);
        
        if (agentName != null) {
            search = search.tag("agent", agentName);
        }
        
        return search.meter()
            .map(meter -> {
                if (meter instanceof Counter) {
                    return ((Counter) meter).count();
                } else if (meter instanceof Gauge) {
                    return ((Gauge) meter).value();
                } else if (meter instanceof Timer) {
                    return ((Timer) meter).mean(TimeUnit.MILLISECONDS);
                } else {
                    return 0.0;
                }
            })
            .orElse(0.0);
    }
    
    private Alert createAlert(AlertRule rule) {
        // 알림 객체 생성
        String metricName = rule.getMetricName();
        String agentName = rule.getAgentName();
        double currentValue = fetchMetricValue(metricName, agentName);
        
        return Alert.builder()
            .type("METRIC_ALERT")
            .severity(rule.getSeverity())
            .message(String.format(
                "Alert for %s on agent %s: value %.2f %s threshold %.2f",
                metricName,
                agentName != null ? agentName : "all",
                currentValue,
                rule.getOperator().getSymbol(),
                rule.getThreshold()
            ))
            .details(Map.of(
                "metricName", metricName,
                "agentName", agentName != null ? agentName : "all",
                "currentValue", currentValue,
                "threshold", rule.getThreshold(),
                "operator", rule.getOperator().toString()
            ))
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

### 19.8.3 대시보드 및 시각화

```java
@RestController
@RequestMapping("/api/monitoring")
public class AgentMonitoringController {
    
    private final AgentMetricsService metricsService;
    private final AgentEvaluationService evaluationService;
    private final AlertHistoryService alertService;
    
    @GetMapping("/metrics/summary")
    public ResponseEntity<MetricsSummary> getMetricsSummary(
            @RequestParam(required = false) String agentName,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end) {
        
        // 메트릭 요약 조회
        MetricsSummary summary = metricsService.getMetricsSummary(agentName, start, end);
        return ResponseEntity.ok(summary);
    }
    
    @GetMapping("/metrics/timeseries")
    public ResponseEntity<MetricsTimeSeries> getMetricsTimeSeries(
            @RequestParam String metricName,
            @RequestParam(required = false) String agentName,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end,
            @RequestParam(defaultValue = "PT1M") Duration interval) {
        
        // 시계열 메트릭 조회
        MetricsTimeSeries timeSeries = metricsService.getMetricsTimeSeries(
            metricName, agentName, start, end, interval);
        return ResponseEntity.ok(timeSeries);
    }
    
    @GetMapping("/evaluation/scores")
    public ResponseEntity<List<EvaluationScore>> getEvaluationScores(
            @RequestParam(required = false) String agentName,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end) {
        
        // 평가 점수 조회
        List<EvaluationScore> scores = evaluationService.getEvaluationScores(agentName, start, end);
        return ResponseEntity.ok(scores);
    }
    
    @GetMapping("/alerts")
    public ResponseEntity<List<Alert>> getAlerts(
            @RequestParam(required = false) String agentName,
            @RequestParam(required = false) AlertSeverity severity,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam(required = false) 
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end) {
        
        // 알림 이력 조회
        List<Alert> alerts = alertService.getAlerts(agentName, severity, start, end);
        return ResponseEntity.ok(alerts);
    }
}
```

## 19.9 에이전트 개선 및 최적화 프로세스

에이전트 평가 결과를 바탕으로 지속적인 개선과 최적화를 수행하는 방법을 알아봅니다.

### 19.9.1 A/B 테스팅 프레임워크

```java
@Service
public class AgentABTestingService {
    
    private final Random random = new Random();
    private final Map<String, ABTest> activeTests = new ConcurrentHashMap<>();
    private final ABTestResultRepository resultRepository;
    private final EvaluationService evaluationService;
    
    public String createABTest(ABTestConfiguration config) {
        // A/B 테스트 생성
        String testId = UUID.randomUUID().toString();
        
        ABTest test = new ABTest();
        test.setId(testId);
        test.setName(config.getName());
        test.setDescription(config.getDescription());
        test.setControlAgent(config.getControlAgent());
        test.setVariantAgent(config.getVariantAgent());
        test.setEvaluationDataset(config.getEvaluationDataset());
        test.setMetrics(config.getMetrics());
        test.setTrafficSplit(config.getTrafficSplit());
        test.setStartTime(LocalDateTime.now());
        test.setStatus("RUNNING");
        
        activeTests.put(testId, test);
        
        return testId;
    }
    
    public Agent selectAgentForRequest(String testId, AgentRequest request) {
        // 요청에 대한 에이전트 선택 (A 또는 B)
        ABTest test = activeTests.get(testId);
        if (test == null) {
            throw new IllegalArgumentException("A/B test not found: " + testId);
        }
        
        // 트래픽 분할에 따른 에이전트 선택
        double randomValue = random.nextDouble();
        boolean useVariant = randomValue < test.getTrafficSplit();
        
        Agent selectedAgent = useVariant ? test.getVariantAgent() : test.getControlAgent();
        
        // 선택 이력 기록
        recordAgentSelection(testId, request, selectedAgent, useVariant);
        
        return selectedAgent;
    }
    
    public void recordTestResult(String testId, AgentRequest request, AgentResponse response, boolean isVariant) {
        // 테스트 결과 기록
        ABTestResult result = new ABTestResult();
        result.setTestId(testId);
        result.setRequestId(UUID.randomUUID().toString());
        result.setRequest(request);
        result.setResponse(response);
        result.setVariant(isVariant);
        result.setTimestamp(LocalDateTime.now());
        
        // 평가 지표에 따른 점수 계산
        ABTest test = activeTests.get(testId);
        Map<String, Double> metricScores = new HashMap<>();
        
        for (String metricName : test.getMetrics()) {
            // 각 지표별 점수 계산
            EvaluationMetric metric = evaluationService.getMetric(metricName);
            EvaluationSample sample = findMatchingSample(test.getEvaluationDataset(), request);
            
            if (sample != null && metric != null) {
                double score = metric.evaluate(sample, response);
                metricScores.put(metricName, score);
            }
        }
        
        result.setMetricScores(metricScores);
        resultRepository.save(result);
    }
    
    public ABTestReport generateTestReport(String testId) {
        // 테스트 보고서 생성
        ABTest test = activeTests.get(testId);
        if (test == null) {
            throw new IllegalArgumentException("A/B test not found: " + testId);
        }
        
        List<ABTestResult> results = resultRepository.findByTestId(testId);
        
        // 컨트롤과 변형 그룹 분리
        List<ABTestResult> controlResults = results.stream()
            .filter(r -> !r.isVariant())
            .collect(Collectors.toList());
        
        List<ABTestResult> variantResults = results.stream()
            .filter(ABTestResult::isVariant)
            .collect(Collectors.toList());
        
        // 각 지표별 평균 점수 계산
        Map<String, ABTestMetricComparison> metricComparisons = new HashMap<>();
        
        for (String metricName : test.getMetrics()) {
            double controlAverage = calculateAverageScore(controlResults, metricName);
            double variantAverage = calculateAverageScore(variantResults, metricName);
            double improvement = variantAverage - controlAverage;
            double relativeChange = controlAverage != 0 ? improvement / controlAverage * 100 : 0;
            
            metricComparisons.put(metricName, new ABTestMetricComparison(
                metricName, controlAverage, variantAverage, improvement, relativeChange));
        }
        
        return new ABTestReport(test, controlResults.size(), variantResults.size(), metricComparisons);
    }
    
    private void recordAgentSelection(String testId, AgentRequest request, Agent selectedAgent, boolean isVariant) {
        // 에이전트 선택 이력 기록
    }
    
    private EvaluationSample findMatchingSample(EvaluationDataset dataset, AgentRequest request) {
        // 요청과 일치하는 평가 샘플 찾기
    }
    
    private double calculateAverageScore(List<ABTestResult> results, String metricName) {
        // 평균 점수 계산
        return results.stream()
            .map(r -> r.getMetricScores().getOrDefault(metricName, 0.0))
            .mapToDouble(Double::doubleValue)
            .average()
            .orElse(0.0);
    }
}
```

### 19.9.2 성능 최적화 도구

```java
@Service
public class AgentOptimizationService {
    
    private final ModelSelectionService modelSelectionService;
    private final PromptOptimizationService promptOptimizationService;
    private final TokenUsageAnalyzer tokenUsageAnalyzer;
    private final ParameterTuningService parameterTuningService;
    
    public OptimizationReport analyzeAndOptimize(Agent agent, EvaluationDataset dataset) {
        OptimizationReport report = new OptimizationReport();
        report.setAgentType(agent.getClass().getName());
        report.setTimestamp(LocalDateTime.now());
        
        // 1. 토큰 사용량 분석
        TokenUsageAnalysis tokenAnalysis = tokenUsageAnalyzer.analyzeTokenUsage(agent, dataset);
        report.setTokenUsageAnalysis(tokenAnalysis);
        
        // 2. 프롬프트 최적화 제안
        PromptOptimizationSuggestions promptSuggestions = 
            promptOptimizationService.generateSuggestions(agent, dataset);
        report.setPromptSuggestions(promptSuggestions);
        
        // 3. 모델 선택 분석
        ModelSelectionAnalysis modelAnalysis = modelSelectionService.analyzeModelFit(agent, dataset);
        report.setModelAnalysis(modelAnalysis);
        
        // 4. 파라미터 튜닝 제안
        ParameterTuningSuggestions parameterSuggestions = 
            parameterTuningService.suggestParameterAdjustments(agent, dataset);
        report.setParameterSuggestions(parameterSuggestions);
        
        // 5. 비용-성능 분석
        CostPerformanceAnalysis costAnalysis = analyzeCostPerformance(agent, dataset);
        report.setCostAnalysis(costAnalysis);
        
        return report;
    }
    
    private CostPerformanceAnalysis analyzeCostPerformance(Agent agent, EvaluationDataset dataset) {
        // 비용-성능 분석 로직
        CostPerformanceAnalysis analysis = new CostPerformanceAnalysis();
        
        // 샘플당 평균 비용 계산
        double totalCost = 0.0;
        for (EvaluationSample sample : dataset.getSamples()) {
            AgentResponse response = agent.execute(sample.getInput());
            double cost = calculateResponseCost(response);
            totalCost += cost;
        }
        
        double averageCost = totalCost / dataset.size();
        analysis.setAverageCostPerQuery(averageCost);
        
        // 성능당 비용 계산
        double performanceScore = evaluatePerformance(agent, dataset);
        analysis.setPerformanceScore(performanceScore);
        analysis.setCostPerformanceRatio(averageCost / performanceScore);
        
        // 최적화 제안 생성
        List<OptimizationSuggestion> suggestions = generateCostOptimizationSuggestions(analysis);
        analysis.setOptimizationSuggestions(suggestions);
        
        return analysis;
    }
    
    private double calculateResponseCost(AgentResponse response) {
        // 응답 비용 계산 로직
    }
    
    private double evaluatePerformance(Agent agent, EvaluationDataset dataset) {
        // 에이전트 성능 평가 로직
    }
    
    private List<OptimizationSuggestion> generateCostOptimizationSuggestions(CostPerformanceAnalysis analysis) {
        // 비용 최적화 제안 생성
    }
}
```

### 19.9.3 지속적 개선 파이프라인

```java
@Service
public class ContinuousImprovementPipeline {
    
    private final AgentRepository agentRepository;
    private final EvaluationDatasetRepository datasetRepository;
    private final EvaluationService evaluationService;
    private final AgentOptimizationService optimizationService;
    private final AgentVersioningService versioningService;
    
    @Scheduled(cron = "0 0 0 * * ?") // 매일 자정에 실행
    public void runImprovementPipeline() {
        // 개선 대상 에이전트 목록 조회
        List<Agent> agents = agentRepository.findByImprovementEnabled(true);
        
        for (Agent agent : agents) {
            try {
                // 에이전트별 개선 프로세스 실행
                improveAgent(agent);
            } catch (Exception e) {
                // 오류 처리
                log.error("Error improving agent: {}", agent.getClass().getName(), e);
            }
        }
    }
    
    private void improveAgent(Agent agent) {
        // 1. 평가 데이터셋 로드
        EvaluationDataset dataset = loadEvaluationDataset(agent);
        
        // 2. 현재 성능 평가
        EvaluationResult currentPerformance = evaluationService.evaluateAgent(agent, dataset);
        
        // 3. 성능 최적화 분석
        OptimizationReport optimizationReport = optimizationService.analyzeAndOptimize(agent, dataset);
        
        // 4. 개선 항목 필터링 (임계값 이상의 개선 효과가 예상되는 항목)
        List<ImprovementAction> improvementActions = filterImprovementActions(
            optimizationReport, currentPerformance);
        
        if (improvementActions.isEmpty()) {
            log.info("No significant improvements identified for agent: {}", agent.getClass().getName());
            return;
        }
        
        // 5. 개선 항목 적용
        Agent improvedAgent = applyImprovementActions(agent, improvementActions);
        
        // 6. 개선된 에이전트 평가
        EvaluationResult improvedPerformance = evaluationService.evaluateAgent(improvedAgent, dataset);
        
        // 7. 개선 효과 검증
        boolean isImproved = validateImprovement(currentPerformance, improvedPerformance);
        
        if (isImproved) {
            // 8. 개선된 버전 저장
            String newVersion = versioningService.createNewVersion(
                agent, improvedAgent, improvementActions, improvedPerformance);
            
            log.info("Created improved version {} for agent: {}", 
                newVersion, agent.getClass().getName());
        } else {
            log.info("Improvements did not yield significant results for agent: {}", 
                agent.getClass().getName());
        }
    }
    
    private EvaluationDataset loadEvaluationDataset(Agent agent) {
        // 에이전트에 적합한 평가 데이터셋 로드
    }
    
    private List<ImprovementAction> filterImprovementActions(
            OptimizationReport report, 
            EvaluationResult currentPerformance) {
        // 개선 효과가 기대되는 항목 필터링
    }
    
    private Agent applyImprovementActions(Agent agent, List<ImprovementAction> actions) {
        // 개선 항목 적용
    }
    
    private boolean validateImprovement(
            EvaluationResult currentPerformance, 
            EvaluationResult improvedPerformance) {
        // 개선 효과 검증
    }
}
```

## 19.10 실전 에이전트 평가 사례

실제 애플리케이션에서 에이전트를 평가하고 모니터링하는 구체적인 사례를 살펴봅니다.

### 19.10.1 고객 지원 에이전트 평가

```java
@Service
public class CustomerSupportAgentEvaluator {
    
    private final CustomerSupportAgent agent;
    private final EvaluationDatasetService datasetService;
    private final EvaluationService evaluationService;
    private final UserFeedbackRepository feedbackRepository;
    
    public CustomerSupportEvaluationReport evaluateAgent() {
        // 1. 기술적 평가
        EvaluationDataset technicalDataset = datasetService.getDataset("customer_support_technical");
        EvaluationResult technicalResult = evaluationService.evaluateAgent(agent, technicalDataset);
        
        // 2. 사용자 만족도 평가
        UserSatisfactionMetrics satisfactionMetrics = evaluateUserSatisfaction();
        
        // 3. 비즈니스 지표 평가
        BusinessMetrics businessMetrics = evaluateBusinessMetrics();
        
        // 4. 결합 보고서 생성
        return new CustomerSupportEvaluationReport(
            technicalResult, satisfactionMetrics, businessMetrics);
    }
    
    private UserSatisfactionMetrics evaluateUserSatisfaction() {
        // 사용자 만족도 평가 로직
        LocalDateTime oneMonthAgo = LocalDateTime.now().minusMonths(1);
        List<UserFeedback> recentFeedback = feedbackRepository.findAfter(oneMonthAgo);
        
        // 평균 만족도 점수
        double averageRating = recentFeedback.stream()
            .mapToDouble(UserFeedback::getRating)
            .average()
            .orElse(0.0);
        
        // 긍정적 피드백 비율
        long positiveCount = recentFeedback.stream()
            .filter(feedback -> feedback.getRating() >= 4.0)
            .count();
        
        double positiveRatio = (double) positiveCount / recentFeedback.size();
        
        // 부정적 피드백 분석
        List<UserFeedback> negativeFeedback = recentFeedback.stream()
            .filter(feedback -> feedback.getRating() < 3.0)
            .collect(Collectors.toList());
        
        Map<String, Integer> issueCategories = analyzeNegativeFeedback(negativeFeedback);
        
        return new UserSatisfactionMetrics(
            averageRating, positiveRatio, issueCategories);
    }
    
    private Map<String, Integer> analyzeNegativeFeedback(List<UserFeedback> negativeFeedback) {
        // 부정적 피드백 카테고리화 및 분석
    }
    
    private BusinessMetrics evaluateBusinessMetrics() {
        // 비즈니스 지표 평가 로직
        
        // 티켓 처리 속도
        double averageResolutionTime = calculateAverageResolutionTime();
        
        // 에스컬레이션 비율
        double escalationRate = calculateEscalationRate();
        
        // 첫 번째 응답에서 해결된 비율
        double firstContactResolutionRate = calculateFirstContactResolutionRate();
        
        // 비용 절감 추정치
        double costSavingsEstimate = estimateCostSavings();
        
        return new BusinessMetrics(
            averageResolutionTime,
            escalationRate,
            firstContactResolutionRate,
            costSavingsEstimate
        );
    }
    
    // 비즈니스 지표 계산 메서드들...
}
```

### 19.10.2 RAG 기반 지식 검색 에이전트 평가

```java
@Service
public class KnowledgeSearchAgentEvaluator {
    
    private final KnowledgeSearchAgent agent;
    private final DocumentRepository documentRepository;
    private final EvaluationService evaluationService;
    private final TextEmbedding embeddingService;
    
    public KnowledgeSearchEvaluationReport evaluateAgent() {
        // 1. 정보 검색 정확성 평가
        RetrievalAccuracyResult retrievalAccuracy = evaluateRetrievalAccuracy();
        
        // 2. 응답 품질 평가
        ResponseQualityResult responseQuality = evaluateResponseQuality();
        
        // 3. 지식 커버리지 평가
        KnowledgeCoverageResult knowledgeCoverage = evaluateKnowledgeCoverage();
        
        return new KnowledgeSearchEvaluationReport(
            retrievalAccuracy, responseQuality, knowledgeCoverage);
    }
    
    private RetrievalAccuracyResult evaluateRetrievalAccuracy() {
        // 정보 검색 정확성 평가
        EvaluationDataset retrievalDataset = createRetrievalEvaluationDataset();
        
        List<RetrievalEvaluation> evaluations = new ArrayList<>();
        
        for (EvaluationSample sample : retrievalDataset.getSamples()) {
            // 에이전트 실행
            AgentRequest request = sample.getInput();
            AgentResponse response = agent.execute(request);
            
            // 검색된 문서 추출
            List<Document> retrievedDocuments = extractRetrievedDocuments(response);
            
            // 관련성 점수 계산
            List<String> expectedDocumentIds = getExpectedDocumentIds(sample);
            
            double precision = calculatePrecision(retrievedDocuments, expectedDocumentIds);
            double recall = calculateRecall(retrievedDocuments, expectedDocumentIds);
            double f1Score = calculateF1Score(precision, recall);
            
            evaluations.add(new RetrievalEvaluation(
                sample.getId(), precision, recall, f1Score, retrievedDocuments.size()));
        }
        
        return new RetrievalAccuracyResult(evaluations);
    }
    
    private ResponseQualityResult evaluateResponseQuality() {
        // 응답 품질 평가
        EvaluationDataset qualityDataset = createQualityEvaluationDataset();
        EvaluationResult result = evaluationService.evaluateAgent(agent, qualityDataset);
        
        List<String> evaluatedMetrics = Arrays.asList(
            "accuracy", "completeness", "relevance", "conciseness", "coherence");
        
        Map<String, Double> metricScores = new HashMap<>();
        for (String metric : evaluatedMetrics) {
            metricScores.put(metric, result.getAverageScore(metric));
        }
        
        return new ResponseQualityResult(metricScores);
    }
    
    private KnowledgeCoverageResult evaluateKnowledgeCoverage() {
        // 지식 커버리지 평가
        
        // 1. 전체 문서 코퍼스 로드
        List<Document> allDocuments = documentRepository.findAll();
        
        // 2. 모든 문서의 내용 추출 및 임베딩
        List<float[]> documentEmbeddings = new ArrayList<>();
        for (Document doc : allDocuments) {
            float[] embedding = embeddingService.embed(doc.getContent()).getVector();
            documentEmbeddings.add(embedding);
        }
        
        // 3. 임베딩 공간 분석
        KnowledgeCoverageAnalysis coverageAnalysis = analyzeEmbeddingCoverage(documentEmbeddings);
        
        // 4. 랜덤 쿼리 생성 및 유사 문서 검색
        List<CoverageTestResult> testResults = performCoverageTests(allDocuments);
        
        return new KnowledgeCoverageResult(coverageAnalysis, testResults);
    }
    
    // 정보 검색 정확성 평가 관련 메서드들...
    
    // 응답 품질 평가 관련 메서드들...
    
    // 지식 커버리지 평가 관련 메서드들...
}
```

### 19.10.3 다중 에이전트 시스템 평가

```java
@Service
public class MultiAgentSystemEvaluator {
    
    private final List<Agent> agents;
    private final AgentCoordinator coordinator;
    private final EvaluationService evaluationService;
    private final SystemPerformanceMonitor performanceMonitor;
    
    public MultiAgentEvaluationReport evaluateSystem() {
        // 1. 개별 에이전트 평가
        Map<String, EvaluationResult> agentResults = new HashMap<>();
        for (Agent agent : agents) {
            String agentName = agent.getClass().getSimpleName();
            EvaluationDataset dataset = getDatasetForAgent(agent);
            EvaluationResult result = evaluationService.evaluateAgent(agent, dataset);
            agentResults.put(agentName, result);
        }
        
        // 2. 통합 시스템 평가
        SystemEvaluationResult systemResult = evaluateIntegratedSystem();
        
        // 3. 에이전트 간 상호작용 평가
        InteractionEvaluationResult interactionResult = evaluateAgentInteractions();
        
        // 4. 시스템 성능 평가
        PerformanceEvaluationResult performanceResult = evaluateSystemPerformance();
        
        return new MultiAgentEvaluationReport(
            agentResults, systemResult, interactionResult, performanceResult);
    }
    
    private EvaluationDataset getDatasetForAgent(Agent agent) {
        // 에이전트별 평가 데이터셋 결정
    }
    
    private SystemEvaluationResult evaluateIntegratedSystem() {
        // 통합 시스템 평가 로직
        EvaluationDataset systemDataset = createSystemEvaluationDataset();
        
        List<SystemTestResult> testResults = new ArrayList<>();
        
        for (EvaluationSample sample : systemDataset.getSamples()) {
            // 시스템 실행
            AgentRequest request = sample.getInput();
            AgentResponse response = coordinator.processRequest(request);
            
            // 결과 평가
            boolean taskCompleted = isTaskCompleted(response, sample);
            int steps = countExecutionSteps(response);
            long executionTime = extractExecutionTime(response);
            
            testResults.add(new SystemTestResult(
                sample.getId(), 
                taskCompleted, 
                steps, 
                executionTime
            ));
        }
        
        // 종합 결과 계산
        double taskCompletionRate = calculateTaskCompletionRate(testResults);
        double averageSteps = calculateAverageSteps(testResults);
        double averageExecutionTime = calculateAverageExecutionTime(testResults);
        
        return new SystemEvaluationResult(
            taskCompletionRate,
            averageSteps,
            averageExecutionTime,
            testResults
        );
    }
    
    private InteractionEvaluationResult evaluateAgentInteractions() {
        // 에이전트 간 상호작용 평가 로직
        List<InteractionTestResult> results = new ArrayList<>();
        
        // 에이전트 쌍 간 메시지 전달 정확성 테스트
        for (int i = 0; i < agents.size(); i++) {
            for (int j = 0; j < agents.size(); j++) {
                if (i != j) {
                    Agent sender = agents.get(i);
                    Agent receiver = agents.get(j);
                    
                    InteractionTestResult result = testAgentInteraction(sender, receiver);
                    results.add(result);
                }
            }
        }
        
        // 대화 연속성 테스트
        ConversationContinuityResult continuityResult = testConversationContinuity();
        
        // 작업 위임 테스트
        TaskDelegationResult delegationResult = testTaskDelegation();
        
        return new InteractionEvaluationResult(
            results, continuityResult, delegationResult);
    }
    
    private PerformanceEvaluationResult evaluateSystemPerformance() {
        // 시스템 성능 평가 로직
        return performanceMonitor.collectPerformanceMetrics();
    }
    
    // 기타 평가 관련 메서드들...
}
```

## 19.11 결론

이 장에서는 Spring AI 에이전트를 효과적으로 평가하고, 디버깅하며, 모니터링하는 다양한 방법을 살펴보았습니다. 평가 프레임워크, 디버깅 도구, 모니터링 시스템, 그리고 지속적 개선 파이프라인을 구현하는 방법을 통해 에이전트의 품질과 성능을 지속적으로 향상시킬 수 있습니다.

에이전트를 체계적으로 평가하고 모니터링함으로써 AI 시스템의 안정성과 신뢰성을 확보할 수 있으며, 사용자에게 더 나은 경험을 제공할 수 있습니다. 또한, 지속적인 개선 프로세스를 통해 시간이 지남에 따라 에이전트의 성능이 향상되도록 할 수 있습니다.

이로써 "Part V: AI 에이전트 설계와 구현"을 마무리합니다. 다음 파트에서는 지금까지 배운 내용을 바탕으로 실전 프로젝트와 케이스 스터디를 통해 Spring AI의 활용 방법을 더 깊이 살펴보겠습니다.

## 연습 문제

1. Spring AI 에이전트를 평가하기 위한 간단한 평가 프레임워크를 설계하고 구현해 보세요. 정확성, 일관성, 응답 시간 등의 지표를 포함해야 합니다.
2. 에이전트의 응답을 평가하기 위한 의미적 유사성 메트릭을 구현해 보세요. Embedding 서비스를 활용하여 코사인 유사도를 계산하는 방법을 사용하세요.
3. 에이전트 실행을 추적하고 디버깅하기 위한 로깅 시스템을 구현해 보세요. 각 단계별 실행 시간과 중간 결과를 기록해야 합니다.
4. 에이전트의 성능을 실시간으로 모니터링하기 위한 메트릭 수집 시스템을 설계하고 구현해 보세요. Micrometer와 Spring Boot Actuator를 활용하세요.
5. 두 에이전트 버전을 비교하기 위한 A/B 테스트 시스템을 구현해 보세요. 사용자 요청을 무작위로 두 버전에 분배하고 결과를 비교하는 기능을 포함해야 합니다.