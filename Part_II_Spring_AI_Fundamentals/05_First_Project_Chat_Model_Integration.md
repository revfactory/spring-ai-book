# 첫 번째 프로젝트: Chat Model 연동

## 개요

이 장에서는 Spring AI를 사용하여 첫 번째 프로젝트를 구현해보겠습니다. OpenAI와 Anthropic의 고급 채팅 모델을 Spring 애플리케이션에 통합하는 전 과정을 단계별로 살펴보고, 실제 사용 사례를 통해 Spring AI의 핵심 개념을 적용해봅니다.

## 준비 사항

이 프로젝트를 진행하기 위해 다음과 같은 준비가 필요합니다:

1. Java 17 이상 (Java 21 권장)
2. Spring Boot 3.4.x 지원 개발 환경
3. IDE (IntelliJ IDEA, Eclipse, VS Code 등)
4. OpenAI API 키 (해당 프로바이더 사용 시)
5. Anthropic API 키 (해당 프로바이더 사용 시)

이전 장의 개발 환경 설정 가이드를 참조하여 필요한 소프트웨어와 API 키를 준비해주세요.

## 프로젝트 생성

### Spring Boot 프로젝트 생성

첫 번째 스텝은 Spring Boot 프로젝트를 생성하는 것입니다. Spring Initializr(https://start.spring.io/)를 사용하여 다음 옵션으로 프로젝트를 생성합니다:

- **프로젝트**: Gradle - Groovy / Maven 중 선택
- **언어**: Java
- **Spring Boot**: 3.4.x
- **그룹**: com.example
- **아티팩트**: spring-ai-chat
- **이름**: spring-ai-chat
- **설명**: Spring AI Chat Demo
- **패키지 이름**: com.example.springaichat
- **패키징**: Jar
- **Java**: 17 이상 (21 권장)

의존성을 추가합니다:
- Spring Web
- Spring Boot DevTools  
- AI Models에서 원하는 모델 선택:
  - OpenAI
  - Anthropic
- Lombok (선택사항)

### Spring AI 의존성 추가

생성된 프로젝트에 Spring AI 관련 의존성을 추가합니다. 이 예제에서는 OpenAI와 Anthropic 모델을 모두 사용하겠습니다.

#### Gradle (build.gradle)

```gradle
repositories {
    mavenCentral()
}

dependencies {
    implementation platform('org.springframework.ai:spring-ai-bom:1.0.0')
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
    implementation 'org.springframework.ai:spring-ai-anthropic-spring-boot-starter'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.ai:spring-ai-test'
}
```

#### Maven (pom.xml)

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

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 환경 설정

`src/main/resources/application.yml` 파일을 생성하여 다음과 같이 API 키와 모델 설정을 구성합니다:

```yaml
spring:
  ai:
    # 다중 모델 사용 시 ChatClient.Builder 자동 구성 비활성화
    chat:
      client:
        enabled: false
    
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
          model: claude-3-5-sonnet-20241022
          temperature: 0.7

logging:
  level:
    org.springframework.ai: INFO
```

환경 변수를 설정하여 API 키를 제공합니다:

```bash
# Windows
setx OPENAI_API_KEY "your-openai-api-key"
setx ANTHROPIC_API_KEY "your-anthropic-api-key"

# macOS/Linux
export OPENAI_API_KEY="your-openai-api-key"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
```

## 기본 채팅 구현

### 데이터 모델 정의

먼저 사용자 입력과 AI 응답을 처리하기 위한 데이터 모델을 정의합니다:

```java
package com.example.springaichat.model;

public class ChatRequest {
    private String message;
    private String provider; // "openai" 또는 "anthropic"

    // getter, setter, constructor
    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public String getProvider() {
        return provider;
    }

    public void setProvider(String provider) {
        this.provider = provider;
    }
}
```

```java
package com.example.springaichat.model;

public class ChatResponse {
    private String response;
    private String provider;
    private long processingTimeMs;

    // getter, setter, constructor
    public String getResponse() {
        return response;
    }

    public void setResponse(String response) {
        this.response = response;
    }

    public String getProvider() {
        return provider;
    }

    public void setProvider(String provider) {
        this.provider = provider;
    }

    public long getProcessingTimeMs() {
        return processingTimeMs;
    }

    public void setProcessingTimeMs(long processingTimeMs) {
        this.processingTimeMs = processingTimeMs;
    }

    public ChatResponse() {}

    public ChatResponse(String response, String provider, long processingTimeMs) {
        this.response = response;
        this.provider = provider;
        this.processingTimeMs = processingTimeMs;
    }
}
```

### 채팅 서비스 구현

이제 서비스 레이어를 구현해봅니다. 이 서비스는 OpenAI와 Anthropic 모델을 모두 사용할 수 있도록 설계됩니다:

```java
package com.example.springaichat.service;

import com.example.springaichat.model.ChatResponse;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.SystemPromptTemplate;
import org.springframework.ai.chat.prompt.UserPrompt;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;
import java.util.Map;

@Service
public class ChatService {

    private final ChatClient openAiChatClient;
    private final ChatClient anthropicChatClient;

    public ChatService(
            @Qualifier("openAiChatClient") ChatClient openAiChatClient,
            @Qualifier("anthropicChatClient") ChatClient anthropicChatClient) {
        this.openAiChatClient = openAiChatClient;
        this.anthropicChatClient = anthropicChatClient;
    }

    public ChatResponse chat(String message, String provider) {
        SystemPromptTemplate systemPromptTemplate = 
            new SystemPromptTemplate(
                "You are a helpful assistant specialized in {domain}. " +
                "Your name is {name} and you were created by {company}.");
        
        Map<String, Object> parameters = Map.of(
            "domain", "Spring Boot and AI",
            "name", provider.equals("openai") ? "GPT Assistant" : "Claude Assistant",
            "company", provider.equals("openai") ? "OpenAI" : "Anthropic"
        );
        
        Prompt prompt = new Prompt(
            systemPromptTemplate.create(parameters),
            new UserPrompt(message)
        );
        
        ChatClient selectedClient = 
            provider.equals("openai") ? openAiChatClient : anthropicChatClient;
        
        Instant start = Instant.now();
        org.springframework.ai.chat.ChatResponse aiResponse = selectedClient.call(prompt);
        Instant end = Instant.now();
        
        long processingTime = Duration.between(start, end).toMillis();
        String content = aiResponse.getResult().getOutput().getContent();
        
        return new ChatResponse(content, provider, processingTime);
    }
}
```

### 채팅 컨트롤러 구현

사용자 요청을 처리할 REST 컨트롤러를 구현합니다:

```java
package com.example.springaichat.controller;

import com.example.springaichat.model.ChatRequest;
import com.example.springaichat.model.ChatResponse;
import com.example.springaichat.service.ChatService;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatService chatService;

    public ChatController(ChatService chatService) {
        this.chatService = chatService;
    }

    @PostMapping
    public ChatResponse chat(@RequestBody ChatRequest request) {
        return chatService.chat(request.getMessage(), request.getProvider());
    }
}
```

### 애플리케이션 실행

이제 애플리케이션을 실행할 준비가 되었습니다. 다음 명령어로 애플리케이션을 시작합니다:

```bash
# Gradle
./gradlew bootRun

# Maven
./mvnw spring-boot:run
```

애플리케이션이 실행되면, 다음과 같이 curl이나 Postman을 사용해 API를 테스트해볼 수 있습니다:

```bash
curl -X POST http://localhost:8080/api/chat \
-H "Content-Type: application/json" \
-d '{"message":"What are the benefits of using Spring AI?", "provider":"openai"}'
```

## Advisors를 활용한 고급 기능

Spring AI 1.0.0 GA의 Advisors API를 사용하여 AI 상호작용을 향상시킬 수 있습니다. Advisors는 프롬프트를 수정하고, 응답을 변환하며, 다양한 패턴을 구현할 수 있습니다.

### SimpleLoggerAdvisor 사용

요청과 응답을 로깅하는 간단한 Advisor를 추가해봅시다:

```java
package com.example.springaichat.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.SimpleLoggerAdvisor;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class ChatServiceWithLogging {

    private final ChatClient chatClient;

    public ChatServiceWithLogging(@Qualifier("openAiChatModel") ChatModel chatModel) {
        this.chatClient = ChatClient.builder(chatModel)
            .defaultAdvisors(new SimpleLoggerAdvisor())
            .build();
    }

    public String chat(String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

application.yml에 로깅 레벨을 추가합니다:

```yaml
logging:
  level:
    org.springframework.ai.chat.client.advisor: DEBUG
```

## 프롬프트 템플릿 개선

이제 더 복잡한 프롬프트 템플릿을 사용하여 응답을 구조화해보겠습니다. 템플릿 파일을 외부에서 관리하고 로드하는 방법을 살펴봅니다.

### 템플릿 파일 생성

`src/main/resources/prompts` 디렉토리를 생성하고 `system-message.st` 파일을 추가합니다:

```
You are a specialized AI assistant focused on {domain} technology.

Your name is {name} and you were created by {company}.

When responding, please follow these guidelines:
1. Be concise and to the point with your answers
2. When appropriate, use bullet points for clarity
3. If you mention code, provide brief examples in Java using Spring Boot 3.x syntax
4. If you're unsure about something, acknowledge the uncertainty
5. Always include one helpful resource or documentation link at the end of your response

Response format should be in Markdown to ensure good readability.
```

### 프롬프트 템플릿 로드 및 사용

서비스 코드를 업데이트하여 외부 템플릿 파일을 사용하도록 변경합니다:

```java
package com.example.springaichat.service;

import com.example.springaichat.model.ChatResponse;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.SystemPromptTemplate;
import org.springframework.ai.chat.prompt.UserPrompt;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.core.io.Resource;
import org.springframework.core.io.ResourceLoader;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;

@Service
public class ChatService {

    private final ChatClient openAiChatClient;
    private final ChatClient anthropicChatClient;
    private final ResourceLoader resourceLoader;

    public ChatService(
            @Qualifier("openAiChatClient") ChatClient openAiChatClient,
            @Qualifier("anthropicChatClient") ChatClient anthropicChatClient,
            ResourceLoader resourceLoader) {
        this.openAiChatClient = openAiChatClient;
        this.anthropicChatClient = anthropicChatClient;
        this.resourceLoader = resourceLoader;
    }

    public ChatResponse chat(String message, String provider) throws IOException {
        Resource promptResource = resourceLoader.getResource("classpath:prompts/system-message.st");
        String promptTemplate = new String(promptResource.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
        
        SystemPromptTemplate systemPromptTemplate = new SystemPromptTemplate(promptTemplate);
        
        Map<String, Object> parameters = Map.of(
            "domain", "Spring Boot and AI",
            "name", provider.equals("openai") ? "GPT Assistant" : "Claude Assistant",
            "company", provider.equals("openai") ? "OpenAI" : "Anthropic"
        );
        
        Prompt prompt = new Prompt(
            systemPromptTemplate.create(parameters),
            new UserPrompt(message)
        );
        
        ChatClient selectedClient = 
            provider.equals("openai") ? openAiChatClient : anthropicChatClient;
        
        Instant start = Instant.now();
        org.springframework.ai.chat.ChatResponse aiResponse = selectedClient.call(prompt);
        Instant end = Instant.now();
        
        long processingTime = Duration.between(start, end).toMillis();
        String content = aiResponse.getResult().getOutput().getContent();
        
        return new ChatResponse(content, provider, processingTime);
    }
}
```

이제 컨트롤러도 IOException을 처리하도록 업데이트합니다:

```java
package com.example.springaichat.controller;

import com.example.springaichat.model.ChatRequest;
import com.example.springaichat.model.ChatResponse;
import com.example.springaichat.service.ChatService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatService chatService;

    public ChatController(ChatService chatService) {
        this.chatService = chatService;
    }

    @PostMapping
    public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request) {
        try {
            ChatResponse response = chatService.chat(request.getMessage(), request.getProvider());
            return ResponseEntity.ok(response);
        } catch (IOException e) {
            return ResponseEntity.internalServerError().build();
        }
    }
}
```

## 메시지 히스토리 구현

이번에는 Spring AI 1.0.0 GA의 Chat Memory 기능을 활용하여 대화 기록을 유지하는 기능을 추가해봅니다. 대화 기록을 유지하면 AI가 이전 문맥을 기억하고 연속적인 대화를 할 수 있습니다.

### Chat Memory 설정

Spring AI는 MessageWindowChatMemory를 통해 대화 기록을 관리합니다:

```java
package com.example.springaichat.config;

import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.chat.memory.InMemoryChatMemoryRepository;
import org.springframework.ai.chat.memory.MessageWindowChatMemory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ChatMemoryConfig {
    
    @Bean
    public ChatMemory chatMemory() {
        return new MessageWindowChatMemory(
            new InMemoryChatMemoryRepository(),
            20  // 최대 20개의 메시지 저장
        );
    }
}
```

### 대화 세션 모델 추가

```java
package com.example.springaichat.model;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

public class ChatSession {
    private String sessionId;
    private String provider;
    private List<ChatMessage> messages;

    public ChatSession(String provider) {
        this.sessionId = UUID.randomUUID().toString();
        this.provider = provider;
        this.messages = new ArrayList<>();
    }

    public void addMessage(String role, String content) {
        messages.add(new ChatMessage(role, content));
    }

    // getter, setter
    public String getSessionId() {
        return sessionId;
    }

    public String getProvider() {
        return provider;
    }

    public List<ChatMessage> getMessages() {
        return messages;
    }
}
```

```java
package com.example.springaichat.model;

public class ChatMessage {
    private String role; // "user" 또는 "assistant"
    private String content;

    public ChatMessage(String role, String content) {
        this.role = role;
        this.content = content;
    }

    // getter, setter
    public String getRole() {
        return role;
    }

    public String getContent() {
        return content;
    }
}
```

### 세션 관리 서비스 구현

```java
package com.example.springaichat.service;

import com.example.springaichat.model.ChatMessage;
import com.example.springaichat.model.ChatSession;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class SessionService {

    private final Map<String, ChatSession> sessions = new ConcurrentHashMap<>();

    public ChatSession getOrCreateSession(String sessionId, String provider) {
        return sessions.computeIfAbsent(sessionId, k -> new ChatSession(provider));
    }

    public void addMessageToSession(String sessionId, String role, String content) {
        ChatSession session = sessions.get(sessionId);
        if (session != null) {
            session.addMessage(role, content);
        }
    }

    public ChatSession getSession(String sessionId) {
        return sessions.get(sessionId);
    }
}
```

### 서비스 및 API 업데이트

ChatMemory를 활용하여 채팅 서비스를 업데이트합니다:

```java
package com.example.springaichat.service;

import com.example.springaichat.model.ChatResponse;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;

import static org.springframework.ai.chat.client.advisor.AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY;

@Service
public class ChatService {

    private final ChatClient openAiChatClient;
    private final ChatClient anthropicChatClient;
    private final ChatMemory chatMemory;

    public ChatService(
            @Qualifier("openAiChatModel") ChatModel openAiChatModel,
            @Qualifier("anthropicChatModel") ChatModel anthropicChatModel,
            ChatMemory chatMemory) {
        
        this.chatMemory = chatMemory;
        
        // MessageChatMemoryAdvisor를 사용하여 ChatClient 생성
        this.openAiChatClient = ChatClient.builder(openAiChatModel)
            .defaultSystem("You are a helpful assistant specialized in {domain}. " +
                          "Your name is {name} and you were created by {company}.")
            .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
            .build();
            
        this.anthropicChatClient = ChatClient.builder(anthropicChatModel)
            .defaultSystem("You are a helpful assistant specialized in {domain}. " +
                          "Your name is {name} and you were created by {company}.")
            .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
            .build();
    }

    public ChatResponse chat(String message, String provider, String conversationId) {
        ChatClient selectedClient = 
            provider.equals("openai") ? openAiChatClient : anthropicChatClient;
            
        String name = provider.equals("openai") ? "GPT Assistant" : "Claude Assistant";
        String company = provider.equals("openai") ? "OpenAI" : "Anthropic";
        
        Instant start = Instant.now();
        
        // Advisor를 통해 대화 기록을 자동으로 관리
        String content = selectedClient.prompt()
            .system(s -> s
                .param("domain", "Spring Boot and AI")
                .param("name", name)
                .param("company", company))
            .user(message)
            .advisors(a -> a.param(CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId))
            .call()
            .content();
            
        Instant end = Instant.now();
        long processingTime = Duration.between(start, end).toMillis();
        
        return new ChatResponse(content, provider, processingTime);
    }
}
```

채팅 요청 모델과 컨트롤러도 세션 ID를 처리하도록 업데이트합니다:

```java
package com.example.springaichat.model;

public class ChatRequest {
    private String message;
    private String provider;
    private String sessionId;

    // getter, setter, constructor
    // ... (기존 코드)

    public String getSessionId() {
        return sessionId;
    }

    public void setSessionId(String sessionId) {
        this.sessionId = sessionId;
    }
}
```

```java
package com.example.springaichat.controller;

import com.example.springaichat.model.ChatRequest;
import com.example.springaichat.model.ChatResponse;
import com.example.springaichat.model.ChatSession;
import com.example.springaichat.service.ChatService;
import com.example.springaichat.service.SessionService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;
import java.util.UUID;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatService chatService;
    private final SessionService sessionService;

    public ChatController(ChatService chatService, SessionService sessionService) {
        this.chatService = chatService;
        this.sessionService = sessionService;
    }

    @PostMapping
    public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request) {
        try {
            String sessionId = request.getSessionId();
            if (sessionId == null || sessionId.isEmpty()) {
                sessionId = UUID.randomUUID().toString();
            }
            
            ChatResponse response = chatService.chat(
                request.getMessage(), 
                request.getProvider(), 
                sessionId
            );
            
            return ResponseEntity.ok(response);
        } catch (IOException e) {
            return ResponseEntity.internalServerError().build();
        }
    }

    @GetMapping("/sessions/{sessionId}")
    public ResponseEntity<ChatSession> getSession(@PathVariable String sessionId) {
        ChatSession session = sessionService.getSession(sessionId);
        if (session != null) {
            return ResponseEntity.ok(session);
        } else {
            return ResponseEntity.notFound().build();
        }
    }
}
```

## 스트리밍 응답 구현

이제 스트맍 응답을 구현해보겠습니다. Spring AI 1.0.0 GA는 기본적으로 스트리밍을 지원하므로 추가 의존성 없이 구현할 수 있습니다. 단, 스트리밍 기능을 사용하려면 Spring WebFlux가 필요합니다:

추가 의존성을 추가합니다:

```gradle
// Gradle (build.gradle)
implementation 'org.springframework.boot:spring-boot-starter-webflux'
```

```xml
<!-- Maven (pom.xml) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

스트림 응답을 위한 데이터 모델을 정의합니다:

```java
package com.example.springaichat.model;

public class StreamResponse {
    private String content;
    private boolean done;

    public StreamResponse() {
    }

    public StreamResponse(String content, boolean done) {
        this.content = content;
        this.done = done;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public boolean isDone() {
        return done;
    }

    public void setDone(boolean done) {
        this.done = done;
    }
}
```

채팅 서비스에 스트리밍 메서드를 추가합니다:

```java
// ChatService 클래스에 아래 메서드 추가
public Flux<StreamResponse> streamChat(String message, String provider, String sessionId) {
    ChatSession session = sessionService.getOrCreateSession(sessionId, provider);
    
    ChatClient selectedClient = 
        provider.equals("openai") ? openAiChatClient : anthropicChatClient;
        
    String name = provider.equals("openai") ? "GPT Assistant" : "Claude Assistant";
    String company = provider.equals("openai") ? "OpenAI" : "Anthropic";
    
    // 사용자 메시지 기록
    sessionService.addMessageToSession(sessionId, "user", message);
    
    StringBuilder fullResponseBuilder = new StringBuilder();
    
    // Spring AI 1.0.0 GA의 새로운 스트리밍 API 사용
    return selectedClient.prompt()
        .system(s -> s
            .param("domain", "Spring Boot and AI")
            .param("name", name)
            .param("company", company))
        .user(message)
        .stream()
        .content()
        .map(content -> {
            fullResponseBuilder.append(content);
            return new StreamResponse(content, false);
        })
        .concatWith(Mono.fromCallable(() -> {
            // 마지막에 전체 응답을 세션에 저장
            String fullResponse = fullResponseBuilder.toString();
            sessionService.addMessageToSession(sessionId, "assistant", fullResponse);
            return new StreamResponse("", true);
        }));
}
```

컨트롤러에 스트리밍 엔드포인트를 추가합니다:

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<StreamResponse> streamChat(
        @RequestParam String message,
        @RequestParam String provider,
        @RequestParam(required = false) String sessionId) {
    try {
        if (sessionId == null || sessionId.isEmpty()) {
            sessionId = UUID.randomUUID().toString();
        }
        
        return chatService.streamChat(message, provider, sessionId);
    } catch (IOException e) {
        return Flux.error(e);
    }
}
```

## 사용자 인터페이스 구현

이번에는 기본적인 HTML, CSS, JavaScript를 사용하여 사용자 인터페이스를 구현해보겠습니다.

### 정적 리소스 추가

`src/main/resources/static` 폴더에 다음 파일들을 생성합니다:

**index.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Spring AI Chat Demo</title>
    <link rel="stylesheet" href="/css/style.css">
</head>
<body>
    <div class="chat-container">
        <div class="header">
            <h1>Spring AI Chat</h1>
            <div class="provider-selector">
                <label>
                    <input type="radio" name="provider" value="openai" checked> OpenAI
                </label>
                <label>
                    <input type="radio" name="provider" value="anthropic"> Anthropic
                </label>
            </div>
        </div>
        
        <div class="chat-messages" id="chatMessages"></div>
        
        <div class="input-area">
            <textarea id="userInput" placeholder="Type your message here..."></textarea>
            <button id="sendButton">Send</button>
        </div>
    </div>
    
    <script src="/js/app.js"></script>
</body>
</html>
```

**/css/style.css**

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: Arial, sans-serif;
}

body {
    background-color: #f5f5f5;
    height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
}

.chat-container {
    width: 90%;
    max-width: 800px;
    height: 80vh;
    background-color: white;
    border-radius: 10px;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
    display: flex;
    flex-direction: column;
    overflow: hidden;
}

.header {
    padding: 20px;
    background-color: #6200ee;
    color: white;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.provider-selector {
    display: flex;
    gap: 15px;
}

.chat-messages {
    flex: 1;
    padding: 20px;
    overflow-y: auto;
}

.message {
    margin-bottom: 15px;
    padding: 10px 15px;
    border-radius: 5px;
    max-width: 80%;
    position: relative;
}

.user-message {
    background-color: #e1f5fe;
    margin-left: auto;
    border-bottom-right-radius: 0;
}

.assistant-message {
    background-color: #f1f1f1;
    margin-right: auto;
    border-bottom-left-radius: 0;
}

.message-content {
    word-wrap: break-word;
}

.input-area {
    padding: 20px;
    display: flex;
    gap: 10px;
    background-color: #f9f9f9;
    border-top: 1px solid #ddd;
}

textarea {
    flex: 1;
    padding: 10px;
    border: 1px solid #ddd;
    border-radius: 5px;
    resize: none;
    height: 60px;
}

button {
    padding: 10px 20px;
    background-color: #6200ee;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
}

button:hover {
    background-color: #3700b3;
}

.typing-indicator {
    padding: 10px 15px;
    background-color: #f1f1f1;
    border-radius: 5px;
    margin-right: auto;
    margin-bottom: 15px;
    display: none;
}

.typing-indicator span {
    height: 10px;
    width: 10px;
    float: left;
    margin: 0 1px;
    background-color: #9E9E9E;
    display: block;
    border-radius: 50%;
    opacity: 0.4;
}

.typing-indicator span:nth-of-type(1) {
    animation: 1s blink infinite 0.3333s;
}

.typing-indicator span:nth-of-type(2) {
    animation: 1s blink infinite 0.6666s;
}

.typing-indicator span:nth-of-type(3) {
    animation: 1s blink infinite 0.9999s;
}

@keyframes blink {
    50% {
        opacity: 1;
    }
}
```

**/js/app.js**

```javascript
document.addEventListener('DOMContentLoaded', function() {
    const chatMessages = document.getElementById('chatMessages');
    const userInput = document.getElementById('userInput');
    const sendButton = document.getElementById('sendButton');
    const providerRadios = document.getElementsByName('provider');
    
    let sessionId = null;
    let typing = false;
    
    function getSelectedProvider() {
        for (const radio of providerRadios) {
            if (radio.checked) {
                return radio.value;
            }
        }
        return 'openai'; // default
    }
    
    function addMessage(content, isUser) {
        const messageDiv = document.createElement('div');
        messageDiv.className = `message ${isUser ? 'user-message' : 'assistant-message'}`;
        
        const contentDiv = document.createElement('div');
        contentDiv.className = 'message-content';
        
        if (!isUser) {
            // 마크다운 문법 처리를 위한 외부 라이브러리 사용
            import('https://cdn.jsdelivr.net/npm/marked/marked.min.js')
                .then(() => {
                    contentDiv.innerHTML = marked.parse(content);
                })
                .catch(() => {
                    contentDiv.textContent = content;
                });
        } else {
            contentDiv.textContent = content;
        }
        
        messageDiv.appendChild(contentDiv);
        chatMessages.appendChild(messageDiv);
        chatMessages.scrollTop = chatMessages.scrollHeight;
    }
    
    function addTypingIndicator() {
        if (typing) return;
        typing = true;
        
        const indicator = document.createElement('div');
        indicator.className = 'typing-indicator';
        indicator.id = 'typingIndicator';
        
        for (let i = 0; i < 3; i++) {
            const dot = document.createElement('span');
            indicator.appendChild(dot);
        }
        
        chatMessages.appendChild(indicator);
        indicator.style.display = 'block';
        chatMessages.scrollTop = chatMessages.scrollHeight;
    }
    
    function removeTypingIndicator() {
        typing = false;
        const indicator = document.getElementById('typingIndicator');
        if (indicator) {
            indicator.remove();
        }
    }
    
    function sendMessage() {
        const message = userInput.value.trim();
        if (!message) return;
        
        addMessage(message, true);
        userInput.value = '';
        
        const provider = getSelectedProvider();
        
        // 스트리밍 API 사용
        addTypingIndicator();
        
        let fullResponse = '';
        const eventSource = new EventSource(`/api/chat/stream?message=${encodeURIComponent(message)}&provider=${provider}&sessionId=${sessionId || ''}`);
        
        eventSource.onmessage = function(event) {
            const data = JSON.parse(event.data);
            
            if (data.done) {
                eventSource.close();
                removeTypingIndicator();
                return;
            }
            
            fullResponse += data.content;
            
            // 응답 업데이트
            const existingMessage = document.querySelector('.assistant-message:last-child');
            if (existingMessage) {
                const contentDiv = existingMessage.querySelector('.message-content');
                import('https://cdn.jsdelivr.net/npm/marked/marked.min.js')
                    .then(() => {
                        contentDiv.innerHTML = marked.parse(fullResponse);
                    })
                    .catch(() => {
                        contentDiv.textContent = fullResponse;
                    });
            } else {
                removeTypingIndicator();
                addMessage(fullResponse, false);
            }
            
            chatMessages.scrollTop = chatMessages.scrollHeight;
        };
        
        eventSource.onerror = function() {
            eventSource.close();
            removeTypingIndicator();
            if (!fullResponse) {
                addMessage("Sorry, there was an error processing your request.", false);
            }
        };
    }
    
    sendButton.addEventListener('click', sendMessage);
    
    userInput.addEventListener('keydown', function(e) {
        if (e.key === 'Enter' && !e.shiftKey) {
            e.preventDefault();
            sendMessage();
        }
    });
});
```

### 컨트롤러 업데이트

정적 파일을 제공하기 위한 컨트롤러를 추가합니다:

```java
package com.example.springaichat.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class WebController {

    @GetMapping("/")
    public String home() {
        return "index";
    }
}
```

## 결론

이 장에서는 Spring AI 1.0.0 GA를 사용하여 채팅 모델을 연동하는 처음부터 끝까지의 과정을 살펴보았습니다. OpenAI와 Anthropic를 연동하는 기본 채팅 애플리케이션을 구현하고, 대화 기록을 관리하고 스트리밍 응답을 제공하는 기능도 추가했습니다.

이를 통해 Spring AI 1.0.0 GA의 다음과 같은 주요 기능을 활용해보았습니다:

1. **ChatClient Fluent API**: 유창한 API를 통한 직관적인 AI 상호작용 구현
2. **다중 모델 지원**: OpenAI와 Anthropic 모델을 동시에 사용하며 각 프로바이더 간의 전환을 쉽게 구현
3. **Advisors API**: SimpleLoggerAdvisor와 MessageChatMemoryAdvisor를 활용한 기능 확장
4. **Chat Memory**: Spring AI의 내장 메모리 관리 기능으로 대화 기록 유지
5. **스트리밍 응답**: 기본 제공되는 스트리밍 API로 실시간 응답 제공
6. **프롬프트 템플릿**: 파라미터화된 템플릿을 통한 동적 프롬프트 생성

Spring AI 1.0.0 GA는 프로덕션 환경에서 안정적으로 사용할 수 있는 성숙한 프레임워크로, 20개 이상의 AI 모델 프로바이더를 지원하며 엔터프라이즈급 AI 애플리케이션 개발을 위한 완전한 기능을 제공합니다.

다음 장에서는 Embedding과 벡터 스토어를 활용하여 더 고급 AI 기능을 구현하는 방법에 대해 알아보겠습니다.