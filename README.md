# Spring AI 완벽 가이드

Spring AI는 Spring 생태계의 모듈화·POJO 지향 철학을 AI 도메인에 적용하여, 개발자가 최소한의 설정으로 다양한 AI 모델을 교체 가능하게 통합할 수 있는 애플리케이션 프레임워크입니다.

## 최신 버전 정보

**Spring AI 1.0.0 GA 정식 출시** (2025년 5월 20일)

2년 이상의 개발과 8번의 마일스톤을 거쳐 Spring AI 1.0.0 GA가 정식 출시되었습니다. 이번 GA 버전은 프로덕션 환경에서 사용할 수 있는 완전한 AI 애플리케이션 프레임워크를 제공합니다.

### 주요 GA 기능
- **ChatClient API**: 20개 이상의 AI 모델을 지원하는 통합 클라이언트 API
- **Tool/Function Calling**: `@Tool` 어노테이션을 통한 선언적 도구 정의
- **Model Context Protocol (MCP)**: OAuth2 보안과 함께 완전한 MCP 지원
- **Agent API**: 계획, 메모리, 액션을 결합한 AI 에이전트 구현
- **Structured Outputs**: AI 응답을 POJO로 매핑하는 타입 안전 처리
- **Advisors API**: 프롬프트 수정을 위한 인터셉터 체인

([Spring AI 1.0 GA 공식 발표](https://spring.io/blog/2025/05/20/spring-ai-1-0-GA-released/), [InfoQ 리뷰](https://www.infoq.com/news/2025/05/spring-ai-1-0-streamlines-apps/), [Microsoft Azure 통합](https://techcommunity.microsoft.com/blog/appsonazureblog/spring-ai-1-0-ga-is-here---build-java-ai-apps-end-to-end-on-azure-today/4414763))

## 주요 특징

**1. 20개 이상 AI 모델 지원**
- **메이저 프로바이더**: OpenAI, Anthropic, Microsoft Azure OpenAI, Amazon Bedrock, Google Vertex AI
- **아시아 프로바이더**: ZhiPu AI, Qianfan, Moonshot 등
- **로컬 모델**: Ollama, Hugging Face 등
- **멀티모달**: Chat, Embedding, Image, Audio, Transcription, Text-to-Speech, Moderation 지원

**2. ChatClient API**
- 모든 AI 모델에 대한 통합된 포털 인터페이스
- 멀티모달 입력/출력 지원 (모델이 지원하는 경우)
- Structured Output을 통한 JSON 응답 자동 파싱
- 동기/스트리밍 호출 모두 지원

**3. RAG 및 벡터 스토어**
- **벡터 데이터베이스**: Cassandra, PostgreSQL/PGVector, MongoDB Atlas, Milvus, Pinecone, Redis 등
- 일관된 Vector Store API를 통한 엔터프라이즈 데이터 그라운딩
- Retrieval-Augmented Generation(RAG) 패턴 완벽 지원

**4. 프로덕션 준비 기능**
- Spring 패턴 및 관례 준수로 기존 개발자 친화적
- Spring Security와 Authorization Server를 통한 OAuth2 보안
- 포괄적인 관찰가능성(Observability) 및 모니터링
- OpenRewrite 레시피를 통한 자동 업그레이드 지원

## 책의 구성

이 책은 Spring AI의 기초부터 고급 기능, 실전 프로젝트까지 체계적으로 다룹니다. 각 장은 실습 예제와 함께 구성되어 있어 따라하며 학습할 수 있습니다.

---

## 목차

### Part 01: AI와 Spring AI 개요

1. **[AI 트렌드와 핵심 개념](Part01_AI와_Spring_AI_개요/01_AI_Trends_and_Key_Concepts.md)**
   - AI의 현재와 미래
   - 생성형 AI와 LLM의 이해
   - AI 애플리케이션 아키텍처 패턴

2. **[Spring AI의 탄생 배경과 목표](Part01_AI와_Spring_AI_개요/02_Spring_AI_Background_and_Goals.md)**
   - Spring AI의 철학과 비전
   - 경쟁 프레임워크와의 차별점
   - Spring 생태계와의 통합

3. **[개발 환경 설정](Part01_AI와_Spring_AI_개요/03_Development_Environment_Setup.md)**
   - Java 21, Spring Boot 3.3+, Spring AI 1.0.0 GA 설정
   - IDE 설정 및 도구 구성
   - API 키 관리와 보안

### Part 02: Spring AI 기초

4. **[Spring AI 아키텍처](Part02_Spring_AI_기초/04_Spring_AI_Architecture.md)**
   - Provider, Model, Template, Client, Message 구조
   - 핵심 추상화와 인터페이스
   - 설정과 커스터마이징

5. **[첫 번째 프로젝트: Chat Model 연동](Part02_Spring_AI_기초/05_First_Project_Chat_Model_Integration.md)**
   - OpenAI와 Anthropic 모델 연동
   - 프롬프트 엔지니어링 기초
   - 응답 처리와 오류 관리

6. **[Embedding과 벡터 스토어 기본](Part02_Spring_AI_기초/06_Embeddings_and_Vector_Stores.md)**
   - 임베딩의 개념과 활용
   - 벡터 스토어 통합
   - 유사성 검색 구현

### Part 03: 고급 기능

7. **[Retrieval-Augmented Generation(RAG) 패턴 구현](Part03_고급_기능/07_RAG_Pattern_Implementation.md)**
   - RAG 아키텍처 설계
   - 문서 처리 파이프라인
   - 검색 최적화 전략

8. **[Function Calling & `@Tool` 어노테이션](Part03_고급_기능/08_Function_Calling_and_Tool_Annotation.md)**
   - Function Calling 메커니즘
   - `@Tool` / `@ToolParam` 활용
   - 동적 도구 등록

9. **[멀티모달 AI](Part03_고급_기능/09_Multimodal_AI_Integration.md)**
   - 이미지 생성과 분석
   - 음성 인식과 합성
   - 멀티모달 파이프라인 구축

10. **[스트리밍 응답과 비동기 처리](Part03_고급_기능/10_Streaming_Responses_and_Async_Processing.md)**
    - 스트리밍 API 활용
    - 비동기 처리 패턴
    - 성능 최적화

### Part 04: 품질·보안·관측성

11. **[AI 애플리케이션 테스트 전략](Part04_품질_보안_관측성/11_AI_Application_Testing_Strategy.md)**
    - Mock LLM과 테스트 더블
    - 녹화·재생 패턴
    - 평가 메트릭 구현

12. **[데이터 보안·비용 관리](Part04_품질_보안_관측성/12_Data_Security_Cost_Management.md)**
    - 민감 데이터 보호
    - API 키 관리
    - 비용 모니터링과 최적화

13. **[Observability와 로깅](Part04_품질_보안_관측성/13_Observability_and_Logging.md)**
    - 메트릭과 추적
    - 로깅 전략
    - 모니터링 대시보드 구축

### Part 05: AI 에이전트 설계와 구현

14. **[AI 에이전트 기초](Part05_AI_에이전트_설계와_구현/14_AI_Agent_Fundamentals.md)**
    - 에이전트 아키텍처 패턴
    - 추론과 계획 메커니즘
    - 메모리와 상태 관리

15. **[Spring AI로 에이전트 구축하기](Part05_AI_에이전트_설계와_구현/15_Building_Agents_with_Spring_AI.md)**
    - Spring AI Agent API 활용
    - Model Context Protocol(MCP) 통합
    - Google ADK와의 연동

16. **[A2A(Agent-to-Agent) 프로토콜](Part05_AI_에이전트_설계와_구현/16_A2A_Agent_to_Agent_Protocol.md)**
    - Google A2A Protocol v0.2.1 구현
    - 에이전트 간 통신 패턴
    - 분산 에이전트 시스템

17. **[Spring AI Agent API](Part05_AI_에이전트_설계와_구현/17_Spring_AI_Agent_API.md)**
    - Agent, Planner, ToolContext API
    - 에이전트 빌더 패턴
    - 커스텀 에이전트 구현

18. **[에이전트 체인 & 워크플로 구축](Part05_AI_에이전트_설계와_구현/18_Agent_Chains_and_Workflows.md)**
    - 에이전트 오케스트레이션
    - 복잡한 워크플로 설계
    - 효과적인 에이전트 패턴

19. **[에이전트 평가·디버깅·모니터링](Part05_AI_에이전트_설계와_구현/19_Agent_Evaluation_Debugging_Monitoring.md)**
    - 에이전트 성능 평가
    - 프로덕션 디버깅 기법
    - 고급 모니터링 구성

### Part 06: 실전 프로젝트 케이스스터디

20. **[고객 지원 챗봇 구축](Part06_실전_프로젝트_케이스스터디/20_Customer_Support_Chatbot.md)**
    - 요구사항 분석과 설계
    - RAG 기반 지식베이스 구축
    - 대화 관리와 컨텍스트 유지

21. **[문서 Q&A 검색 서비스(RAG)](Part06_실전_프로젝트_케이스스터디/21_Document_QA_Search_Service.md)**
    - 대용량 문서 처리
    - 검색 정확도 최적화
    - 답변 생성과 인용 처리

22. **[이미지 생성 웹서비스](Part06_실전_프로젝트_케이스스터디/22_Image_Generation_Web_Service.md)**
    - 프롬프트 엔지니어링
    - 이미지 생성 파이프라인
    - 갤러리와 사용자 관리

23. **[음성 비서 API](Part06_실전_프로젝트_케이스스터디/23_Voice_Assistant_API.md)**
    - 음성 인식과 합성 통합
    - 실시간 대화 처리
    - 멀티턴 대화 관리

### Part 07: 부록

A. **[AI 윤리와 책임 있는 AI 개발](Part07_부록/A_AI_Ethics_and_Responsible_AI_Development.md)**
   - AI 윤리 원칙
   - 편향성과 공정성
   - 투명성과 설명가능성

B. **[주요 프로바이더 설정 가이드](Part07_부록/B_Provider_Configuration_Guide.md)**
   - OpenAI, Anthropic, Azure 설정
   - Bedrock, Vertex AI 설정
   - 로컬 모델 (Ollama) 설정

C. **[Spring AI 커뮤니티 참여와 기여 방법](Part07_부록/C_Community_Participation_and_Contribution.md)**
   - 오픈소스 기여 가이드
   - 커뮤니티 리소스
   - 문제 해결과 지원

---

## 이 책의 특징

- **실습 중심**: 모든 장에 실행 가능한 예제 코드 포함
- **최신 버전 대응**: Spring AI 1.0.0 GA 기준으로 작성
- **프로덕션 준비**: 실제 프로덕션 환경에서 사용 가능한 패턴과 모범 사례
- **점진적 학습**: 기초부터 고급까지 단계별 학습 구조
- **에이전트 중심**: 2025년 AI 에이전트 시대에 맞춘 심화 내용

## 대상 독자

- Spring Framework 경험이 있는 Java 개발자
- AI/ML을 애플리케이션에 통합하고자 하는 개발자
- 엔터프라이즈 환경에서 AI 솔루션을 구축하는 아키텍트
- AI 에이전트 시스템을 구축하려는 개발자
- Spring AI 1.0.0 GA의 최신 기능을 학습하고자 하는 모든 개발자

## 시작하기

### Maven 의존성

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Gradle 의존성

```gradle
dependencies {
    implementation platform('org.springframework.ai:spring-ai-bom:1.0.0')
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
}
```

## 예제 코드

모든 예제 코드는 [GitHub 저장소](https://github.com/revfactory/spring-ai-book)에서 확인할 수 있습니다.

---

## 업그레이드 가이드

기존 Spring AI 마일스톤 버전에서 1.0.0 GA로 업그레이드하려면 [OpenRewrite 레시피](https://github.com/openrewrite/rewrite-spring/tree/main/src/main/resources/META-INF/rewrite/spring-ai)를 사용하여 자동화할 수 있습니다.

## 관련 링크

- [Spring AI 1.0.0 GA 공식 발표](https://spring.io/blog/2025/05/20/spring-ai-1-0-GA-released/)
- [Spring AI 공식 문서](https://docs.spring.io/spring-ai/reference/)
- [Spring AI GitHub 저장소](https://github.com/spring-projects/spring-ai)
- [InfoQ: Spring AI 1.0 출시 뉴스](https://www.infoq.com/news/2025/05/spring-ai-1-0-streamlines-apps/)
- [Microsoft Azure Spring AI 통합](https://techcommunity.microsoft.com/blog/appsonazureblog/spring-ai-1-0-ga-is-here---build-java-ai-apps-end-to-end-on-azure-today/4414763)

---

**주의**: 이 책은 Spring AI 1.0.0 GA 버전을 기준으로 작성되었으며, 프로덕션 환경에서 안전하게 사용할 수 있는 실무 중심의 내용을 담고 있습니다.