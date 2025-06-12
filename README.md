# Spring AI 완벽 가이드

Spring AI는 Spring 생태계의 모듈화·POJO 지향 철학을 AI 도메인에 적용하여, 개발자가 최소한의 설정으로 다양한 AI 모델을 교체 가능하게 통합할 수 있는 애플리케이션 프레임워크입니다.

## 최신 버전 정보

2025년 2월 공개된 1.0.0 Milestone 6(M6) 버전은 다음과 같은 주요 기능을 포함합니다:
- `@Tool` / `@ToolParam` 어노테이션을 통한 향상된 Function Calling
- Model Context Protocol(MCP) SDK 지원
- 개선된 VectorStore API
- Spring AI Agent API (실험적 기능)

([spring.io][1], [galaxy.ai][2], [infoq.com][3])

## 주요 특징

**1. 통합 AI 모델 지원**
- OpenAI, Anthropic, Microsoft Azure, Amazon Bedrock, Google Gemini, Ollama 등 주요 모델 프로바이더를 단일 API로 지원
- Chat Completion, Embedding, Text-to-Image, Transcription, Text-to-Speech, Moderation 등 다양한 모델 타입 지원
- 동기 및 스트리밍 호출 모두 제공 ([spring.io][4])

**2. RAG 및 벡터 스토어**
- VectorStore API를 통한 벡터 데이터베이스 통합
- ChromaDB, Pinecone, Postgres pgvector 등 다양한 벡터 스토어 지원
- Retrieval-Augmented Generation(RAG) 패턴의 손쉬운 구현 ([docs.spring.io][5], [piotrminkowski.com][6], [baeldung.com][7])

**3. 엔터프라이즈 준비**
- 강력한 테스트 및 모킹 지원
- 포괄적인 관찰가능성(Observability) 기능
- 보안 및 비용 관리 기능
- Spring Boot와의 완벽한 통합 ([piotrminkowski.com][6])

## 책의 구성

이 책은 Spring AI의 기초부터 고급 기능, 실전 프로젝트까지 체계적으로 다룹니다. 각 장은 실습 예제와 함께 구성되어 있어 따라하며 학습할 수 있습니다.

---

## 목차

### Part I: AI와 Spring AI 개요

1. **[AI 트렌드와 핵심 개념](Part_I_AI_and_Spring_AI_Overview/01_AI_Trends_and_Key_Concepts.md)**
   - AI의 현재와 미래
   - 생성형 AI와 LLM의 이해
   - AI 애플리케이션 아키텍처 패턴

2. **[Spring AI의 탄생 배경과 목표](Part_I_AI_and_Spring_AI_Overview/02_Spring_AI_Background_and_Goals.md)**
   - Spring AI의 철학과 비전
   - 경쟁 프레임워크와의 차별점
   - Spring 생태계와의 통합

3. **[개발 환경 설정](Part_I_AI_and_Spring_AI_Overview/03_Development_Environment_Setup.md)**
   - Java 21, Spring Boot 3.3, Spring AI 1.0.0-M6 설정
   - IDE 설정 및 도구 구성
   - API 키 관리와 보안

### Part II: Spring AI 기초 다지기

4. **[Spring AI 아키텍처](Part_II_Spring_AI_Fundamentals/04_Spring_AI_Architecture.md)**
   - Provider, Model, Template, Client, Message 구조
   - 핵심 추상화와 인터페이스
   - 설정과 커스터마이징

5. **[첫 번째 프로젝트: Chat Model 연동](Part_II_Spring_AI_Fundamentals/05_First_Project_Chat_Model_Integration.md)**
   - OpenAI와 Anthropic 모델 연동
   - 프롬프트 엔지니어링 기초
   - 응답 처리와 오류 관리

6. **[Embedding과 벡터 스토어 기본](Part_II_Spring_AI_Fundamentals/06_Embeddings_and_Vector_Stores.md)**
   - 임베딩의 개념과 활용
   - 벡터 스토어 통합
   - 유사성 검색 구현

### Part III: 고급 기능 집중 탐구

7. **[Retrieval-Augmented Generation(RAG) 패턴 구현](Part_III_Advanced_Features/07_RAG_Pattern_Implementation.md)**
   - RAG 아키텍처 설계
   - 문서 처리 파이프라인
   - 검색 최적화 전략

8. **[Function Calling & `@Tool` 어노테이션](Part_III_Advanced_Features/08_Function_Calling_and_Tool_Annotation.md)**
   - Function Calling 메커니즘
   - `@Tool` / `@ToolParam` 활용
   - 동적 도구 등록

9. **[멀티모달 AI](Part_III_Advanced_Features/09_Multimodal_AI_Integration.md)**
   - 이미지 생성과 분석
   - 음성 인식과 합성
   - 멀티모달 파이프라인 구축

10. **[스트리밍 응답과 비동기 처리](Part_III_Advanced_Features/10_Streaming_Responses_and_Async_Processing.md)**
    - 스트리밍 API 활용
    - 비동기 처리 패턴
    - 성능 최적화

### Part IV: 품질·보안·관측성

11. **[AI 애플리케이션 테스트 전략](Part_IV_Quality_Security_Observability/11_AI_Application_Testing_Strategy.md)**
    - Mock LLM과 테스트 더블
    - 녹화·재생 패턴
    - 평가 메트릭 구현

12. **[데이터 보안·비용 관리](Part_IV_Quality_Security_Observability/12_Data_Security_Cost_Management.md)**
    - 민감 데이터 보호
    - API 키 관리
    - 비용 모니터링과 최적화

13. **[Observability와 로깅](Part_IV_Quality_Security_Observability/13_Observability_and_Logging.md)**
    - 메트릭과 추적
    - 로깅 전략
    - 모니터링 대시보드 구축

### Part V: AI 에이전트 설계와 구현

14. **[AI 에이전트 기초](Part_V_AI_Agent_Design_and_Implementation/14_AI_Agent_Fundamentals.md)**
    - 에이전트 아키텍처 패턴
    - 추론과 계획 메커니즘
    - 메모리와 상태 관리

15. **[Spring AI로 에이전트 구축하기](Part_V_AI_Agent_Design_and_Implementation/15_Building_Agents_with_Spring_AI.md)**
    - Spring AI Agent API 활용
    - Model Context Protocol(MCP) 통합
    - Google ADK와의 연동

16. **[A2A(Agent-to-Agent) 프로토콜](Part_V_AI_Agent_Design_and_Implementation/16_A2A_Agent_to_Agent_Protocol.md)**
    - Google A2A Protocol v0.2.1 구현
    - 에이전트 간 통신 패턴
    - 분산 에이전트 시스템

17. **[Spring AI Agent API](Part_V_AI_Agent_Design_and_Implementation/17_Spring_AI_Agent_API.md)**
    - Agent, Planner, ToolContext API
    - 에이전트 빌더 패턴
    - 커스텀 에이전트 구현

18. **[에이전트 체인 & 워크플로 구축](Part_V_AI_Agent_Design_and_Implementation/18_Agent_Chains_and_Workflows.md)**
    - 에이전트 오케스트레이션
    - 복잡한 워크플로 설계
    - 효과적인 에이전트 패턴

19. **[에이전트 평가·디버깅·모니터링](Part_V_AI_Agent_Design_and_Implementation/19_Agent_Evaluation_Debugging_Monitoring.md)**
    - 에이전트 성능 평가
    - 프로덕션 디버깅 기법
    - 고급 모니터링 구성

### Part VI: 실전 프로젝트 케이스스터디

20. **[고객 지원 챗봇 구축](Part_VI_Practical_Projects_Case_Studies/20_Customer_Support_Chatbot.md)**
    - 요구사항 분석과 설계
    - RAG 기반 지식베이스 구축
    - 대화 관리와 컨텍스트 유지

21. **[문서 Q&A 검색 서비스(RAG)](Part_VI_Practical_Projects_Case_Studies/21_Document_QA_Search_Service.md)**
    - 대용량 문서 처리
    - 검색 정확도 최적화
    - 답변 생성과 인용 처리

22. **[이미지 생성 웹서비스](Part_VI_Practical_Projects_Case_Studies/22_Image_Generation_Web_Service.md)**
    - 프롬프트 엔지니어링
    - 이미지 생성 파이프라인
    - 갤러리와 사용자 관리

23. **[음성 비서 API](Part_VI_Practical_Projects_Case_Studies/23_Voice_Assistant_API.md)**
    - 음성 인식과 합성 통합
    - 실시간 대화 처리
    - 멀티턴 대화 관리

### Part VII: 부록

A. **[AI 윤리와 책임 있는 AI 개발](Part_VII_Appendices/A_AI_Ethics_and_Responsible_AI_Development.md)**
   - AI 윤리 원칙
   - 편향성과 공정성
   - 투명성과 설명가능성

B. **[주요 프로바이더 설정 가이드](Part_VII_Appendices/B_Provider_Configuration_Guide.md)**
   - OpenAI, Anthropic, Azure 설정
   - Bedrock, Vertex AI 설정
   - 로컬 모델 (Ollama) 설정

C. **[Spring AI 커뮤니티 참여와 기여 방법](Part_VII_Appendices/C_Community_Participation_and_Contribution.md)**
   - 오픈소스 기여 가이드
   - 커뮤니티 리소스
   - 문제 해결과 지원

---

## 이 책의 특징

- **실습 중심**: 모든 장에 실행 가능한 예제 코드 포함
- **최신 버전 대응**: Spring AI 1.0.0-M6 기준으로 작성
- **실무 적용**: 실제 프로젝트에 바로 적용 가능한 패턴과 모범 사례
- **점진적 학습**: 기초부터 고급까지 단계별 학습 구조

## 대상 독자

- Spring Framework 경험이 있는 Java 개발자
- AI/ML을 애플리케이션에 통합하고자 하는 개발자
- 엔터프라이즈 환경에서 AI 솔루션을 구축하는 아키텍트
- Spring AI의 최신 기능을 학습하고자 하는 모든 개발자

## 예제 코드

모든 예제 코드는 [GitHub 저장소](https://github.com/revfactory/spring-ai-book)에서 확인할 수 있습니다.

---

[1]: https://spring.io/blog/2025/02/14/spring-ai-1-0-0-m6-released "Spring AI 1.0.0 M6 Released"
[2]: https://galaxy.ai/youtube-summarizer/exploring-the-new-features-of-spring-ai-m6-and-the-mcp-protocol-cE1h-rC2o2U "Exploring the New Features of Spring AI M6 and the mCP Protocol"
[3]: https://www.infoq.com/news/2025/02/spring-news-roundup-feb17-2025/ "Spring News Roundup: Milestone Releases of Boot, Security, Auth Server, Integration ..."
[4]: https://spring.io/projects/spring-ai "Spring AI"
[5]: https://docs.spring.io/spring-ai/reference/api/vectordbs.html "Vector Databases :: Spring AI Reference"
[6]: https://piotrminkowski.com/2025/02/24/using-rag-and-vector-store-with-spring-ai/ "Using RAG and Vector Store with Spring AI - Piotr's TechBlog"
[7]: https://www.baeldung.com/spring-ai-chromadb-vector-store "Spring AI With ChromaDB Vector Store - Baeldung"