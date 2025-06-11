# Your First AI-Powered Application

## 개요

이 장에서는 Spring AI를 사용하여 첫 번째 AI 기반 애플리케이션을 만들어보겠습니다. 간단한 채팅 애플리케이션부터 시작하여 점진적으로 기능을 추가하면서 Spring AI의 핵심 개념과 API를 실습해봅니다.

이 장을 통해 다음을 학습하게 됩니다:
- Spring Boot 프로젝트에 Spring AI 통합
- 기본적인 채팅 기능 구현
- 프롬프트 엔지니어링 기초
- 스트리밍 응답 처리
- 대화 기록 관리
- 간단한 웹 인터페이스 구축

## 프로젝트 생성 및 설정

### Spring Initializr로 프로젝트 생성

Spring Initializr를 사용하여 새 프로젝트를 생성합니다:

1. https://start.spring.io 접속
2. 다음 옵션 선택:
   - Project: Gradle - Groovy (또는 Maven)
   - Language: Java
   - Spring Boot: 3.3.0 이상
   - Java: 17 이상
   - Dependencies:
     - Spring Web
     - Spring Boot DevTools

### 의존성 추가

생성된 프로젝트에 Spring AI 의존성을 추가합니다:

#### Gradle (build.gradle)

```gradle
repositories {
    mavenCentral()
}

dependencies {
    implementation platform('org.springframework.ai:spring-ai-bom:1.0.0')
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
    
    // 개발 도구
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    
    // 테스트
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

### 애플리케이션 설정

`src/main/resources/application.yml` 파일을 생성하고 설정을 추가합니다:

```yaml
spring:
  application:
    name: my-first-ai-app
  
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7

# 로깅 설정
logging:
  level:
    org.springframework.ai: INFO
```

환경 변수로 API 키를 설정합니다:

```bash
export OPENAI_API_KEY=your-api-key-here
```

## 기본 채팅 애플리케이션

### 1. 간단한 채팅 컨트롤러

가장 기본적인 채팅 기능을 구현해봅시다:

```java
package com.example.aiapp.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/chat")
public class SimpleChatController {

    private final ChatClient chatClient;

    public SimpleChatController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    @PostMapping("/simple")
    public String chat(@RequestBody String message) {
        return chatClient.prompt()
                .user(message)
                .call()
                .content();
    }
}
```

### 2. 요청/응답 DTO 추가

더 구조화된 API를 위해 DTO를 정의합니다:

```java
package com.example.aiapp.dto;

public class ChatRequest {
    private String message;
    private String systemPrompt;
    
    // 생성자, getter, setter
    public ChatRequest() {}
    
    public ChatRequest(String message) {
        this.message = message;
    }
    
    public String getMessage() { return message; }
    public void setMessage(String message) { this.message = message; }
    
    public String getSystemPrompt() { return systemPrompt; }
    public void setSystemPrompt(String systemPrompt) { this.systemPrompt = systemPrompt; }
}

public class ChatResponse {
    private String message;
    private long timestamp;
    
    public ChatResponse(String message) {
        this.message = message;
        this.timestamp = System.currentTimeMillis();
    }
    
    // getter, setter
    public String getMessage() { return message; }
    public void setMessage(String message) { this.message = message; }
    
    public long getTimestamp() { return timestamp; }
    public void setTimestamp(long timestamp) { this.timestamp = timestamp; }
}
```

### 3. 개선된 채팅 컨트롤러

```java
package com.example.aiapp.controller;

import com.example.aiapp.dto.ChatRequest;
import com.example.aiapp.dto.ChatResponse;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder
                .defaultSystem("You are a helpful AI assistant.")
                .build();
    }

    @PostMapping
    public ChatResponse chat(@RequestBody ChatRequest request) {
        var prompt = chatClient.prompt();
        
        // 시스템 프롬프트가 제공되면 사용
        if (request.getSystemPrompt() != null && !request.getSystemPrompt().isEmpty()) {
            prompt.system(request.getSystemPrompt());
        }
        
        String response = prompt
                .user(request.getMessage())
                .call()
                .content();
                
        return new ChatResponse(response);
    }
}
```

## 스트리밍 응답 구현

실시간 스트리밍 응답을 구현하여 사용자 경험을 개선합니다:

```java
package com.example.aiapp.controller;

import com.example.aiapp.dto.ChatRequest;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

@RestController
@RequestMapping("/api/chat")
public class StreamingChatController {

    private final ChatClient chatClient;

    public StreamingChatController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestBody ChatRequest request) {
        return chatClient.prompt()
                .user(request.getMessage())
                .stream()
                .content();
    }
    
    @PostMapping(value = "/stream-detailed", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ChatResponse> streamChatDetailed(@RequestBody ChatRequest request) {
        return chatClient.prompt()
                .user(request.getMessage())
                .stream()
                .chatResponse();
    }
}
```

## 대화 기록 관리

### 1. 대화 서비스 구현

```java
package com.example.aiapp.service;

import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.messages.AssistantMessage;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class ConversationService {

    private final ChatClient chatClient;
    private final Map<String, List<Message>> conversations = new ConcurrentHashMap<>();

    public ConversationService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder
                .defaultSystem("You are a helpful AI assistant. Maintain context of our conversation.")
                .build();
    }

    public String chat(String conversationId, String userMessage) {
        // 대화 기록 가져오기 또는 생성
        List<Message> messages = conversations.computeIfAbsent(
            conversationId, 
            k -> new ArrayList<>()
        );
        
        // 사용자 메시지 추가
        messages.add(new UserMessage(userMessage));
        
        // AI 응답 생성
        String response = chatClient.prompt()
                .messages(messages)
                .call()
                .content();
        
        // AI 응답 저장
        messages.add(new AssistantMessage(response));
        
        // 대화 기록이 너무 길어지지 않도록 제한
        if (messages.size() > 20) {
            messages.subList(0, 2).clear(); // 오래된 메시지 제거
        }
        
        return response;
    }
    
    public List<Message> getConversationHistory(String conversationId) {
        return conversations.getOrDefault(conversationId, Collections.emptyList());
    }
    
    public void clearConversation(String conversationId) {
        conversations.remove(conversationId);
    }
}
```

### 2. 대화 컨트롤러

```java
package com.example.aiapp.controller;

import com.example.aiapp.service.ConversationService;
import org.springframework.ai.chat.messages.Message;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/conversation")
public class ConversationController {

    private final ConversationService conversationService;

    public ConversationController(ConversationService conversationService) {
        this.conversationService = conversationService;
    }

    @PostMapping("/{conversationId}/chat")
    public Map<String, String> chat(
            @PathVariable String conversationId,
            @RequestBody Map<String, String> request) {
        
        String userMessage = request.get("message");
        String response = conversationService.chat(conversationId, userMessage);
        
        return Map.of(
            "response", response,
            "conversationId", conversationId
        );
    }
    
    @GetMapping("/{conversationId}/history")
    public List<Message> getHistory(@PathVariable String conversationId) {
        return conversationService.getConversationHistory(conversationId);
    }
    
    @DeleteMapping("/{conversationId}")
    public Map<String, String> clearConversation(@PathVariable String conversationId) {
        conversationService.clearConversation(conversationId);
        return Map.of("message", "Conversation cleared");
    }
}
```

## 프롬프트 템플릿 활용

### 1. 프롬프트 템플릿 서비스

```java
package com.example.aiapp.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class PromptTemplateService {

    private final ChatClient chatClient;

    public PromptTemplateService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    public String generateProductDescription(String productName, String features, String targetAudience) {
        String template = """
            Create a compelling product description for the following:
            
            Product Name: {productName}
            Key Features: {features}
            Target Audience: {targetAudience}
            
            The description should be:
            - Engaging and persuasive
            - Highlight the main benefits
            - Appeal to the target audience
            - Be concise (under 150 words)
            """;
        
        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
            "productName", productName,
            "features", features,
            "targetAudience", targetAudience
        ));
        
        return chatClient.prompt(prompt)
                .call()
                .content();
    }
    
    public String translateText(String text, String sourceLanguage, String targetLanguage) {
        String template = """
            Translate the following text from {sourceLanguage} to {targetLanguage}:
            
            {text}
            
            Provide only the translation without any explanations.
            """;
        
        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
            "text", text,
            "sourceLanguage", sourceLanguage,
            "targetLanguage", targetLanguage
        ));
        
        return chatClient.prompt(prompt)
                .call()
                .content();
    }
    
    public String summarizeText(String text, int maxWords) {
        String template = """
            Summarize the following text in no more than {maxWords} words:
            
            {text}
            
            Focus on the key points and main ideas.
            """;
        
        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
            "text", text,
            "maxWords", maxWords
        ));
        
        return chatClient.prompt(prompt)
                .call()
                .content();
    }
}
```

### 2. 프롬프트 템플릿 컨트롤러

```java
package com.example.aiapp.controller;

import com.example.aiapp.service.PromptTemplateService;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/templates")
public class TemplateController {

    private final PromptTemplateService templateService;

    public TemplateController(PromptTemplateService templateService) {
        this.templateService = templateService;
    }

    @PostMapping("/product-description")
    public Map<String, String> generateProductDescription(@RequestBody Map<String, String> request) {
        String description = templateService.generateProductDescription(
            request.get("productName"),
            request.get("features"),
            request.get("targetAudience")
        );
        
        return Map.of("description", description);
    }
    
    @PostMapping("/translate")
    public Map<String, String> translate(@RequestBody Map<String, String> request) {
        String translation = templateService.translateText(
            request.get("text"),
            request.get("sourceLanguage"),
            request.get("targetLanguage")
        );
        
        return Map.of("translation", translation);
    }
    
    @PostMapping("/summarize")
    public Map<String, String> summarize(@RequestBody Map<String, Object> request) {
        String summary = templateService.summarizeText(
            (String) request.get("text"),
            (Integer) request.get("maxWords")
        );
        
        return Map.of("summary", summary);
    }
}
```

## 웹 인터페이스 구축

### 1. HTML 인터페이스

`src/main/resources/static/index.html`:

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My First AI App</title>
    <link rel="stylesheet" href="/css/style.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>🤖 My First AI Application</h1>
            <p>Spring AI로 만든 첫 번째 AI 채팅 애플리케이션</p>
        </header>
        
        <div class="chat-container">
            <div class="chat-messages" id="chatMessages"></div>
            
            <div class="chat-input-container">
                <textarea 
                    id="messageInput" 
                    class="chat-input" 
                    placeholder="메시지를 입력하세요..."
                    rows="3"></textarea>
                <div class="button-group">
                    <button id="sendButton" class="btn btn-primary">전송</button>
                    <button id="clearButton" class="btn btn-secondary">대화 초기화</button>
                </div>
            </div>
        </div>
        
        <div class="features">
            <h2>추가 기능</h2>
            <div class="feature-grid">
                <div class="feature-card">
                    <h3>제품 설명 생성</h3>
                    <button onclick="showProductDescriptionModal()">시작하기</button>
                </div>
                <div class="feature-card">
                    <h3>텍스트 번역</h3>
                    <button onclick="showTranslationModal()">시작하기</button>
                </div>
                <div class="feature-card">
                    <h3>텍스트 요약</h3>
                    <button onclick="showSummaryModal()">시작하기</button>
                </div>
            </div>
        </div>
    </div>
    
    <script src="/js/app.js"></script>
</body>
</html>
```

### 2. CSS 스타일

`src/main/resources/static/css/style.css`:

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background-color: #f5f5f5;
    color: #333;
    line-height: 1.6;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
}

header {
    text-align: center;
    margin-bottom: 30px;
}

header h1 {
    color: #2c3e50;
    margin-bottom: 10px;
}

.chat-container {
    background: white;
    border-radius: 10px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    padding: 20px;
    margin-bottom: 30px;
}

.chat-messages {
    height: 400px;
    overflow-y: auto;
    padding: 20px;
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    margin-bottom: 20px;
    background-color: #fafafa;
}

.message {
    margin-bottom: 15px;
    padding: 10px 15px;
    border-radius: 8px;
    max-width: 70%;
    word-wrap: break-word;
}

.message.user {
    background-color: #007bff;
    color: white;
    margin-left: auto;
    text-align: right;
}

.message.assistant {
    background-color: #e9ecef;
    color: #333;
}

.message .timestamp {
    font-size: 0.8em;
    opacity: 0.7;
    margin-top: 5px;
}

.chat-input-container {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.chat-input {
    width: 100%;
    padding: 12px;
    border: 1px solid #ddd;
    border-radius: 8px;
    font-size: 16px;
    resize: vertical;
}

.button-group {
    display: flex;
    gap: 10px;
    justify-content: flex-end;
}

.btn {
    padding: 10px 20px;
    border: none;
    border-radius: 5px;
    font-size: 16px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.btn-primary {
    background-color: #007bff;
    color: white;
}

.btn-primary:hover {
    background-color: #0056b3;
}

.btn-secondary {
    background-color: #6c757d;
    color: white;
}

.btn-secondary:hover {
    background-color: #545b62;
}

.features {
    margin-top: 40px;
}

.features h2 {
    text-align: center;
    margin-bottom: 20px;
    color: #2c3e50;
}

.feature-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 20px;
}

.feature-card {
    background: white;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    text-align: center;
}

.feature-card h3 {
    margin-bottom: 15px;
    color: #333;
}

.feature-card button {
    background-color: #28a745;
    color: white;
    border: none;
    padding: 10px 20px;
    border-radius: 5px;
    cursor: pointer;
    font-size: 14px;
}

.feature-card button:hover {
    background-color: #218838;
}

/* Loading animation */
.loading {
    display: inline-block;
    width: 20px;
    height: 20px;
    border: 3px solid rgba(0, 0, 0, 0.1);
    border-radius: 50%;
    border-top-color: #007bff;
    animation: spin 1s ease-in-out infinite;
}

@keyframes spin {
    to { transform: rotate(360deg); }
}
```

### 3. JavaScript 클라이언트

`src/main/resources/static/js/app.js`:

```javascript
// 전역 변수
let conversationId = generateUUID();
const chatMessages = document.getElementById('chatMessages');
const messageInput = document.getElementById('messageInput');
const sendButton = document.getElementById('sendButton');
const clearButton = document.getElementById('clearButton');

// UUID 생성 함수
function generateUUID() {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
        const r = Math.random() * 16 | 0;
        const v = c === 'x' ? r : (r & 0x3 | 0x8);
        return v.toString(16);
    });
}

// 메시지 추가 함수
function addMessage(content, isUser) {
    const messageDiv = document.createElement('div');
    messageDiv.className = `message ${isUser ? 'user' : 'assistant'}`;
    
    const contentDiv = document.createElement('div');
    contentDiv.textContent = content;
    messageDiv.appendChild(contentDiv);
    
    const timestampDiv = document.createElement('div');
    timestampDiv.className = 'timestamp';
    timestampDiv.textContent = new Date().toLocaleTimeString();
    messageDiv.appendChild(timestampDiv);
    
    chatMessages.appendChild(messageDiv);
    chatMessages.scrollTop = chatMessages.scrollHeight;
}

// 로딩 표시
function showLoading() {
    const loadingDiv = document.createElement('div');
    loadingDiv.className = 'message assistant loading-message';
    loadingDiv.innerHTML = '<div class="loading"></div> AI가 생각하고 있습니다...';
    chatMessages.appendChild(loadingDiv);
    chatMessages.scrollTop = chatMessages.scrollHeight;
    return loadingDiv;
}

// 메시지 전송
async function sendMessage() {
    const message = messageInput.value.trim();
    if (!message) return;
    
    // 사용자 메시지 표시
    addMessage(message, true);
    messageInput.value = '';
    
    // 로딩 표시
    const loadingDiv = showLoading();
    
    try {
        const response = await fetch(`/api/conversation/${conversationId}/chat`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ message: message }),
        });
        
        if (!response.ok) {
            throw new Error('서버 오류가 발생했습니다.');
        }
        
        const data = await response.json();
        
        // 로딩 제거
        loadingDiv.remove();
        
        // AI 응답 표시
        addMessage(data.response, false);
        
    } catch (error) {
        loadingDiv.remove();
        addMessage('오류: ' + error.message, false);
    }
}

// 대화 초기화
async function clearConversation() {
    if (!confirm('대화를 초기화하시겠습니까?')) return;
    
    try {
        await fetch(`/api/conversation/${conversationId}`, {
            method: 'DELETE',
        });
        
        // 새 대화 ID 생성
        conversationId = generateUUID();
        
        // 화면 초기화
        chatMessages.innerHTML = '';
        addMessage('새로운 대화가 시작되었습니다.', false);
        
    } catch (error) {
        console.error('Error clearing conversation:', error);
    }
}

// 이벤트 리스너
sendButton.addEventListener('click', sendMessage);
clearButton.addEventListener('click', clearConversation);

messageInput.addEventListener('keypress', (e) => {
    if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault();
        sendMessage();
    }
});

// 추가 기능 모달 (예시)
function showProductDescriptionModal() {
    // 실제 구현에서는 모달 UI를 표시
    const productName = prompt('제품 이름을 입력하세요:');
    const features = prompt('주요 기능을 입력하세요:');
    const targetAudience = prompt('타겟 고객층을 입력하세요:');
    
    if (productName && features && targetAudience) {
        generateProductDescription(productName, features, targetAudience);
    }
}

async function generateProductDescription(productName, features, targetAudience) {
    const loadingDiv = showLoading();
    
    try {
        const response = await fetch('/api/templates/product-description', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                productName,
                features,
                targetAudience
            }),
        });
        
        const data = await response.json();
        loadingDiv.remove();
        addMessage(`제품 설명이 생성되었습니다:\n\n${data.description}`, false);
        
    } catch (error) {
        loadingDiv.remove();
        addMessage('오류: ' + error.message, false);
    }
}

// 초기 메시지
addMessage('안녕하세요! 무엇을 도와드릴까요?', false);
```

## 애플리케이션 테스트

### 1. 단위 테스트

```java
package com.example.aiapp;

import com.example.aiapp.controller.ChatController;
import com.example.aiapp.dto.ChatRequest;
import com.example.aiapp.dto.ChatResponse;
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;

@SpringBootTest
class ChatControllerTest {

    @Autowired
    private ChatController chatController;
    
    @MockBean
    private ChatModel chatModel;
    
    @Test
    void testBasicChat() {
        // Given
        ChatRequest request = new ChatRequest("Hello, AI!");
        
        // When
        ChatResponse response = chatController.chat(request);
        
        // Then
        assertThat(response).isNotNull();
        assertThat(response.getMessage()).isNotEmpty();
        assertThat(response.getTimestamp()).isGreaterThan(0);
    }
}
```

### 2. 통합 테스트

```java
package com.example.aiapp;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class ApiIntegrationTest {

    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void testChatEndpoint() throws Exception {
        String requestBody = """
            {
                "message": "What is Spring AI?"
            }
            """;
        
        mockMvc.perform(post("/api/chat")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.message").exists())
                .andExpect(jsonPath("$.timestamp").exists());
    }
}
```

## 애플리케이션 실행 및 배포

### 1. 로컬 실행

```bash
# Gradle
./gradlew bootRun

# Maven
./mvnw spring-boot:run
```

브라우저에서 http://localhost:8080 접속

### 2. 프로덕션 빌드

```bash
# Gradle
./gradlew build

# Maven
./mvnw clean package
```

### 3. Docker 이미지 생성

`Dockerfile`:

```dockerfile
FROM eclipse-temurin:17-jre-alpine
VOLUME /tmp
COPY build/libs/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

```bash
docker build -t my-first-ai-app .
docker run -p 8080:8080 -e OPENAI_API_KEY=your-key my-first-ai-app
```

## 다음 단계 및 개선 사항

### 1. 보안 강화
- API 키 관리 개선 (Key Vault 사용)
- 사용자 인증 및 권한 관리
- Rate limiting 구현

### 2. 기능 확장
- 파일 업로드 및 처리
- 음성 입력/출력 지원
- 다국어 지원

### 3. 성능 최적화
- 응답 캐싱
- 비동기 처리
- 데이터베이스 통합

### 4. 모니터링
- 로깅 및 메트릭 수집
- 사용량 추적
- 오류 모니터링

## 결론

이 장에서는 Spring AI를 사용하여 첫 번째 AI 기반 애플리케이션을 만들어보았습니다. 간단한 채팅 기능부터 시작하여 스트리밍, 대화 관리, 템플릿 활용 등 다양한 기능을 구현했습니다.

주요 학습 내용:
- Spring Boot 프로젝트에 Spring AI 통합
- ChatClient API를 사용한 기본 채팅 구현
- 스트리밍 응답 처리
- 대화 컨텍스트 관리
- 프롬프트 템플릿 활용
- 웹 인터페이스 구축

이제 Spring AI의 기본적인 사용법을 익혔으므로, 다음 장에서는 더 고급 기능들을 살펴보겠습니다. RAG 패턴, Function Calling, 멀티모달 AI 등을 활용하여 더욱 강력한 AI 애플리케이션을 구축할 수 있습니다.