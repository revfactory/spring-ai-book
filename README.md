# Spring AI : 스프링 부트로 만드는 AI 에이전트 애플리케이션

## 서문

Spring AI는 Spring 생태계의 모듈화·POJO 지향 철학을 AI 도메인에 적용하여, 개발자가 최소한의 설정으로 다양한 AI 모델을 교체 가능하게 통합할 수 있는 애플리케이션 프레임워크입니다.

2025년 2월 공개된 1.0.0 Milestone 6(M6) 버전은 `@Tool` / `@ToolParam` 어노테이션과 Model Context Protocol(MCP) SDK, 개선된 VectorStore API를 포함해 최신 기능을 대거 도입했습니다.

Spring AI는 OpenAI·Anthropic·Microsoft Azure·Amazon Bedrock·Google Gemini·Ollama 등 주요 모델 프로바이더를 단일 API로 지원하며, Chat Completion·Embedding·Text-to-Image·Transcription·Text-to-Speech·Moderation과 같은 여러 모델 타입에 대해 동기 및 스트리밍 호출을 제공합니다.

이 책은 Spring AI를 활용하여 실무에서 바로 적용할 수 있는 AI 애플리케이션 개발 방법을 다룹니다. 기초적인 개념부터 고급 기능, 실전 프로젝트까지 단계별로 학습할 수 있도록 구성되었으며, 다양한 예제 코드와 실습 프로젝트를 통해 실무에 필요한 지식을 습득할 수 있습니다.

Java와 Spring Boot의 기본 지식이 있는 개발자라면 누구나 이 책을 통해 최신 AI 기술을 Spring 애플리케이션에 통합하는 방법을 배울 수 있습니다.

## 목차

### [전체 목차 보기](index.md)

### Part I : AI와 Spring AI 개요

- [1. AI 트렌드와 핵심 개념](Part_I_AI_and_Spring_AI_Overview/01_AI_Trends_and_Key_Concepts.md)
- [2. Spring AI의 탄생 배경과 목표](Part_I_AI_and_Spring_AI_Overview/02_Spring_AI_Background_and_Goals.md)
- [3. 개발 환경 설정 — Java 21, Spring Boot 3.3, Spring AI 1.0.0-M6](Part_I_AI_and_Spring_AI_Overview/03_Development_Environment_Setup.md)

### Part II : Spring AI 기초 다지기

- [4. Spring AI 아키텍처 — Provider, Model, Template, Client, Message](Part_II_Spring_AI_Fundamentals/04_Spring_AI_Architecture.md)
- [5. 첫 번째 프로젝트: Chat Model 연동 (OpenAI·Anthropic)](Part_II_Spring_AI_Fundamentals/05_First_Project_Chat_Model_Integration.md)
- 6. Embedding과 벡터 스토어 기본

### Part III : 고급 기능 집중 탐구

- [7. Retrieval-Augmented Generation(RAG) 패턴 구현](Part_III_Advanced_Features/07_RAG_Pattern_Implementation.md)
- [8. Function Calling & `@Tool` 어노테이션](Part_III_Advanced_Features/08_Function_Calling_and_Tool_Annotation.md)
- [9. 멀티모달 AI — 이미지·음성 모델 통합](Part_III_Advanced_Features/09_Multimodal_AI_Integration.md)
- [10. 스트리밍 응답과 비동기 처리](Part_III_Advanced_Features/10_Streaming_Responses_and_Async_Processing.md)

### Part IV : 품질·보안·관측성

- [11. AI 애플리케이션 테스트 전략 — Mock LLM, 녹화·재생](Part_IV_Quality_Security_Observability/11_AI_Application_Testing_Strategy.md)
- [12. 데이터 보안·비용 관리](Part_IV_Quality_Security_Observability/12_Data_Security_Cost_Management.md)
- [13. Observability와 로깅](Part_IV_Quality_Security_Observability/13_Observability_and_Logging.md)

### Part V : AI 에이전트 설계와 구현

- [14. 추론 모델 이해와 설계](Part_V_AI_Agent_Design_and_Implementation/14_AI_Agent_Fundamentals.md)
- [15. Model Context Protocol(MCP) 활용 전략](Part_V_AI_Agent_Design_and_Implementation/15_Building_Agents_with_Spring_AI.md)
- [16. A2A(Agent-to-Agent) 프로토콜](Part_V_AI_Agent_Design_and_Implementation/16_A2A_Agent_to_Agent_Protocol.md)
- [17. Spring AI Agent API](Part_V_AI_Agent_Design_and_Implementation/17_Spring_AI_Agent_API.md)
- [18. 에이전트 체인 & 워크플로 구축](Part_V_AI_Agent_Design_and_Implementation/18_Agent_Chains_and_Workflows.md)
- [19. 에이전트 평가·디버깅·모니터링](Part_V_AI_Agent_Design_and_Implementation/19_Agent_Evaluation_Debugging_Monitoring.md)

### Part VI : 실전 프로젝트 케이스스터디

- [20. 고객 지원 챗봇 구축](Part_VI_Practical_Projects_Case_Studies/20_Customer_Support_Chatbot.md)
- [21. 문서 Q&A 검색 서비스(RAG)](Part_VI_Practical_Projects_Case_Studies/21_Document_QA_Search_Service.md)
- [22. 이미지 생성 웹서비스](Part_VI_Practical_Projects_Case_Studies/22_Image_Generation_Web_Service.md)
- [23. 음성 비서 API](Part_VI_Practical_Projects_Case_Studies/23_Voice_Assistant_API.md)

### Part VII : 부록

- A. AI 윤리와 책임 있는 AI 개발
- B. 주요 프로바이더 설정 가이드
- C. Spring AI 커뮤니티 참여와 기여 방법

## 저자 소개

- **황민호(Robin Hwang)** - Kakao 기술전략

## 사전 준비사항

이 책은 다음과 같은 기본 지식을 갖춘 독자를 대상으로 합니다:

- Java 프로그래밍 기본 지식
- Spring Framework 및 Spring Boot에 대한 기본 이해
- Maven 또는 Gradle 빌드 도구 사용 경험

## 코드 예제

이 저장소에는 책에서 다루는 모든 예제 코드가 포함되어 있습니다. 각 장별로 구성된 폴더에서 관련 코드를 찾을 수 있습니다.

## 업데이트 및 정오표

책 내용에 대한 업데이트와 정오표는 이 저장소를 통해 제공됩니다. 문제점이나 개선사항을 발견하시면 issues를 통해 알려주시기 바랍니다.

## 라이센스

이 책의 내용과 예제 코드는 MIT 라이센스하에 제공됩니다.