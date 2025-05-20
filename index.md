Spring AI는 Spring 생태계의 모듈화·POJO 지향 철학을 AI 도메인에 적용하여, 개발자가 최소한의 설정으로 다양한 AI 모델을 교체 가능하게 통합할 수 있는 애플리케이션 프레임워크입니다.&#x20;
2025년 2월 공개된 1.0.0 Milestone 6(M6) 버전은 `@Tool` / `@ToolParam` 어노테이션과 Model Context Protocol(MCP) SDK, 개선된 VectorStore API를 포함해 최신 기능을 대거 도입했습니다. ([spring.io][1], [galaxy.ai][2], [infoq.com][3])
Spring AI는 OpenAI·Anthropic·Microsoft Azure·Amazon Bedrock·Google Gemini·Ollama 등 주요 모델 프로바이더를 단일 API로 지원하며, Chat Completion·Embedding·Text-to-Image·Transcription·Text-to-Speech·Moderation과 같은 여러 모델 타입에 대해 동기 및 스트리밍 호출을 제공합니다. ([spring.io][4])
VectorStore API와 Retrieval-Augmented Generation(RAG) 지원 덕분에 ChromaDB·Pinecone·Postgres pgvector 등 벡터 데이터베이스를 쉽게 연결해 문서 검색 및 컨텍스트 주입 기반 애플리케이션을 구현할 수 있습니다. ([docs.spring.io][5], [piotrminkowski.com][6], [baeldung.com][7])
또한 OpenAI Function Calling을 자바 메서드에 자동 매핑하는 기능과 강력한 테스트/모킹 지원으로 실무 프로젝트의 유지보수성과 안정성이 크게 향상되었습니다. ([piotrminkowski.com][6])
공식 레퍼런스와 Baeldung·국내 개발 블로그 등 다수의 튜토리얼이 Spring Boot 애플리케이션에서 이러한 기능을 단계별로 설명하고 있어 초심자도 빠르게 따라 해 볼 수 있습니다.&#x20;

## 제안 목차

### Part I  AI와 Spring AI 개요

1. **AI 트렌드와 핵심 개념**
2. **Spring AI의 탄생 배경과 목표**
3. **개발 환경 설정** — Java 21, Spring Boot 3.3, Spring AI 1.0.0-M6

### Part II  Spring AI 기초 다지기

4. **Spring AI 아키텍처** — Provider, Model, Template, Client, Message
5. **첫 번째 프로젝트: Chat Model 연동** (OpenAI·Anthropic)
6. **Embedding과 벡터 스토어 기본**

### Part III  고급 기능 집중 탐구

7. **Retrieval-Augmented Generation(RAG) 패턴 구현**
8. **Function Calling & `@Tool` 어노테이션**
9. **멀티모달 AI** — 이미지·음성 모델 통합
10. **스트리밍 응답과 비동기 처리**

### Part IV  품질·보안·관측성

11. **AI 애플리케이션 테스트 전략** — Mock LLM, 녹화·재생
12. **데이터 보안·비용 관리**
13. **Observability와 로깅**

## Part V : AI 에이전트 설계와 구현

14. **추론 모델 이해와 설계** 
15. **Model Context Protocol(MCP) 활용 전략**
16. **A2A(Agent-to-Agent) 프로토콜** 
17. **Spring AI Agent API** 
18. **에이전트 체인 & 워크플로 구축**
19. **에이전트 평가·디버깅·모니터링**


### Part VI  실전 프로젝트 케이스스터디

20. **고객 지원 챗봇 구축**
21. **문서 Q\&A 검색 서비스(RAG)**
22. **이미지 생성 웹서비스**
23. **음성 비서 API**

### Part VII  부록

A. **AI 윤리와 책임 있는 AI 개발**
B. **주요 프로바이더 설정 가이드**
C. **Spring AI 커뮤니티 참여와 기여 방법**

[1]: https://spring.io/blog/2025/02/14/spring-ai-1-0-0-m6-released "Spring AI 1.0.0 M6 Released"
[2]: https://galaxy.ai/youtube-summarizer/exploring-the-new-features-of-spring-ai-m6-and-the-mcp-protocol-cE1h-rC2o2U "Exploring the New Features of Spring AI M6 and the mCP Protocol"
[3]: https://www.infoq.com/news/2025/02/spring-news-roundup-feb17-2025/ "Spring News Roundup: Milestone Releases of Boot, Security, Auth Server, Integration ..."
[4]: https://spring.io/projects/spring-ai "Spring AI"
[5]: https://docs.spring.io/spring-ai/reference/api/vectordbs.html "Vector Databases :: Spring AI Reference"
[6]: https://piotrminkowski.com/2025/02/24/using-rag-and-vector-store-with-spring-ai/ "Using RAG and Vector Store with Spring AI - Piotr's TechBlog"
[7]: https://www.baeldung.com/spring-ai-chromadb-vector-store "Spring AI With ChromaDB Vector Store - Baeldung"
