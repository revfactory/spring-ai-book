# 스트리밍 응답과 비동기 처리

## 개요

실시간 및 장문의 응답이 필요한 현대적인 AI 애플리케이션에서 스트리밍 응답과 비동기 처리는 필수적인 기능입니다. 사용자 경험을 향상시키고, 서버 리소스를 효율적으로 활용하며, 복잡한 AI 워크플로우를 구현하기 위해서는 이러한 기술이 필요합니다.

이 장에서는 Spring AI를 사용하여 스트리밍 응답을 처리하고 비동기 API를 설계하는 방법을 살펴보겠습니다. Spring WebFlux, Project Reactor, 그리고 Spring AI의 비동기 클라이언트를 활용하여 확장 가능하고 반응형 AI 애플리케이션을 구축하는 방법을 배우게 됩니다.

## 스트리밍 응답 이해하기

### 스트리밍 응답의 중요성

AI 모델, 특히 LLM은 일반적으로 응답을 단어 단위로 생성합니다. 전통적인 요청-응답 모델에서는 AI가 전체 응답을 생성한 후에야 사용자에게 응답이 표시됩니다. 이는 다음과 같은 문제를 야기합니다:

1. **긴 대기 시간**: 특히 긴 응답의 경우 사용자는 전체 응답이 생성될 때까지 기다려야 합니다.
2. **사용자 경험 저하**: 응답이 없는 시간이 길어지면 사용자는 시스템이 작동하지 않는다고 생각할 수 있습니다.
3. **리소스 비효율성**: 클라이언트와 서버 간의 연결이 응답이 생성되는 전체 시간 동안 유지되어야 합니다.

스트리밍 응답은 이러한 문제를 해결하고 다음과 같은 이점을 제공합니다:

1. **실시간 피드백**: 사용자는 응답이 생성되는 대로 즉시 볼 수 있습니다.
2. **개선된 사용자 경험**: 사용자는 시스템이 작동 중임을 즉시 확인할 수 있습니다.
3. **조기 중단 가능**: 사용자는 원하는 정보를 얻으면 응답 생성을 중단할 수 있습니다.

### Spring AI의 스트리밍 지원

Spring AI는 주요 AI 모델 프로바이더의 스트리밍 응답을 지원합니다:

1. **OpenAI**: ChatGPT 모델의 스트리밍 응답
2. **Anthropic**: Claude 모델의 스트리밍 응답
3. **Vertex AI**: Google의 Gemini 모델 스트리밍
4. **Amazon Bedrock**: 다양한 모델의 스트리밍 지원

Spring AI는 이러한 다양한 API들을 통합하여 일관된 인터페이스를 제공합니다. 대부분의 ChatClient 구현체는 아래 세 가지 주요 호출 패턴을 지원합니다:

```java
// 동기 호출 (전체 응답 반환까지 대기)
ChatResponse response = chatClient.call(prompt);

// 리액티브 호출 (비동기, Mono 타입 반환)
Mono<ChatResponse> responseMono = chatClient.callReactive(prompt);

// 스트리밍 호출 (비동기, Flux 타입으로 부분 응답 스트림 반환)
Flux<ChatResponse> responseFlux = chatClient.stream(prompt);
```

## Spring WebFlux와 Project Reactor 기초

Spring AI의 스트리밍 및 비동기 기능을 활용하기 위해서는 Spring WebFlux와 Project Reactor를 이해해야 합니다.

### Project Reactor 핵심 개념

Project Reactor는 JVM에서 리액티브 프로그래밍을 위한 라이브러리로, Spring WebFlux의 기반이 됩니다. 핵심 타입으로는 다음이 있습니다:

1. **Mono\<T>**: 0 또는 1개의 결과를 비동기적으로 생성하는 Publisher
2. **Flux\<T>**: 0개 이상의 결과를 비동기적으로 생성하는 Publisher

이러한 타입들은 다음과 같은 특징을 가집니다:

- **선언적**: 실행할 작업 흐름을 선언적으로 정의
- **조합 가능**: 여러 연산자를 결합하여 복잡한 데이터 흐름 생성
- **백프레셔 지원**: 데이터 생성자가 소비자의 처리 능력에 맞춰 데이터 생성 속도 조절

### Spring WebFlux 소개

Spring WebFlux는 Spring 5에서 도입된 완전한 비동기, 논블로킹 웹 프레임워크입니다. 이는 다음과 같은 특징을 가집니다:

- **논블로킹 I/O**: 적은 수의 스레드로 많은 동시 연결 처리 가능
- **리액티브 스트림**: Mono와 Flux를 사용한 비동기 데이터 스트림 처리
- **이벤트 루프 모델**: 논블로킹 API를 기반으로 하는 이벤트 기반 프로그래밍 모델

Spring AI의 스트리밍 기능은 이러한 리액티브 타입들을 활용합니다.

## Spring AI에서 스트리밍 구현하기

이제 Spring AI와 WebFlux를 사용하여 스트리밍 응답을 구현하는 방법을 살펴보겠습니다.

### 의존성 설정

스트리밍 응답을 구현하기 위해 필요한 의존성을 추가합니다:

```kotlin
// Gradle (build.gradle.kts)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springframework.ai:spring-ai-openai-spring-boot-starter:1.0.0-M6")
    // 추가 의존성 생략
}
```

```xml
<!-- Maven (pom.xml) -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        <version>1.0.0-M6</version>
    </dependency>
    <!-- 추가 의존성 생략 -->
</dependencies>
```

### 기본 스트리밍 서비스 구현

```java
package com.example.streamingdemo.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.List;

@Service
public class ChatStreamingService {

    private final ChatClient chatClient;

    @Autowired
    public ChatStreamingService(@Qualifier("openAiChatClient") ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    // 기본 동기식 호출 (스트리밍 없음)
    public String chat(String userMessage) {
        Prompt prompt = new Prompt(new UserMessage(userMessage));
        ChatResponse response = chatClient.call(prompt);
        return response.getResult().getOutput().getContent();
    }

    // 리액티브 호출 (비동기, 단일 응답)
    public Mono<String> chatReactive(String userMessage) {
        Prompt prompt = new Prompt(new UserMessage(userMessage));
        return chatClient.callReactive(prompt)
                .map(response -> response.getResult().getOutput().getContent());
    }

    // 스트리밍 호출 (비동기, 증분 응답)
    public Flux<String> chatStream(String userMessage) {
        Prompt prompt = new Prompt(new UserMessage(userMessage));
        return chatClient.stream(prompt)
                .map(response -> response.getResult().getOutput().getContent());
    }

    // 시스템 프롬프트가 포함된 스트리밍 호출
    public Flux<String> chatStreamWithSystem(String userMessage, String systemMessage) {
        List<Message> messages = List.of(
                new SystemMessage(systemMessage),
                new UserMessage(userMessage)
        );
        
        Prompt prompt = new Prompt(messages);
        return chatClient.stream(prompt)
                .map(response -> response.getResult().getOutput().getContent());
    }
}
```

### 스트리밍 컨트롤러 구현

```java
package com.example.streamingdemo.controller;

import com.example.streamingdemo.service.ChatStreamingService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/api/chat")
public class ChatStreamingController {

    private final ChatStreamingService chatStreamingService;

    @Autowired
    public ChatStreamingController(ChatStreamingService chatStreamingService) {
        this.chatStreamingService = chatStreamingService;
    }

    // 동기식 엔드포인트
    @PostMapping
    public ChatResponse chat(@RequestBody ChatRequest request) {
        String response = chatStreamingService.chat(request.getMessage());
        return new ChatResponse(response);
    }

    // 리액티브 엔드포인트
    @PostMapping("/reactive")
    public Mono<ChatResponse> chatReactive(@RequestBody ChatRequest request) {
        return chatStreamingService.chatReactive(request.getMessage())
                .map(ChatResponse::new);
    }

    // Server-Sent Events 스트리밍 엔드포인트
    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ChatResponse> chatStream(@RequestBody ChatRequest request) {
        return chatStreamingService.chatStream(request.getMessage())
                .map(ChatResponse::new);
    }

    // 시스템 프롬프트가 포함된 스트리밍 엔드포인트
    @PostMapping(value = "/stream-with-system", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ChatResponse> chatStreamWithSystem(@RequestBody ChatRequestWithSystem request) {
        return chatStreamingService.chatStreamWithSystem(
                request.getMessage(),
                request.getSystemMessage()
        ).map(ChatResponse::new);
    }

    // 요청 및 응답 클래스
    public static class ChatRequest {
        private String message;

        // getter, setter
        public String getMessage() {
            return message;
        }

        public void setMessage(String message) {
            this.message = message;
        }
    }

    public static class ChatRequestWithSystem {
        private String message;
        private String systemMessage;

        // getter, setter
        public String getMessage() {
            return message;
        }

        public void setMessage(String message) {
            this.message = message;
        }

        public String getSystemMessage() {
            return systemMessage;
        }

        public void setSystemMessage(String systemMessage) {
            this.systemMessage = systemMessage;
        }
    }

    public static class ChatResponse {
        private String content;

        public ChatResponse(String content) {
            this.content = content;
        }

        public String getContent() {
            return content;
        }
    }
}
```

### 웹 클라이언트에서 스트리밍 처리하기

다음 예제는 JavaScript와 HTML을 사용하여 서버에서 스트리밍되는 응답을 처리하는 방법을 보여줍니다:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI Chat Streaming Demo</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        }
        
        #chat-container {
            display: flex;
            flex-direction: column;
            gap: 20px;
        }
        
        #message-input {
            padding: 10px;
            width: 100%;
            box-sizing: border-box;
        }
        
        #send-button, #stream-button {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            cursor: pointer;
        }
        
        #stream-button {
            background-color: #2196F3;
        }
        
        .response-area {
            border: 1px solid #ddd;
            padding: 15px;
            min-height: 100px;
            margin-top: 10px;
            white-space: pre-wrap;
        }
        
        .typing-indicator {
            display: inline-block;
        }
        
        .typing-indicator span {
            display: inline-block;
            width: 8px;
            height: 8px;
            border-radius: 50%;
            background-color: #666;
            margin: 0 2px;
            animation: bounce 1.4s infinite ease-in-out;
        }
        
        .typing-indicator span:nth-child(1) { animation-delay: 0s; }
        .typing-indicator span:nth-child(2) { animation-delay: 0.2s; }
        .typing-indicator span:nth-child(3) { animation-delay: 0.4s; }
        
        @keyframes bounce {
            0%, 80%, 100% { transform: translateY(0); }
            40% { transform: translateY(-10px); }
        }
    </style>
</head>
<body>
    <h1>AI Chat Streaming Demo</h1>
    
    <div id="chat-container">
        <div>
            <label for="message-input">Your message:</label>
            <textarea id="message-input" rows="4" placeholder="Type your question here..."></textarea>
        </div>
        
        <div>
            <button id="send-button">Send (No Streaming)</button>
            <button id="stream-button">Send (With Streaming)</button>
        </div>
        
        <div>
            <h3>Response:</h3>
            <div id="response-container" class="response-area"></div>
            <div id="typing-indicator" class="typing-indicator" style="display: none;">
                <span></span>
                <span></span>
                <span></span>
            </div>
        </div>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', function() {
            const messageInput = document.getElementById('message-input');
            const sendButton = document.getElementById('send-button');
            const streamButton = document.getElementById('stream-button');
            const responseContainer = document.getElementById('response-container');
            const typingIndicator = document.getElementById('typing-indicator');
            
            // 일반 요청 (스트리밍 없음)
            sendButton.addEventListener('click', async function() {
                const message = messageInput.value.trim();
                if (!message) return;
                
                responseContainer.textContent = '';
                typingIndicator.style.display = 'inline-block';
                
                try {
                    const response = await fetch('/api/chat', {
                        method: 'POST',
                        headers: {
                            'Content-Type': 'application/json'
                        },
                        body: JSON.stringify({ message: message })
                    });
                    
                    const result = await response.json();
                    responseContainer.textContent = result.content;
                } catch (error) {
                    console.error('Error:', error);
                    responseContainer.textContent = 'Error: ' + error.message;
                } finally {
                    typingIndicator.style.display = 'none';
                }
            });
            
            // 스트리밍 요청
            streamButton.addEventListener('click', function() {
                const message = messageInput.value.trim();
                if (!message) return;
                
                responseContainer.textContent = '';
                
                // 이벤트 소스 생성
                const eventSource = new EventSource(`/api/chat/stream?message=${encodeURIComponent(message)}`);
                
                // 연결 열림 이벤트
                eventSource.onopen = function() {
                    console.log('SSE connection opened');
                };
                
                // 메시지 수신 이벤트
                eventSource.onmessage = function(event) {
                    const data = JSON.parse(event.data);
                    
                    // 이전 콘텐츠에 추가
                    if (responseContainer.textContent === '') {
                        responseContainer.textContent = data.content;
                    } else {
                        // 이전 응답에 새 청크 추가
                        responseContainer.textContent += data.content;
                    }
                    
                    // 자동 스크롤
                    responseContainer.scrollTop = responseContainer.scrollHeight;
                };
                
                // 오류 처리
                eventSource.onerror = function(error) {
                    console.error('SSE Error:', error);
                    eventSource.close();
                };
                
                // 완료 처리 (서버에서 'complete' 이벤트 발생 시)
                eventSource.addEventListener('complete', function() {
                    eventSource.close();
                    console.log('Streaming complete');
                });
            });
        });
    </script>
</body>
</html>
```

## 스트리밍 응답 최적화

스트리밍 응답을 효과적으로 활용하기 위한 몇 가지 최적화 기법을 살펴보겠습니다.

### 스트리밍 응답 통합

대부분의 AI 모델은 응답을 작은 청크(토큰 또는 단어)로 스트리밍합니다. 이러한 청크가 너무 작으면 클라이언트 처리가 비효율적일 수 있습니다. 다음은 청크를 통합하는 방법입니다:

```java
package com.example.streamingdemo.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;

import java.time.Duration;
import java.util.concurrent.atomic.AtomicReference;

@Service
public class OptimizedStreamingService {

    private final ChatClient chatClient;

    public OptimizedStreamingService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    // 타임 기반 버퍼링
    public Flux<String> streamWithTimeBuffering(String userMessage) {
        Prompt prompt = new Prompt(new UserMessage(userMessage));
        
        return chatClient.stream(prompt)
                .map(response -> response.getResult().getOutput().getContent())
                .bufferTimeout(5, Duration.ofMillis(100))  // 5개 청크 또는 100ms마다 버퍼링
                .map(chunks -> String.join("", chunks));
    }

    // 크기 기반 버퍼링
    public Flux<String> streamWithSizeThreshold(String userMessage) {
        Prompt prompt = new Prompt(new UserMessage(userMessage));
        AtomicReference<StringBuilder> buffer = new AtomicReference<>(new StringBuilder());
        final int THRESHOLD = 20;  // 문자 단위 임계값
        
        return chatClient.stream(prompt)
                .map(response -> response.getResult().getOutput().getContent())
                .concatMap(chunk -> {
                    buffer.get().append(chunk);
                    
                    if (buffer.get().length() >= THRESHOLD) {
                        String result = buffer.get().toString();
                        buffer.set(new StringBuilder());
                        return Flux.just(result);
                    } else {
                        return Flux.empty();
                    }
                })
                // 남은 버퍼 내용 처리
                .concatWith(Flux.defer(() -> {
                    if (buffer.get().length() > 0) {
                        return Flux.just(buffer.get().toString());
                    }
                    return Flux.empty();
                }));
    }
}
```

### 스트리밍 중 토큰 집계

전체 응답을 생성하면서 스트리밍하는 방법을 구현해 보겠습니다:

```java
package com.example.streamingdemo.controller;

import com.example.streamingdemo.service.ChatStreamingService;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

import java.util.concurrent.atomic.AtomicReference;

@RestController
@RequestMapping("/api/chat")
public class AdvancedStreamingController {

    private final ChatStreamingService chatStreamingService;

    public AdvancedStreamingController(ChatStreamingService chatStreamingService) {
        this.chatStreamingService = chatStreamingService;
    }

    @PostMapping(value = "/stream-accumulated", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<StreamingResponse> streamWithAccumulation(@RequestBody ChatRequest request) {
        AtomicReference<StringBuilder> fullResponseBuilder = new AtomicReference<>(new StringBuilder());
        
        return chatStreamingService.chatStream(request.getMessage())
                .map(chunk -> {
                    // 현재 청크를 전체 응답에 추가
                    fullResponseBuilder.get().append(chunk);
                    String fullText = fullResponseBuilder.get().toString();
                    
                    // 부분 응답(델타)와 전체 텍스트 모두 반환
                    return new StreamingResponse(chunk, fullText, false);
                })
                // 스트림 종료 시 완료 플래그가 있는 마지막 응답 추가
                .concatWith(Flux.just(
                    new StreamingResponse("", fullResponseBuilder.get().toString(), true)
                ));
    }
    
    public static class ChatRequest {
        private String message;
        
        // getter, setter
        public String getMessage() {
            return message;
        }
        
        public void setMessage(String message) {
            this.message = message;
        }
    }
    
    public static class StreamingResponse {
        private String delta;
        private String fullText;
        private boolean done;
        
        public StreamingResponse(String delta, String fullText, boolean done) {
            this.delta = delta;
            this.fullText = fullText;
            this.done = done;
        }
        
        public String getDelta() {
            return delta;
        }
        
        public String getFullText() {
            return fullText;
        }
        
        public boolean isDone() {
            return done;
        }
    }
}
```

## 비동기 처리 패턴

복잡한 AI 워크플로우를 위한 비동기 처리 패턴을 살펴보겠습니다.

### 병렬 AI 요청 처리

여러 AI 요청을 병렬로 처리하는 예제입니다:

```java
package com.example.streamingdemo.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.Map;

@Service
public class ParallelProcessingService {

    private final ChatClient chatClient;

    public ParallelProcessingService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    // 여러 AI 요청을 병렬로 처리
    public Mono<Map<String, String>> processQuestionsInParallel(List<String> questions) {
        return Flux.fromIterable(questions)
                .flatMap(question -> processQuestion(question)
                        .map(answer -> Map.entry(question, answer)))
                .collectMap(Map.Entry::getKey, Map.Entry::getValue);
    }

    private Mono<String> processQuestion(String question) {
        Prompt prompt = new Prompt(new UserMessage(question));
        return chatClient.callReactive(prompt)
                .map(response -> response.getResult().getOutput().getContent());
    }

    // 여러 질문을 합쳐서 단일 AI 요청으로 처리
    public Mono<String> processBatchedQuestions(List<String> questions) {
        // 모든 질문을 결합
        String combinedQuestions = String.join("\n\n", questions);
        
        String fullPrompt = """
                Answer each of the following questions concisely:
                
                %s
                
                Format your answers as:
                Q1: [first question]
                A1: [your answer]
                
                Q2: [second question]
                A2: [your answer]
                
                And so on.
                """.formatted(combinedQuestions);
        
        Prompt prompt = new Prompt(new UserMessage(fullPrompt));
        return chatClient.callReactive(prompt)
                .map(response -> response.getResult().getOutput().getContent());
    }
}
```

### 비동기 콜백 처리

긴 실행 시간이 필요한 AI 작업을 처리하기 위한 비동기 콜백 패턴입니다:

```java
package com.example.streamingdemo.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CompletableFuture;
import java.util.function.Consumer;

@Service
public class AsyncCallbackService {

    private final ChatClient chatClient;
    private final Map<String, JobStatus> jobStatusMap = new ConcurrentHashMap<>();

    public AsyncCallbackService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    // 작업 상태 확인
    public JobStatus checkJobStatus(String jobId) {
        return jobStatusMap.getOrDefault(jobId, new JobStatus(jobId, "NOT_FOUND", null, null));
    }

    // 비동기 작업 시작
    public String startAsyncJob(String prompt) {
        String jobId = UUID.randomUUID().toString();
        jobStatusMap.put(jobId, new JobStatus(jobId, "PENDING", null, null));
        
        processAsyncJob(jobId, prompt);
        
        return jobId;
    }

    // 콜백이 있는 비동기 작업 시작
    public String startAsyncJobWithCallback(String prompt, Consumer<JobStatus> callback) {
        String jobId = UUID.randomUUID().toString();
        jobStatusMap.put(jobId, new JobStatus(jobId, "PENDING", null, null));
        
        CompletableFuture.runAsync(() -> {
            try {
                // 상태 업데이트
                jobStatusMap.put(jobId, new JobStatus(jobId, "PROCESSING", null, null));
                
                // AI 요청 처리
                Prompt aiPrompt = new Prompt(new UserMessage(prompt));
                String result = chatClient.call(aiPrompt).getResult().getOutput().getContent();
                
                // 완료 상태로 업데이트
                JobStatus completedStatus = new JobStatus(jobId, "COMPLETED", result, null);
                jobStatusMap.put(jobId, completedStatus);
                
                // 콜백 호출
                callback.accept(completedStatus);
                
            } catch (Exception e) {
                // 오류 상태로 업데이트
                JobStatus errorStatus = new JobStatus(jobId, "ERROR", null, e.getMessage());
                jobStatusMap.put(jobId, errorStatus);
                callback.accept(errorStatus);
            }
        });
        
        return jobId;
    }

    @Async
    public void processAsyncJob(String jobId, String prompt) {
        try {
            // 상태 업데이트
            jobStatusMap.put(jobId, new JobStatus(jobId, "PROCESSING", null, null));
            
            // AI 요청 처리
            Prompt aiPrompt = new Prompt(new UserMessage(prompt));
            String result = chatClient.call(aiPrompt).getResult().getOutput().getContent();
            
            // 완료 상태로 업데이트
            jobStatusMap.put(jobId, new JobStatus(jobId, "COMPLETED", result, null));
            
        } catch (Exception e) {
            // 오류 상태로 업데이트
            jobStatusMap.put(jobId, new JobStatus(jobId, "ERROR", null, e.getMessage()));
        }
    }

    // 작업 상태 클래스
    public static class JobStatus {
        private final String jobId;
        private final String status;  // PENDING, PROCESSING, COMPLETED, ERROR
        private final String result;
        private final String errorMessage;
        
        public JobStatus(String jobId, String status, String result, String errorMessage) {
            this.jobId = jobId;
            this.status = status;
            this.result = result;
            this.errorMessage = errorMessage;
        }
        
        // getters
        public String getJobId() {
            return jobId;
        }
        
        public String getStatus() {
            return status;
        }
        
        public String getResult() {
            return result;
        }
        
        public String getErrorMessage() {
            return errorMessage;
        }
    }
}
```

### 비동기 API 엔드포인트

비동기 작업을 처리하는 컨트롤러입니다:

```java
package com.example.streamingdemo.controller;

import com.example.streamingdemo.service.AsyncCallbackService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.async.DeferredResult;

import java.util.concurrent.ForkJoinPool;
import java.util.function.Consumer;

@RestController
@RequestMapping("/api/async")
public class AsyncController {

    private final AsyncCallbackService asyncCallbackService;

    public AsyncController(AsyncCallbackService asyncCallbackService) {
        this.asyncCallbackService = asyncCallbackService;
    }

    @PostMapping("/start-job")
    public ResponseEntity<JobResponse> startJob(@RequestBody JobRequest request) {
        String jobId = asyncCallbackService.startAsyncJob(request.getPrompt());
        return ResponseEntity.accepted().body(new JobResponse(jobId));
    }

    @GetMapping("/job-status/{jobId}")
    public ResponseEntity<AsyncCallbackService.JobStatus> getJobStatus(@PathVariable String jobId) {
        AsyncCallbackService.JobStatus status = asyncCallbackService.checkJobStatus(jobId);
        
        if ("NOT_FOUND".equals(status.getStatus())) {
            return ResponseEntity.notFound().build();
        }
        
        return ResponseEntity.ok(status);
    }

    @PostMapping("/long-polling")
    public DeferredResult<AsyncCallbackService.JobStatus> longPollingJob(@RequestBody JobRequest request) {
        DeferredResult<AsyncCallbackService.JobStatus> output = new DeferredResult<>(120000L);  // 2분 타임아웃
        
        Consumer<AsyncCallbackService.JobStatus> callback = output::setResult;
        
        String jobId = asyncCallbackService.startAsyncJobWithCallback(request.getPrompt(), callback);
        
        // 클라이언트 연결이 끊어진 경우 처리
        output.onTimeout(() -> 
            output.setErrorResult(ResponseEntity.status(408)
                .body(new AsyncCallbackService.JobStatus(jobId, "TIMEOUT", null, "Request timed out"))));
        
        return output;
    }

    @PostMapping("/server-sent-events")
    public ResponseEntity<JobResponse> sseJob(@RequestBody JobRequest request) {
        String jobId = asyncCallbackService.startAsyncJob(request.getPrompt());
        return ResponseEntity.accepted().body(new JobResponse(jobId));
    }

    // 서버 전송 이벤트 엔드포인트
    @GetMapping(value = "/sse/{jobId}", produces = "text/event-stream")
    public DeferredResult<String> streamJobStatus(@PathVariable String jobId) {
        DeferredResult<String> output = new DeferredResult<>(120000L);  // 2분 타임아웃
        
        Runnable statusChecker = new Runnable() {
            @Override
            public void run() {
                AsyncCallbackService.JobStatus status = asyncCallbackService.checkJobStatus(jobId);
                
                // SSE 형식으로 상태 업데이트 전송
                String eventData = String.format("event: status\ndata: {\"status\":\"%s\",\"jobId\":\"%s\"}\n\n", 
                        status.getStatus(), status.getJobId());
                
                if ("COMPLETED".equals(status.getStatus())) {
                    // 완료 이벤트 및 결과 전송
                    eventData += String.format("event: complete\ndata: {\"result\":%s}\n\n", 
                            escapeJson(status.getResult()));
                    output.setResult(eventData);
                } else if ("ERROR".equals(status.getStatus())) {
                    // 오류 이벤트 전송
                    eventData += String.format("event: error\ndata: {\"message\":\"%s\"}\n\n", 
                            status.getErrorMessage());
                    output.setResult(eventData);
                } else if ("NOT_FOUND".equals(status.getStatus())) {
                    output.setErrorResult("Job not found");
                } else {
                    output.setResult(eventData);
                    
                    // 1초 후 다시 상태 확인
                    ForkJoinPool.commonPool().submit(() -> {
                        try {
                            Thread.sleep(1000);
                            this.run();
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                    });
                }
            }
        };
        
        // 즉시 첫 번째 상태 확인 시작
        ForkJoinPool.commonPool().submit(statusChecker);
        
        return output;
    }

    private String escapeJson(String json) {
        if (json == null) {
            return "null";
        }
        return "\"" + json.replace("\"", "\\\"").replace("\n", "\\n") + "\"";
    }

    // 요청 및 응답 클래스
    public static class JobRequest {
        private String prompt;
        
        // getter, setter
        public String getPrompt() {
            return prompt;
        }
        
        public void setPrompt(String prompt) {
            this.prompt = prompt;
        }
    }
    
    public static class JobResponse {
        private String jobId;
        
        public JobResponse(String jobId) {
            this.jobId = jobId;
        }
        
        public String getJobId() {
            return jobId;
        }
    }
}
```

## 스트리밍과 WebSocket 통합

WebSocket을 사용하여 더 대화형 AI 스트리밍 경험을 구현해 보겠습니다.

### WebSocket 구성

```java
package com.example.streamingdemo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatWebSocketHandler(), "/ws/chat")
                .setAllowedOrigins("*");
    }

    @Bean
    public ChatWebSocketHandler chatWebSocketHandler() {
        return new ChatWebSocketHandler();
    }

    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

### WebSocket 핸들러

```java
package com.example.streamingdemo.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

import java.io.IOException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class ChatWebSocketHandler extends TextWebSocketHandler {

    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Autowired
    private ChatClient chatClient;

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.put(session.getId(), session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String payload = message.getPayload();
        ChatMessage chatMessage = objectMapper.readValue(payload, ChatMessage.class);
        
        if ("PING".equals(chatMessage.getType())) {
            sendMessage(session, new ChatMessage("PONG", null));
            return;
        }
        
        if ("CHAT".equals(chatMessage.getType())) {
            handleChatMessage(session, chatMessage);
        }
    }

    private void handleChatMessage(WebSocketSession session, ChatMessage chatMessage) {
        // 클라이언트에게 메시지 처리 시작을 알림
        sendMessage(session, new ChatMessage("PROCESSING", null));
        
        // 비동기로 AI 응답 스트리밍 처리
        Prompt prompt = new Prompt(new UserMessage(chatMessage.getContent()));
        
        // StringBuilder로 전체 응답을 모을 준비
        StringBuilder fullResponse = new StringBuilder();
        
        // 스트리밍 응답 구독
        chatClient.stream(prompt)
                .subscribe(
                    response -> {
                        String content = response.getResult().getOutput().getContent();
                        fullResponse.append(content);
                        
                        // 각 청크를 클라이언트에 전송
                        sendMessage(session, new ChatMessage("CHUNK", content));
                    },
                    error -> {
                        // 오류 처리
                        sendMessage(session, new ChatMessage("ERROR", error.getMessage()));
                    },
                    () -> {
                        // 완료 시 전체 응답 전송
                        sendMessage(session, new ChatMessage("COMPLETE", fullResponse.toString()));
                    }
                );
    }

    private void sendMessage(WebSocketSession session, ChatMessage message) {
        try {
            String json = objectMapper.writeValueAsString(message);
            session.sendMessage(new TextMessage(json));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        sessions.remove(session.getId());
    }

    public static class ChatMessage {
        private String type;  // CHAT, CHUNK, COMPLETE, ERROR, PING, PONG
        private String content;
        
        public ChatMessage() {
        }
        
        public ChatMessage(String type, String content) {
            this.type = type;
            this.content = content;
        }
        
        // getters, setters
        public String getType() {
            return type;
        }
        
        public void setType(String type) {
            this.type = type;
        }
        
        public String getContent() {
            return content;
        }
        
        public void setContent(String content) {
            this.content = content;
        }
    }
}
```

### WebSocket 클라이언트

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WebSocket Chat Demo</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        }
        
        #chat-container {
            display: flex;
            flex-direction: column;
            height: 70vh;
        }
        
        #messages {
            flex: 1;
            overflow-y: auto;
            border: 1px solid #ccc;
            padding: 10px;
            margin-bottom: 10px;
        }
        
        .message {
            margin-bottom: 10px;
            padding: 8px 12px;
            border-radius: 5px;
        }
        
        .user-message {
            background-color: #e3f2fd;
            align-self: flex-end;
            text-align: right;
        }
        
        .ai-message {
            background-color: #f1f1f1;
        }
        
        #input-container {
            display: flex;
        }
        
        #message-input {
            flex: 1;
            padding: 10px;
        }
        
        #send-button {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            cursor: pointer;
        }
        
        #status {
            margin-top: 10px;
            font-style: italic;
            color: #666;
        }
    </style>
</head>
<body>
    <h1>WebSocket AI Chat</h1>
    
    <div id="chat-container">
        <div id="messages"></div>
        
        <div id="input-container">
            <input id="message-input" type="text" placeholder="Type your message..." />
            <button id="send-button">Send</button>
        </div>
        
        <div id="status">Disconnected</div>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', function() {
            const messagesDiv = document.getElementById('messages');
            const messageInput = document.getElementById('message-input');
            const sendButton = document.getElementById('send-button');
            const statusDiv = document.getElementById('status');
            
            let socket;
            let isProcessing = false;
            let currentAiMessageDiv = null;
            
            // WebSocket 연결 설정
            function connectWebSocket() {
                socket = new WebSocket(`ws://${window.location.host}/ws/chat`);
                
                socket.onopen = function(event) {
                    statusDiv.textContent = 'Connected';
                    
                    // 연결 유지를 위한 핑-퐁
                    setInterval(() => {
                        if (socket.readyState === WebSocket.OPEN) {
                            socket.send(JSON.stringify({ type: 'PING' }));
                        }
                    }, 30000);
                };
                
                socket.onmessage = function(event) {
                    const message = JSON.parse(event.data);
                    
                    if (message.type === 'PONG') {
                        console.log('Received PONG');
                        return;
                    }
                    
                    if (message.type === 'PROCESSING') {
                        isProcessing = true;
                        statusDiv.textContent = 'AI is thinking...';
                        
                        // AI 응답을 위한 새 메시지 div 준비
                        currentAiMessageDiv = document.createElement('div');
                        currentAiMessageDiv.className = 'message ai-message';
                        messagesDiv.appendChild(currentAiMessageDiv);
                    }
                    
                    if (message.type === 'CHUNK') {
                        if (currentAiMessageDiv) {
                            currentAiMessageDiv.textContent += message.content;
                            messagesDiv.scrollTop = messagesDiv.scrollHeight;
                        }
                    }
                    
                    if (message.type === 'COMPLETE') {
                        isProcessing = false;
                        statusDiv.textContent = 'Connected';
                        
                        // 전체 응답으로 업데이트
                        if (currentAiMessageDiv) {
                            currentAiMessageDiv.textContent = message.content;
                            messagesDiv.scrollTop = messagesDiv.scrollHeight;
                            currentAiMessageDiv = null;
                        }
                    }
                    
                    if (message.type === 'ERROR') {
                        isProcessing = false;
                        statusDiv.textContent = 'Error: ' + message.content;
                        
                        if (currentAiMessageDiv) {
                            currentAiMessageDiv.textContent = 'Error: ' + message.content;
                            currentAiMessageDiv.style.color = 'red';
                            currentAiMessageDiv = null;
                        }
                    }
                };
                
                socket.onclose = function(event) {
                    statusDiv.textContent = 'Disconnected';
                    
                    // 자동 재연결
                    setTimeout(connectWebSocket, 3000);
                };
                
                socket.onerror = function(error) {
                    console.error('WebSocket Error:', error);
                    statusDiv.textContent = 'Connection Error';
                };
            }
            
            // 메시지 전송
            function sendMessage() {
                if (!socket || socket.readyState !== WebSocket.OPEN) {
                    statusDiv.textContent = 'Not connected to server';
                    return;
                }
                
                if (isProcessing) {
                    statusDiv.textContent = 'Please wait for AI to finish responding';
                    return;
                }
                
                const text = messageInput.value.trim();
                if (!text) return;
                
                // 사용자 메시지 표시
                const userMessageDiv = document.createElement('div');
                userMessageDiv.className = 'message user-message';
                userMessageDiv.textContent = text;
                messagesDiv.appendChild(userMessageDiv);
                messagesDiv.scrollTop = messagesDiv.scrollHeight;
                
                // 웹소켓으로 메시지 전송
                socket.send(JSON.stringify({
                    type: 'CHAT',
                    content: text
                }));
                
                // 입력 필드 초기화
                messageInput.value = '';
            }
            
            // 이벤트 리스너
            sendButton.addEventListener('click', sendMessage);
            
            messageInput.addEventListener('keydown', function(e) {
                if (e.key === 'Enter' && !e.shiftKey) {
                    e.preventDefault();
                    sendMessage();
                }
            });
            
            // 초기 연결
            connectWebSocket();
        });
    </script>
</body>
</html>
```

## 실제 사용 사례: 스트리밍 문서 요약기

지금까지 배운 스트리밍 응답과 비동기 처리 기법을 활용하여 실제 사용 사례를 구현해 보겠습니다.

### 문서 요약 서비스

```java
package com.example.streamingdemo.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import reactor.core.publisher.Flux;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;
import java.util.stream.Collectors;

@Service
public class DocumentSummaryService {

    private final ChatClient chatClient;

    public DocumentSummaryService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public Flux<SummaryResponse> streamingSummary(MultipartFile document) throws IOException {
        // 1. 문서 텍스트 추출
        String documentText = extractText(document);
        
        // 2. AI에게 요약 지시
        SystemMessage systemMessage = new SystemMessage(
                "You are a document summarization expert. Your task is to provide a concise summary of the provided document. " +
                "Include key points, main arguments, and important conclusions. " +
                "Structure your summary with clear sections and bullet points where appropriate."
        );
        
        UserMessage userMessage = new UserMessage("Document to summarize: " + documentText);
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        
        // 3. 진행 상태 및 최종 요약을 추적하기 위한 변수
        AtomicReference<Integer> progress = new AtomicReference<>(0);
        AtomicReference<StringBuilder> summaryBuilder = new AtomicReference<>(new StringBuilder());
        
        // 4. 스트리밍 응답 처리
        return chatClient.stream(prompt)
                .map(response -> {
                    String content = response.getResult().getOutput().getContent();
                    summaryBuilder.get().append(content);
                    
                    // 진행률 업데이트 (이 예제에서는 단순히 문자 수를 기준으로 추정)
                    // 실제 구현에서는 더 정교한 방법 사용 가능
                    int estimatedTotalLength = documentText.length() / 4;  // 요약은 원본의 약 1/4 길이로 가정
                    int currentLength = summaryBuilder.get().length();
                    int newProgress = Math.min(95, (currentLength * 100) / estimatedTotalLength);
                    progress.getAndSet(newProgress);
                    
                    return new SummaryResponse(
                            content,
                            summaryBuilder.get().toString(),
                            progress.get(),
                            false
                    );
                })
                // 완료 응답 추가
                .concatWith(Flux.just(
                        new SummaryResponse(
                                "",
                                summaryBuilder.get().toString(),
                                100,
                                true
                        )
                ));
    }

    private String extractText(MultipartFile document) throws IOException {
        // 간단한 텍스트 파일 읽기 (실제 구현에서는 PDF, DOCX 등 다양한 형식 지원 필요)
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(document.getInputStream()))) {
            return reader.lines().collect(Collectors.joining("\n"));
        }
    }

    public static class SummaryResponse {
        private String delta;
        private String fullSummary;
        private int progress;
        private boolean done;
        
        public SummaryResponse(String delta, String fullSummary, int progress, boolean done) {
            this.delta = delta;
            this.fullSummary = fullSummary;
            this.progress = progress;
            this.done = done;
        }
        
        // getters
        public String getDelta() {
            return delta;
        }
        
        public String getFullSummary() {
            return fullSummary;
        }
        
        public int getProgress() {
            return progress;
        }
        
        public boolean isDone() {
            return done;
        }
    }
}
```

### 문서 요약 컨트롤러

```java
package com.example.streamingdemo.controller;

import com.example.streamingdemo.service.DocumentSummaryService;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;
import reactor.core.publisher.Flux;

import java.io.IOException;

@RestController
@RequestMapping("/api/documents")
public class DocumentSummaryController {

    private final DocumentSummaryService documentSummaryService;

    public DocumentSummaryController(DocumentSummaryService documentSummaryService) {
        this.documentSummaryService = documentSummaryService;
    }

    @PostMapping(value = "/summarize", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<DocumentSummaryService.SummaryResponse> summarizeDocument(
            @RequestParam("document") MultipartFile document) throws IOException {
        
        return documentSummaryService.streamingSummary(document);
    }
}
```

### 문서 요약 웹 인터페이스

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document Summarizer</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            max-width: 900px;
            margin: 0 auto;
            padding: 20px;
            color: #333;
        }
        
        h1 {
            text-align: center;
            margin-bottom: 30px;
            color: #2c3e50;
        }
        
        .container {
            display: flex;
            flex-direction: column;
            gap: 20px;
        }
        
        .drop-area {
            border: 2px dashed #ccc;
            border-radius: 8px;
            padding: 40px;
            text-align: center;
            background-color: #f9f9f9;
            cursor: pointer;
        }
        
        .drop-area.highlight {
            border-color: #3498db;
            background-color: #e3f2fd;
        }
        
        .file-input {
            display: none;
        }
        
        .button {
            padding: 12px 24px;
            background-color: #3498db;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
            transition: background-color 0.3s;
        }
        
        .button:hover {
            background-color: #2980b9;
        }
        
        .button:disabled {
            background-color: #95a5a6;
            cursor: not-allowed;
        }
        
        .progress-container {
            margin-top: 20px;
            display: none;
        }
        
        .progress-bar {
            height: 10px;
            background-color: #ecf0f1;
            border-radius: 5px;
            overflow: hidden;
        }
        
        .progress-fill {
            height: 100%;
            background-color: #3498db;
            border-radius: 5px;
            width: 0%;
            transition: width 0.3s ease;
        }
        
        .progress-text {
            text-align: center;
            margin-top: 5px;
            color: #7f8c8d;
        }
        
        .result-container {
            margin-top: 30px;
            border: 1px solid #ddd;
            border-radius: 8px;
            padding: 20px;
            min-height: 300px;
            background-color: white;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
            display: none;
        }
        
        .result-title {
            margin-top: 0;
            color: #2c3e50;
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
        }
        
        .summary-content {
            white-space: pre-wrap;
            line-height: 1.8;
        }
    </style>
</head>
<body>
    <h1>AI Document Summarizer</h1>
    
    <div class="container">
        <div id="drop-area" class="drop-area">
            <p>Drag & drop your document here or</p>
            <input type="file" id="file-input" class="file-input" accept=".txt,.pdf,.docx">
            <button id="browse-button" class="button">Browse Files</button>
            <p id="file-name"></p>
        </div>
        
        <button id="summarize-button" class="button" disabled>Summarize Document</button>
        
        <div id="progress-container" class="progress-container">
            <div class="progress-bar">
                <div id="progress-fill" class="progress-fill"></div>
            </div>
            <div id="progress-text" class="progress-text">Processing: 0%</div>
        </div>
        
        <div id="result-container" class="result-container">
            <h2 class="result-title">Document Summary</h2>
            <div id="summary-content" class="summary-content"></div>
        </div>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', function() {
            const dropArea = document.getElementById('drop-area');
            const fileInput = document.getElementById('file-input');
            const browseButton = document.getElementById('browse-button');
            const fileName = document.getElementById('file-name');
            const summarizeButton = document.getElementById('summarize-button');
            const progressContainer = document.getElementById('progress-container');
            const progressFill = document.getElementById('progress-fill');
            const progressText = document.getElementById('progress-text');
            const resultContainer = document.getElementById('result-container');
            const summaryContent = document.getElementById('summary-content');
            
            let selectedFile = null;
            
            // 드래그 앤 드롭 이벤트 리스너
            ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
                dropArea.addEventListener(eventName, preventDefaults, false);
            });
            
            function preventDefaults(e) {
                e.preventDefault();
                e.stopPropagation();
            }
            
            ['dragenter', 'dragover'].forEach(eventName => {
                dropArea.addEventListener(eventName, highlight, false);
            });
            
            ['dragleave', 'drop'].forEach(eventName => {
                dropArea.addEventListener(eventName, unhighlight, false);
            });
            
            function highlight() {
                dropArea.classList.add('highlight');
            }
            
            function unhighlight() {
                dropArea.classList.remove('highlight');
            }
            
            // 파일 드롭 처리
            dropArea.addEventListener('drop', handleDrop, false);
            
            function handleDrop(e) {
                const dt = e.dataTransfer;
                const files = dt.files;
                
                if (files.length > 0) {
                    handleFiles(files);
                }
            }
            
            // 파일 선택 처리
            fileInput.addEventListener('change', function() {
                if (fileInput.files.length > 0) {
                    handleFiles(fileInput.files);
                }
            });
            
            // 파일 찾아보기 버튼
            browseButton.addEventListener('click', function() {
                fileInput.click();
            });
            
            function handleFiles(files) {
                selectedFile = files[0];
                fileName.textContent = 'Selected: ' + selectedFile.name;
                summarizeButton.disabled = false;
            }
            
            // 요약 버튼 클릭
            summarizeButton.addEventListener('click', summarizeDocument);
            
            function summarizeDocument() {
                if (!selectedFile) {
                    alert('Please select a document first');
                    return;
                }
                
                // UI 업데이트
                summarizeButton.disabled = true;
                progressContainer.style.display = 'block';
                resultContainer.style.display = 'block';
                summaryContent.textContent = '';
                
                // FormData 생성
                const formData = new FormData();
                formData.append('document', selectedFile);
                
                // 스트리밍 요청
                const eventSource = new EventSource('/api/documents/summarize?' + new URLSearchParams({
                    documentName: selectedFile.name
                }));
                
                eventSource.onmessage = function(event) {
                    const data = JSON.parse(event.data);
                    
                    // 진행 상태 업데이트
                    progressFill.style.width = data.progress + '%';
                    progressText.textContent = 'Processing: ' + data.progress + '%';
                    
                    // 요약 콘텐츠 업데이트
                    summaryContent.textContent = data.fullSummary;
                    
                    // 완료 시 연결 종료
                    if (data.done) {
                        eventSource.close();
                        summarizeButton.disabled = false;
                    }
                };
                
                eventSource.onerror = function(error) {
                    console.error('SSE Error:', error);
                    eventSource.close();
                    progressText.textContent = 'Error: Could not process the document';
                    summarizeButton.disabled = false;
                };
            }
        });
    </script>
</body>
</html>
```

## 결론

이 장에서는 Spring AI를 사용하여 스트리밍 응답과 비동기 처리를 구현하는 방법을 살펴보았습니다. Spring WebFlux와 Project Reactor를 활용하여 효율적이고 반응형 AI 애플리케이션을 구축하는 다양한 패턴과 기법을 배웠습니다.

주요 내용 요약:

1. **스트리밍 응답의 중요성**: AI 모델의 응답을 실시간으로 스트리밍하면 사용자 경험이 크게 향상됩니다.
2. **Spring WebFlux와 Project Reactor**: Mono와 Flux 타입을 활용한 비동기, 논블로킹 프로그래밍은 확장 가능한 AI 애플리케이션을 구축하는 데 필수적입니다.
3. **다양한 비동기 패턴**: 병렬 처리, 콜백, 장시간 실행 작업 처리를 위한 다양한 패턴을 알아보았습니다.
4. **WebSocket 통합**: 양방향 실시간 통신을 통해 대화형 AI 경험을 제공하는 방법을 살펴보았습니다.
5. **실제 사용 사례**: 문서 요약기 애플리케이션을 통해 스트리밍과 비동기 기법을 실제로 적용해 보았습니다.

Spring AI의 스트리밍 및 비동기 기능을 활용하면 응답성이 뛰어나고 사용자 친화적인 AI 애플리케이션을 구축할 수 있습니다. 특히 챗봇, 콘텐츠 생성, 실시간 분석과 같이 즉각적인 피드백이 중요한 시나리오에서 더욱 가치가 있습니다.

다음 장에서는 AI 애플리케이션 테스트 전략에 대해 알아보고, 모의 객체(mock)와 녹화/재생 기법을 활용하여 안정적이고 예측 가능한 AI 애플리케이션을 구축하는 방법을 살펴보겠습니다.