# Spring AI 아키텍처

## 개요

Spring AI는 개발자가 다양한 AI 프로바이더와 모델을 일관된 방식으로 사용할 수 있도록 설계된 추상화 계층을 제공합니다. 이 장에서는 Spring AI 1.0.0 GA의 아키텍처와 핵심 구성 요소를 살펴봅니다. 

Spring AI의 핵심 철학은 "엔터프라이즈 데이터 및 API를 AI 모델과 연결"하는 근본적인 과제를 해결하는 것입니다. 이를 위해 Generic Model API, ChatClient, Advisors API 등의 핵심 추상화와 20개 이상의 AI 모델 프로바이더 지원을 제공합니다.

## 아키텍처 개요

Spring AI는 다음과 같은 주요 계층으로 구성된 아키텍처를 가지고 있습니다:

1. **프로바이더 계층 (Provider Layer)**: 각 AI 서비스 공급자와의 통신과 인증을 처리합니다.
2. **모델 계층 (Model Layer)**: 다양한 형태의 AI 모델(채팅, 임베딩, 이미지 생성 등)을 추상화합니다.
3. **템플릿 계층 (Template Layer)**: 프롬프트와 메시지 구성을 위한 템플릿 메커니즘을 제공합니다.
4. **클라이언트 계층 (Client Layer)**: 애플리케이션이 모델과 상호작용하기 위한 고수준 API를 제공합니다.
5. **특수 기능 (Special Features)**: 테스트, 모니터링, 개발 도구를 위한 추가 기능을 제공합니다.

이러한 계층화된 구조는 모델 공급자의 대체 가능성, 확장성, 그리고 애플리케이션 코드와 AI 모델 간의 느슨한 결합을 보장합니다.

## Provider

### Provider 인터페이스

Provider는 특정 AI 서비스 공급자(OpenAI, Anthropic, AWS, Google 등)와의 통신 방법을 추상화합니다. 각 프로바이더는 다음과 같은 기능을 처리합니다:

1. **인증**: API 키 또는 자격 증명 관리
2. **통신**: HTTP 요청 생성, 응답 처리, 오류 처리
3. **보안**: 암호화, 토큰 관리
4. **구성**: 기본 설정 및 옵션 관리

### 주요 Provider 구현체

Spring AI는 다음과 같은 주요 AI 서비스 공급자를 지원합니다:

1. **OpenAIApi**: OpenAI의 API 서비스(GPT-4, DALL-E 등)에 접근
2. **AnthropicApi**: Anthropic의 Claude 모델을 위한 인터페이스
3. **AzureOpenAIApi**: Microsoft Azure의 OpenAI 서비스에 접근
4. **BedrockApi**: Amazon Bedrock의 다양한 모델에 접근 (Claude, Titan, Llama 등)
5. **GeminiApi**: Google의 Gemini 모델에 접근
6. **OllamaApi**: 로컬 호스팅 Ollama 서비스에 접근

각 프로바이더는 특정 API에 맞게 구현되지만, 동일한 인터페이스를 준수하여 애플리케이션 코드가 특정 공급자에 종속되지 않도록 합니다.

## Model

### 모델 타입

Spring AI는 다음과 같은 다양한 유형의 AI 모델을 지원합니다:

1. **Chat Model**: 문맥을 이해하고 대화형 응답을 생성하는 모델 (GPT-4, Claude 등)
2. **Embedding Model**: 텍스트를 고차원 벡터로 변환하는 모델
3. **Image Generation Model**: 텍스트 설명을 기반으로 이미지를 생성하는 모델 (DALL-E, Stable Diffusion 등)
4. **Image Understanding Model**: 이미지를 이해하고 분석하는 모델
5. **Transcription Model**: 음성을 텍스트로 변환하는 모델
6. **Text-to-Speech Model**: 텍스트를 음성으로 변환하는 모델
7. **Moderation Model**: 콘텐츠를 검토하고 안전성을 평가하는 모델

각 모델 타입은 해당 기능에 맞는 고유한 인터페이스를 제공하지만, 일관된 디자인 패턴을 따릅니다.

### 모델 옵션

모델 동작을 커스터마이즈하기 위한 다양한 옵션을 제공합니다:

1. **Temperature**: 응답의 다양성과 창의성 정도 (0.0 ~ 1.0)
2. **Top-p/Nucleus Sampling**: 토큰 선택 시 고려할 확률 분포의 범위
3. **Max Tokens**: 생성할 최대 토큰 수
4. **Frequency/Presence Penalty**: 반복 패턴을 제어하기 위한 패널티
5. **Model-Specific Options**: 특정 모델에만 적용되는 고유한 옵션

이러한 옵션은 일반적으로 각 모델의 `Options` 클래스를 통해 구성되며, 빌더 패턴을 통해 쉽게 사용할 수 있습니다.

## Generic Model API

### 핵심 인터페이스

Spring AI의 모든 AI 모델은 Generic Model API를 기반으로 구축되어 일관성 있는 패턴을 제공합니다:

```java
public interface Model<TReq extends ModelRequest<?>, TRes extends ModelResponse<?>> {
    TRes call(TReq request);
}

public interface StreamingModel<TReq extends ModelRequest<?>, TResChunk extends ModelResponse<?>> {
    Flux<TResChunk> stream(TReq request);
}
```

이러한 제네릭 인터페이스는 다양한 AI 모델 구현체들이 공통된 계약을 따르도록 보장합니다.

## Template

### 프롬프트 템플릿

Spring AI는 AI 모델에 전송되는 프롬프트를 정의하고 관리하기 위한 템플릿 시스템을 제공합니다. 이는 StringTemplate 기반의 템플릿 엔진을 활용하여 다양한 프롬프트 패턴을 정의할 수 있습니다.

```java
// 시스템 프롬프트 템플릿 예시
SystemPromptTemplate systemPromptTemplate = 
    new SystemPromptTemplate("You are a helpful assistant specialized in {domain}.");

// 매개변수 전달
Prompt prompt = new Prompt(
    systemPromptTemplate.create(Map.of("domain", "Java programming")),
    new UserPrompt("How do I create a Spring Boot application?")
);
```

### 메시지 구조

Chat 모델과의 상호작용을 위한 메시지 구조는 다음과 같은 유형으로 구분됩니다:

1. **SystemMessage**: AI 시스템의 동작을 정의하는 가이드라인
2. **UserMessage**: 사용자가 AI에게 전달하는 메시지
3. **AssistantMessage**: AI가 사용자에게 응답하는 메시지
4. **FunctionMessage**: 함수 호출 결과를 포함하는 메시지

이러한 메시지를 조합하여 일관된 대화 흐름을 만들 수 있습니다.

## Client

### 클라이언트 인터페이스

Spring AI는 다음과 같은 고수준 클라이언트 인터페이스를 제공합니다:

1. **ChatClient**: 채팅 모델과 상호작용하기 위한 인터페이스
2. **EmbeddingClient**: 텍스트 임베딩을 생성하기 위한 인터페이스
3. **ImageClient**: 이미지 생성 및 이해를 위한 인터페이스
4. **AudioClient**: 음성 처리를 위한 인터페이스
5. **ModerationClient**: 콘텐츠 검토를 위한 인터페이스

이러한 클라이언트는 애플리케이션 코드와 모델 사이의 주요 상호작용 지점을 제공합니다.

### 동기 및 비동기 호출

Spring AI는 동기 및 비동기 호출 방식을 모두 지원합니다:

```java
// 동기 호출
ChatResponse response = chatClient.call(prompt);

// 비동기 호출
Mono<ChatResponse> responseMono = chatClient.callReactive(prompt);

// 스트리밍 호출
Flux<ChatResponse> responseStream = chatClient.stream(prompt);
```

스트리밍 호출은 모델이 응답을 생성함에 따라 즉시 부분적인 결과를 받을 수 있어, 실시간 응답이 필요한 경우에 유용합니다.

## 특수 기능

### Function Calling

Spring AI는 AI 모델이 호출할 수 있는 함수를 정의하여 복잡한 태스크를 수행할 수 있도록 지원합니다. 특히 Spring AI 1.0.0-M6에서 도입된 `@Tool` 및 `@ToolParam` 어노테이션을 통해 Java 메서드를 쉽게 함수로 노출할 수 있습니다.

```java
public class WeatherService {
    
    @Tool("Get current weather for a location")
    public WeatherInfo getWeather(
        @ToolParam("The city name") String city,
        @ToolParam("The country code") String countryCode
    ) {
        // 날씨 정보 조회 및 반환
        return weatherAPI.getWeatherInfo(city, countryCode);
    }
}
```

### Model Context Protocol (MCP)

MCP는 AI 에이전트가 사용할 수 있는 도구를 정의하고 통합하기 위한 표준화된 프로토콜입니다. 이를 통해 AI 에이전트가 다양한 도구를 활용해 복잡한 문제를 해결할 수 있도록 합니다.

### 테스트 지원

Spring AI는 AI 모델 호출을 포함한 코드의 테스트를 위한 강력한 도구를 제공합니다:

1. **MockChatClient**: AI 모델의 응답을 모의하기 위한 클라이언트
2. **녹화/재생 메커니즘**: AI 모델 호출을 녹화하고 재생하여 테스트의 재현성을 높이고 비용을 절감
3. **테스트 유틸리티**: 프롬프트와 응답을 쉽게 검증하기 위한 유틸리티
4. **AI 모델 평가**: 생성된 콘텐츠를 평가하고 환각 반응으로부터 보호하는 유틸리티

이러한 테스트 도구는 AI 기반 애플리케이션의 안정성과 테스트 가능성을 크게 향상시킵니다.

### 관찰가능성 (Observability)

Spring AI는 AI 관련 작업에 대한 통찰력을 제공하는 관찰가능성 기능을 내장하고 있습니다:

1. **메트릭 수집**: AI 모델 호출에 대한 성능 메트릭
2. **추적(Tracing)**: 요청 흐름을 추적하여 문제 진단
3. **로깅**: 구조화된 로그를 통한 디버깅 지원
4. **Advisor 체인 모니터링**: Advisor 실행 과정 추적

### Chat Memory

Spring AI는 대화형 AI 애플리케이션을 위한 메모리 관리 기능을 제공합니다:

1. **MessageWindowChatMemory**: 지정된 크기의 메시지 윈도우 유지
2. **ChatMemoryRepository**: 다양한 저장소 구현 (InMemory, JDBC, Cassandra, Neo4j)
3. **VectorStoreChatMemoryAdvisor**: 벡터 스토어를 활용한 효율적인 메모리 검색

### 구조화된 출력 (Structured Output)

AI 모델의 출력을 Java 객체로 자동 매핑하는 기능을 제공합니다:

```java
record ActorFilms(String actor, List<String> movies) {}

ActorFilms actorFilms = chatClient.prompt()
    .user("Generate the filmography for Tom Hanks.")
    .call()
    .entity(ActorFilms.class);
```

### ETL 파이프라인

문서 수집과 처리를 위한 ETL(Extract, Transform, Load) 프레임워크를 제공합니다:

1. **문서 로더**: 다양한 소스에서 문서 로드 (PDF, TXT, JSON 등)
2. **문서 변환기**: 문서를 청크로 분할하고 메타데이터 추가
3. **문서 작성기**: 벡터 스토어에 문서 저장
4. **파이프라인 조합**: 로더, 변환기, 작성기를 연결하여 ETL 파이프라인 구성

## 스프링 부트 자동 구성

Spring AI는 Spring Boot의 자동 구성 기능을 활용하여 최소한의 설정만으로 AI 서비스를 빠르게 설정할 수 있습니다.

### 스타터 패키지

Spring AI는 다음과 같은 스타터 패키지를 제공합니다:

1. **spring-ai-openai-spring-boot-starter**: OpenAI 모델 통합
2. **spring-ai-anthropic-spring-boot-starter**: Anthropic Claude 모델 통합
3. **spring-ai-gemini-spring-boot-starter**: Google Gemini 모델 통합
4. **spring-ai-bedrock-spring-boot-starter**: Amazon Bedrock 모델 통합
5. **spring-ai-azure-openai-spring-boot-starter**: Azure OpenAI 모델 통합
6. **spring-ai-ollama-spring-boot-starter**: Ollama 로컬 모델 통합

각 스타터 패키지는 해당 프로바이더에 필요한 모든 의존성과 구성 클래스를 포함합니다.

### 속성 구성

애플리케이션의 `application.properties` 또는 `application.yml` 파일을 통해 Spring AI 기능을 구성할 수 있습니다.

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-3-opus-20240229
          temperature: 0.7
```

## 확장 방법

### 사용자 정의 프로바이더

Spring AI는 사용자가 자신의 사용자 정의 프로바이더를 구현할 수 있도록 해당 인터페이스를 노출합니다. 예를 들어, 다음과 같이 사용자 정의 채팅 프로바이더를 구현할 수 있습니다:

```java
@Component
public class CustomChatClient implements ChatClient {
    @Override
    public ChatResponse call(Prompt prompt) {
        // 사용자 정의 로직 구현
    }

    @Override
    public Flux<ChatResponse> stream(Prompt prompt) {
        // 스트리밍 로직 구현
    }
}
```

### 컨버터 및 가이드

Spring AI는 다양한 형식의 데이터를 변환하고 가공하기 위한 컨버터와 가이드를 제공합니다. 사용자는 이러한 컨버터를 사용하거나 자신의 컨버터를 구현할 수 있습니다.

## 결론

이 장에서는 Spring AI의 아키텍처와 핵심 구성 요소를 살펴보았습니다. Provider, Model, Template, Client, Message 등의 개념을 통해 Spring AI가 어떻게 다양한 AI 모델과 프로바이더를 추상화하고 일관된 인터페이스를 제공하는지 이해했습니다.

아키텍처에 대한 이해를 바탕으로, 다음 장에서는 첫 번째 Spring AI 프로젝트를 구현하며 실제로 OpenAI와 Anthropic과 같은 채팅 모델을 연동하는 방법을 살펴보겠습니다.