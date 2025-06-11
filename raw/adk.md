# Google ADK (Agent Development Kit) 스펙 상세 조사 보고서

## Google ADK의 정확한 정의와 목적

Google Agent Development Kit (ADK)는 2025년 4월 Google Cloud NEXT에서 발표된 **오픈소스 코드-우선(code-first) 프레임워크**로, 정교한 AI 에이전트와 멀티 에이전트 시스템을 개발, 평가, 배포할 수 있도록 설계되었습니다. Google의 Agentspace와 Customer Engagement Suite (CES)에서 실제로 사용되는 동일한 프레임워크를 기반으로 하며, 개발자에게 유연성과 정밀한 제어를 제공합니다.

**주요 목적:**
- 복잡한 비즈니스 프로세스 자동화를 위한 멀티 에이전트 시스템 구축
- 다양한 데이터 소스를 활용한 자율적 분석 및 인사이트 도출
- 엔터프라이즈급 보안과 확장성을 갖춘 프로덕션 레디 에이전트 개발
- Google Cloud 에코시스템과의 원활한 통합 지원

## 주요 기능과 특징

### 핵심 기능

**1. 멀티 에이전트 설계 (Multi-Agent by Design)**
- 모듈식 아키텍처로 특화된 에이전트들을 계층적으로 구성
- 코디네이터-워커 패턴, 순차/병렬 처리 패턴 지원
- 동적 라우팅 및 에이전트 간 작업 위임

**2. 풍부한 모델 생태계**
- Gemini 모델에 최적화되어 있으나 모델에 구애받지 않음
- LiteLLM 통합으로 GPT-4, Claude, Mistral 등 200개 이상의 모델 지원
- Vertex AI Model Garden을 통한 다양한 모델 접근

**3. 도구 생태계**
- 사전 구축된 도구 (웹 검색, 코드 실행 등)
- Model Context Protocol (MCP) 통합
- LangChain, LlamaIndex, CrewAI와의 호환성
- 커스텀 함수 및 OpenAPI 스펙 지원

**4. 양방향 스트리밍**
- 실시간 오디오/비디오 스트리밍 기능
- 텍스트를 넘어선 풍부한 멀티모달 상호작용
- 인간과 같은 자연스러운 대화 인터페이스 구현

**5. 평가 프레임워크**
- 에이전트 성능의 체계적 평가
- 최종 응답 품질 및 단계별 실행 궤적 분석
- 사전 정의된 테스트 케이스에 대한 검증

## 아키텍처 및 구성 요소

### 핵심 아키텍처

**기본 컴포넌트:**
- **BaseAgent**: 모든 에이전트의 기반 클래스
- **LlmAgent/Agent**: 지능적 추론을 위한 LLM 기반 에이전트
- **Workflow Agents**: Sequential, Parallel, Loop 에이전트
- **Custom Agents**: 특화된 로직을 위한 확장 BaseAgent 구현

**멀티 에이전트 시스템 아키텍처:**
```python
# 계층적 에이전트 구성 예시
coordinator = LlmAgent(
    name="Coordinator",
    model="gemini-2.0-flash",
    description="작업 조정 에이전트",
    sub_agents=[greeter, task_executor]
)
```

**주요 아키텍처 패턴:**
- 에이전트 오케스트레이션: LLM 기반 워크플로우 또는 결정론적 패턴
- 도구 기반 추론: 다양한 기능을 도구로 장착한 에이전트
- 상태 관리: 세션 상태, 메모리, 컨텍스트 보존

## API 스펙과 프로토콜

### Python ADK v1.0.0 (프로덕션 준비 완료)

**설치:**
```bash
pip install google-adk

# 개발 버전
pip install git+https://github.com/google/adk-python.git@main

# 옵션 확장
pip install google-adk[tracing]    # 모니터링 및 추적
pip install google-adk[web]        # 개발 UI
pip install google-adk[voice]      # 음성 지원
```

**기본 사용 예제:**
```python
from google.adk.agents import Agent
from google.adk.tools import google_search

root_agent = Agent(
    name="search_assistant",
    model="gemini-2.0-flash",
    instruction="검색이 필요할 때 Google Search를 사용하여 질문에 답하세요.",
    description="웹을 검색할 수 있는 어시스턴트",
    tools=[google_search]
)
```

### Java ADK v0.1.0 (초기 릴리스)

**Maven 설정:**
```xml
<dependency>
    <groupId>com.google.adk</groupId>
    <artifactId>google-adk</artifactId>
    <version>0.1.0</version>
</dependency>
```

**Java 예제:**
```java
import com.google.adk.agents.LlmAgent;
import com.google.adk.tools.GoogleSearchTool;

LlmAgent rootAgent = LlmAgent.builder()
    .name("search_assistant")
    .model("gemini-2.0-flash")
    .instruction("검색 어시스턴트입니다.")
    .tools(new GoogleSearchTool())
    .build();
```

### 통신 프로토콜

**1. HTTP/REST**
- FastAPI 통합: `get_fast_api_app()`를 통한 내장 서버 생성
- WebSocket 지원: 실시간 양방향 통신

**2. gRPC**
- Google Cloud 서비스와의 네이티브 통합
- Protocol Buffers를 통한 효율적인 직렬화

**3. Agent-to-Agent (A2A) Protocol**
- OpenAPI 기반 표준화된 에이전트 발견 및 통신
- 에이전트 카드를 통한 서비스 디스커버리
- OAuth 2.0, API 키, 서비스 계정 인증 지원

**4. Model Context Protocol (MCP)**
- SSE(Server-Sent Events) 전송
- 도구 통합을 위한 표준화된 통신

## Spring AI와 Google ADK를 활용한 A2A 통신 구현

### Spring AI에서 A2A 통신 구현 방법

**1. Google Vertex AI Gemini 통합**

Spring AI는 Google Vertex AI와의 네이티브 통합을 제공합니다:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-vertex-ai-gemini-spring-boot-starter</artifactId>
</dependency>
```

**2. 설정 구성**

```properties
spring.ai.vertex.ai.gemini.project-id=${GOOGLE_CLOUD_PROJECT}
spring.ai.vertex.ai.gemini.location=us-east4
spring.ai.vertex.ai.gemini.chat.options.model=gemini-2.0-flash
spring.ai.vertex.ai.gemini.chat.options.temperature=0.5
```

**3. A2A 프로토콜 활용 전략**

Spring AI에서 Google ADK의 A2A 프로토콜을 활용하는 방법:

- **MCP 클라이언트 구성**: Spring AI의 MCP 지원을 통해 ADK 에이전트와 통신
- **REST API 통합**: ADK 에이전트를 FastAPI로 노출하고 Spring RestTemplate/WebClient로 통신
- **비동기 메시징**: Spring의 메시징 프레임워크와 A2A의 SSE 지원 활용

### 실제 구현 패턴

**1. Spring AI 에이전트와 Google ADK 에이전트 간 통신**

```java
@Service
public class SpringAIToADKBridge {
    private final WebClient webClient;
    
    public Mono<String> communicateWithADKAgent(String message) {
        return webClient.post()
            .uri("http://adk-agent-host/api/agent/task")
            .bodyValue(Map.of("message", message))
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

**2. MCP를 통한 도구 공유**

```java
@Component
public class MCPToolProvider {
    @Tool(name = "processData", description = "데이터 처리 도구")
    public String processData(String input) {
        // Spring AI 에이전트의 도구를 ADK 에이전트가 사용
        return processedResult;
    }
}
```

## Spring AI와 Google ADK 통합 방법

### 통합 아키텍처

**1. 하이브리드 접근법**
- Spring AI: 비즈니스 로직과 엔터프라이즈 통합 담당
- Google ADK: 복잡한 멀티 에이전트 오케스트레이션 처리
- A2A 프로토콜: 두 시스템 간 표준화된 통신

**2. 통합 포인트**

```java
@Configuration
public class ADKIntegrationConfig {
    @Bean
    public ADKClient adkClient() {
        return ADKClient.builder()
            .baseUrl("http://adk-engine-host")
            .authentication(OAuth2Authentication.builder()
                .clientId(clientId)
                .clientSecret(clientSecret)
                .build())
            .build();
    }
}
```

### 베스트 프랙티스

**1. 역할 분담**
- Spring AI: 트랜잭션 관리, 데이터 접근, 비즈니스 규칙
- Google ADK: AI 추론, 멀티 에이전트 조정, 복잡한 워크플로우

**2. 보안 고려사항**
- Google Cloud IAM과 Spring Security 통합
- 서비스 계정을 통한 인증
- VPC 및 네트워크 격리

**3. 성능 최적화**
- 비동기 통신 패턴 활용
- 연결 풀링 구성
- 캐싱 전략 구현

## 실제 구현 예제

### 멀티 에이전트 시스템 구현

**1. ADK 에이전트 정의**

```python
# research_agent.py
from google.adk.agents import Agent
from google.adk.tools import google_search, web_fetch

research_agent = Agent(
    name="research_agent",
    model="gemini-2.0-flash",
    instruction="주어진 주제에 대해 심층적인 조사를 수행합니다.",
    tools=[google_search, web_fetch]
)

# API 서버로 노출
from google.adk.runners import get_fast_api_app
app = get_fast_api_app(research_agent)
```

**2. Spring AI 통합 구현**

```java
@RestController
@RequestMapping("/api/research")
public class ResearchController {
    private final ChatClient chatClient;
    private final ADKClient adkClient;
    
    @PostMapping("/analyze")
    public Mono<ResearchResult> analyzeTopics(@RequestBody ResearchRequest request) {
        // Spring AI로 초기 분석
        String analysis = chatClient.prompt()
            .user(request.getTopic())
            .call()
            .content();
        
        // ADK 에이전트로 심층 조사 위임
        return adkClient.createTask(Map.of(
            "action", "deep_research",
            "topic", request.getTopic(),
            "initial_analysis", analysis
        ))
        .map(this::processResults);
    }
}
```

## 최신 버전 정보와 업데이트

### 현재 버전 (2025년 6월 기준)

**Python ADK**
- 안정 버전: v1.0.0 (프로덕션 준비 완료)
- 주간 릴리스 주기
- pip install google-adk로 설치

**Java ADK**
- 초기 릴리스: v0.1.0
- Maven/Gradle을 통한 설치 지원

**A2A Protocol**
- 현재 버전: v0.2
- 무상태 상호작용 지원 추가
- 표준화된 인증 스키마

### 주요 업데이트 사항

**1. 2025년 4월 - Google Cloud NEXT 발표**
- ADK 공식 오픈소스 릴리스
- 50개 이상 파트너사와 A2A 프로토콜 발표

**2. 2025년 5월 - Google I/O 업데이트**
- Python ADK v1.0.0 안정 버전 릴리스
- Java ADK v0.1.0 초기 릴리스
- A2A Protocol v0.2 발표

**3. 주요 기능 추가**
- 양방향 오디오/비디오 스트리밍
- MCP 통합 강화
- Vertex AI Agent Engine 통합

## 관련 문서 및 리소스

### 공식 문서
- **ADK 문서**: https://google.github.io/adk-docs/
- **빠른 시작 가이드**: https://google.github.io/adk-docs/get-started/quickstart/
- **Vertex AI Agent Builder**: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-builder
- **A2A Protocol 스펙**: https://github.com/google-a2a/A2A/blob/main/docs/specification.md

### GitHub 저장소
- **Python ADK**: https://github.com/google/adk-python
- **Java ADK**: https://github.com/google/adk-java
- **ADK 샘플**: https://github.com/google/adk-samples
- **A2A 프로토콜**: https://github.com/google-a2a/A2A

### 커뮤니티 리소스
- **Stack Overflow**: google-agent-development-kit 태그
- **GitHub Discussions**: 각 저장소의 Discussions 섹션
- **Google Cloud Skills Boost**: AI 에이전트 실습 및 교육

### 파트너 및 통합
- Box, Salesforce, SAP, ServiceNow 등 50개 이상 기업 지원
- LangChain, CrewAI, AutoGen과의 호환성
- Spring AI를 통한 Java 생태계 통합

## 결론

Google ADK는 멀티 에이전트 시스템 개발을 위한 강력하고 유연한 프레임워크로, Spring AI와의 통합을 통해 엔터프라이즈 Java 애플리케이션에서도 활용할 수 있습니다. A2A 프로토콜을 통해 다양한 프레임워크와 벤더 간의 에이전트 상호운용성을 보장하며, Google Cloud 생태계와의 깊은 통합을 제공합니다. 개발자들은 ADK의 코드 우선 접근법과 풍부한 도구 생태계를 활용하여 복잡한 비즈니스 문제를 해결하는 지능형 에이전트 시스템을 구축할 수 있습니다.
