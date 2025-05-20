# AI 애플리케이션 테스트 전략

## 개요

AI 애플리케이션 테스팅은 전통적인 소프트웨어 테스팅과는 다른 접근 방식이 필요합니다. AI 모델의 비결정적 특성, 외부 API 의존성, 입력과 출력의 다양성 등이 테스트를 복잡하게 만듭니다. 이 장에서는 Spring AI를 사용하는 애플리케이션의 효과적인 테스트 전략과 방법론을 살펴보고, 특히 Spring AI에서 제공하는 목(Mock) 기능과 녹화/재생 메커니즘을 활용하는 방법에 대해 자세히 알아보겠습니다.

## AI 애플리케이션 테스트의 과제

### 전통적인 테스트와의 차이점

AI 애플리케이션 테스트는 다음과 같은 이유로 전통적인 소프트웨어 테스트와 다릅니다:

1. **비결정성**: 동일한 입력에도 AI 모델은 다양한 출력을 생성할 수 있습니다.
2. **외부 API 의존성**: 대부분의 AI 기능은 OpenAI, Anthropic 등의 외부 서비스에 의존합니다.
3. **비용**: 실제 API 호출은 비용이 발생하므로 테스트 비용이 상당히 증가할 수 있습니다.
4. **지연 시간**: API 호출은 응답 시간이 길어 테스트 속도가 느려질 수 있습니다.
5. **서비스 가용성**: 외부 서비스의 중단이나 변경에 취약합니다.
6. **복잡한 응답 구조**: 응답이 자유 형식 텍스트이거나 복잡한 구조를 가질 수 있습니다.

### 테스트 전략의 주요 목표

AI 애플리케이션 테스트 전략은 다음 목표를 달성해야 합니다:

1. **재현 가능성**: 테스트 결과가 예측 가능하고 일관되어야 합니다.
2. **독립성**: 외부 서비스에 의존하지 않고 테스트를 실행할 수 있어야 합니다.
3. **속도**: 테스트가 빠르게 실행되어야 개발 생산성이 향상됩니다.
4. **비용 효율성**: 불필요한 API 호출 비용을 최소화해야 합니다.
5. **포괄성**: 애플리케이션의 모든 중요 경로를 테스트해야 합니다.
6. **유지 관리성**: 테스트 코드가 쉽게 유지 관리되어야 합니다.

## Spring AI의 테스트 지원 기능

Spring AI는 AI 애플리케이션 테스트를 위한 다양한 기능을 제공합니다. 가장 두드러진 기능은 다음과 같습니다:

### Mock AI 클라이언트

Spring AI는 실제 AI 서비스와 상호 작용하지 않고 테스트할 수 있는 모의(Mock) 클라이언트를 제공합니다. 이 클라이언트는 실제 API 구현체를 대체하며, 미리 정의된 응답을 반환합니다.

### 응답 녹화 및 재생

Spring AI는 실제 AI 모델 응답을 녹화하고, 이후 테스트에서 이를 재생할 수 있는 메커니즘을 제공합니다. 이 기능은 실제 응답과 유사한 테스트 환경을 제공하면서도 외부 의존성을 제거합니다.

### 테스트 유틸리티

Spring AI의 `spring-ai-test` 모듈은 테스트를 쉽게 작성할 수 있는 다양한 유틸리티와 도우미 메서드를 제공합니다.

## Mock 클라이언트 활용하기

### Mock 클라이언트 설정

Spring AI의 Mock 클라이언트를 설정하는 방법을 살펴보겠습니다. 먼저 필요한 의존성을 추가해야 합니다:

```kotlin
// Gradle - Kotlin DSL
testImplementation("org.springframework.ai:spring-ai-test:1.0.0-M6")
```

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-test</artifactId>
    <version>1.0.0-M6</version>
    <scope>test</scope>
</dependency>
```

이제 Mock 클라이언트를 설정해 보겠습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.openai.api.OpenAiApi;
import org.springframework.ai.openai.chat.OpenAiChatClient;
import org.springframework.ai.openai.chat.OpenAiChatOptions;
import org.springframework.ai.test.chat.MockChatClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.context.annotation.Profile;

import java.util.Map;

@Configuration
public class TestConfig {
    
    @Bean
    @Profile("test")
    @Primary
    public ChatClient mockChatClient() {
        // 기본 응답 정의
        return new MockChatClient("이것은 Mock ChatClient에서 생성된 응답입니다.");
    }
    
    // 더 복잡한 Mock 응답 정의
    @Bean
    @Profile("test-advanced")
    @Primary
    public ChatClient advancedMockChatClient() {
        // 프롬프트 패턴과 응답을 매핑
        Map<String, String> promptResponses = Map.of(
            "날씨", "오늘은 맑고 화창한 날씨입니다.",
            "시간", "현재 시간은 오후 3시입니다.",
            "안녕", "안녕하세요! 무엇을 도와드릴까요?"
        );
        
        return new MockChatClient(prompt -> {
            String userInput = "";
            
            // 프롬프트에서 사용자 메시지 추출
            for (Message message : prompt.getMessages()) {
                if (message instanceof UserMessage) {
                    userInput = ((UserMessage) message).getContent();
                    break;
                }
            }
            
            // 패턴 매칭 응답
            for (Map.Entry<String, String> entry : promptResponses.entrySet()) {
                if (userInput.contains(entry.getKey())) {
                    return entry.getValue();
                }
            }
            
            // 기본 응답
            return "죄송합니다, 이해하지 못했습니다.";
        });
    }
}
```

### 단위 테스트에서 Mock 클라이언트 사용

이제 단위 테스트에서 Mock 클라이언트를 사용하는 방법을 살펴보겠습니다:

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.test.chat.MockChatClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ActiveProfiles("test")
public class ChatServiceTest {

    @Autowired
    private ChatClient chatClient;
    
    @Autowired
    private ChatService chatService;

    @Test
    void basicMockTest() {
        // given
        String question = "Spring AI에 대해 알려주세요";
        
        // when
        String response = chatService.ask(question);
        
        // then
        assertThat(response).isEqualTo("이것은 Mock ChatClient에서 생성된 응답입니다.");
    }
    
    @Test
    void customMockTest() {
        // given
        String expectedResponse = "Spring AI는 다양한 AI 서비스를 통합하는 프레임워크입니다.";
        MockChatClient mockChatClient = new MockChatClient(expectedResponse);
        ChatService customChatService = new ChatService(mockChatClient);
        
        // when
        String response = customChatService.ask("Spring AI란 무엇인가요?");
        
        // then
        assertThat(response).isEqualTo(expectedResponse);
    }
}

// 테스트 대상 서비스
class ChatService {
    private final ChatClient chatClient;
    
    public ChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String ask(String question) {
        Prompt prompt = new Prompt(new UserMessage(question));
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
}
```

### 조건부 응답을 위한 고급 Mock 설정

더 복잡한 시나리오를 테스트하기 위해 조건부 응답을 설정할 수 있습니다:

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.test.chat.MockChatClient;

import java.util.function.Function;

import static org.assertj.core.api.Assertions.assertThat;

public class AdvancedMockTest {

    @Test
    void conditionalResponseTest() {
        // 응답 선택 함수 정의
        Function<Prompt, String> responseFunction = prompt -> {
            String userInput = "";
            
            for (Message message : prompt.getMessages()) {
                if (message instanceof UserMessage) {
                    userInput = ((UserMessage) message).getContent().toLowerCase();
                    break;
                }
            }
            
            if (userInput.contains("날씨")) {
                return "오늘은 맑고 화창합니다.";
            } else if (userInput.contains("시간")) {
                return "현재 시간은 오후 3시입니다.";
            } else if (userInput.contains("안녕")) {
                return "안녕하세요! 무엇을 도와드릴까요?";
            }
            
            return "죄송합니다, 이해하지 못했습니다.";
        };
        
        // Mock 클라이언트 생성
        MockChatClient mockChatClient = new MockChatClient(responseFunction);
        ChatService chatService = new ChatService(mockChatClient);
        
        // 다양한 입력에 대한 응답 테스트
        assertThat(chatService.ask("오늘 날씨는 어때요?")).isEqualTo("오늘은 맑고 화창합니다.");
        assertThat(chatService.ask("지금 몇 시인가요?")).isEqualTo("현재 시간은 오후 3시입니다.");
        assertThat(chatService.ask("안녕하세요")).isEqualTo("안녕하세요! 무엇을 도와드릴까요?");
        assertThat(chatService.ask("스프링이란?")).isEqualTo("죄송합니다, 이해하지 못했습니다.");
    }
}
```

## 녹화 및 재생 메커니즘

Spring AI는 실제 AI 모델 응답을 녹화하고 이후 테스트에서 재생할 수 있는 강력한 메커니즘을 제공합니다. 이 기능은 다음과 같은 이점이 있습니다:

1. 실제 AI 응답과 동일한 테스트 데이터 사용
2. 외부 API 의존성 제거
3. 빠른 테스트 실행
4. 일관된 테스트 결과
5. API 비용 절감

### 응답 녹화하기

먼저 응답을 녹화하는 방법을 살펴보겠습니다:

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.recorder.RecordingChatClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Primary;
import org.springframework.test.context.ActiveProfiles;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

@SpringBootTest
@ActiveProfiles("record")
public class RecordingTest {

    @Autowired
    private ChatClient chatClient;
    
    @Test
    void recordResponses() {
        // 다양한 프롬프트로 응답 녹화
        chatClient.call(new Prompt(new UserMessage("Spring AI란 무엇인가요?")));
        chatClient.call(new Prompt(new UserMessage("Spring AI의 주요 기능은 무엇인가요?")));
        chatClient.call(new Prompt(new UserMessage("Spring AI를 사용하는 이점은 무엇인가요?")));
        
        // 녹화 파일이 생성되었는지 확인
        Path recordingsPath = Paths.get("src/test/resources/recordings");
        try {
            assertThat(Files.list(recordingsPath).count()).isGreaterThan(0);
        } catch (IOException e) {
            throw new RuntimeException("녹화 파일 확인 중 오류 발생", e);
        }
    }
    
    @TestConfiguration
    static class Config {
        
        @Bean
        @Primary
        public ChatClient recordingChatClient(@Autowired ChatClient originalChatClient) throws IOException {
            // 녹화 디렉토리 생성
            Path recordingsPath = Paths.get("src/test/resources/recordings");
            if (!Files.exists(recordingsPath)) {
                Files.createDirectories(recordingsPath);
            }
            
            // 녹화 클라이언트 생성
            return new RecordingChatClient(originalChatClient, recordingsPath.toFile());
        }
    }
}
```

### 응답 재생하기

녹화된 응답을 재생하는 방법을 살펴보겠습니다:

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.recorder.PlaybackChatClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Primary;
import org.springframework.test.context.ActiveProfiles;

import java.io.File;
import java.nio.file.Paths;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ActiveProfiles("playback")
public class PlaybackTest {

    @Autowired
    private ChatClient chatClient;
    
    @Autowired
    private ChatService chatService;
    
    @Test
    void playbackRecordedResponses() {
        // 녹화된 응답 재생
        String response1 = chatService.ask("Spring AI란 무엇인가요?");
        String response2 = chatService.ask("Spring AI의 주요 기능은 무엇인가요?");
        String response3 = chatService.ask("Spring AI를 사용하는 이점은 무엇인가요?");
        
        // 응답 확인 (실제 응답 내용은 녹화 시 받은 내용에 따라 다름)
        assertThat(response1).isNotEmpty();
        assertThat(response2).isNotEmpty();
        assertThat(response3).isNotEmpty();
    }
    
    @TestConfiguration
    static class Config {
        
        @Bean
        @Primary
        public ChatClient playbackChatClient() {
            // 재생 클라이언트 생성
            File recordingsDir = Paths.get("src/test/resources/recordings").toFile();
            return new PlaybackChatClient(recordingsDir);
        }
    }
}
```

### 녹화 및 재생 옵션 구성

녹화 및 재생 동작을 세부적으로 구성할 수 있습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.recorder.RecordingChatClient;
import org.springframework.ai.recorder.RecordingMode;
import org.springframework.ai.recorder.PlaybackChatClient;
import org.springframework.ai.recorder.PlaybackMode;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.context.annotation.Profile;

import java.io.File;
import java.nio.file.Paths;

@Configuration
public class RecorderConfig {

    @Bean
    @Profile("record-strict")
    @Primary
    public ChatClient strictRecordingChatClient(ChatClient originalChatClient) {
        File recordingsDir = Paths.get("src/test/resources/recordings").toFile();
        
        return RecordingChatClient.builder(originalChatClient, recordingsDir)
                .withRecordingMode(RecordingMode.STRICT) // 모든 요청을 녹화
                .withRequestMatchStrategy((req1, req2) -> {
                    // 커스텀 요청 매칭 전략
                    // 예: 타임스탬프 또는 임의 ID와 같은 비결정적 요소 무시
                    return req1.getMessages().equals(req2.getMessages());
                })
                .build();
    }
    
    @Bean
    @Profile("playback-lenient")
    @Primary
    public ChatClient lenientPlaybackChatClient() {
        File recordingsDir = Paths.get("src/test/resources/recordings").toFile();
        
        return PlaybackChatClient.builder(recordingsDir)
                .withPlaybackMode(PlaybackMode.LENIENT) // 매치하지 않는 요청에 대해 경고만 하고 계속 진행
                .withPlaybackStrategy((request, recordings) -> {
                    // 커스텀 재생 전략
                    // 예: 가장 유사한 녹화 찾기
                    return recordings.stream()
                            .filter(r -> r.getRequest().toString().contains(request.toString()))
                            .findFirst();
                })
                .build();
    }
}
```

### 녹화 및 재생을 위한 실용적인 패턴

실제 프로젝트에서 녹화 및 재생을 관리하는 몇 가지 유용한 패턴을 살펴보겠습니다:

```java
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.recorder.RecordingChatClient;
import org.springframework.ai.recorder.PlaybackChatClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@ExtendWith(SpringExtension.class)
@SpringBootTest
public class AiRecorderPatternTest {

    private static final String RECORDING_DIR = "src/test/resources/recordings";
    
    @Autowired
    private ChatClient originalChatClient;
    
    private ChatClient testChatClient;
    private File testRecordingDir;
    
    @BeforeEach
    void setUp() throws Exception {
        // 테스트별 고유 녹화 디렉토리 생성
        String testName = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
        testRecordingDir = Paths.get(RECORDING_DIR, testName).toFile();
        Files.createDirectories(testRecordingDir.toPath());
        
        // 녹화 모드 또는 재생 모드 결정
        boolean recordMode = Boolean.parseBoolean(System.getProperty("ai.test.record", "false"));
        
        if (recordMode) {
            testChatClient = new RecordingChatClient(originalChatClient, testRecordingDir);
        } else {
            // 녹화 디렉토리 존재 여부 확인
            if (!testRecordingDir.exists() || testRecordingDir.listFiles() == null || testRecordingDir.listFiles().length == 0) {
                // 녹화 없으면 새로 생성
                testChatClient = new RecordingChatClient(originalChatClient, testRecordingDir);
            } else {
                // 기존 녹화 재생
                testChatClient = new PlaybackChatClient(testRecordingDir);
            }
        }
    }
    
    @Test
    void testChatResponse() {
        // 테스트 로직
        Prompt prompt = new Prompt(new UserMessage("Spring AI를 사용하는 이점은 무엇인가요?"));
        String response = testChatClient.call(prompt).getResult().getOutput().getContent();
        
        // 응답 확인
        assertThat(response).contains("Spring");
    }
    
    @AfterEach
    void tearDown() {
        // 정리 작업 (필요한 경우)
    }
}
```

## 통합 테스트 전략

Spring AI 애플리케이션의 전체 흐름을 테스트하기 위한 통합 테스트 전략을 살펴보겠습니다.

### 테스트 컨텍스트 구성

먼저 테스트 컨텍스트를 구성하는 방법을 살펴보겠습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.openai.OpenAiChatOptions;
import org.springframework.ai.openai.chat.OpenAiChatClient;
import org.springframework.ai.test.chat.MockChatClient;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Primary;
import org.springframework.context.annotation.Profile;

@TestConfiguration
public class AiTestConfig {

    @Bean
    @Primary
    @Profile("test")
    public ChatClient mockChatClient() {
        return new MockChatClient("통합 테스트를 위한 Mock 응답입니다.");
    }
    
    @Bean
    @Primary
    @Profile("test-integration")
    public ChatClient realChatClientWithTestModel() {
        // 통합 테스트용 저비용 모델 설정
        OpenAiChatOptions options = OpenAiChatOptions.builder()
                .withModel("gpt-3.5-turbo") // 저비용 모델 사용
                .withMaxTokens(100)         // 응답 길이 제한
                .withTemperature(0.0f)      // 일관된 결과를 위한 설정
                .build();
        
        return new OpenAiChatClient(API_KEY, options);
    }
}
```

### 웹 계층 통합 테스트

SpringMVC 테스트 프레임워크를 사용한 웹 계층 통합 테스트 예제입니다:

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(ChatController.class)
@Import(AiTestConfig.class)
@ActiveProfiles("test")
public class ChatControllerTest {

    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private ChatService chatService;
    
    @Test
    void testChatEndpoint() throws Exception {
        // given
        when(chatService.chat(any())).thenReturn("Spring AI에 대한 응답입니다.");
        
        // when & then
        mockMvc.perform(post("/api/chat")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"message\":\"Spring AI란 무엇인가요?\"}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.response").value("Spring AI에 대한 응답입니다."));
    }
}
```

### 전체 애플리케이션 통합 테스트

애플리케이션의 전체 흐름을 테스트하는 통합 테스트 예제입니다:

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
public class ChatApplicationIntegrationTest {

    @LocalServerPort
    private int port;
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void testChatFlow() {
        // given
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        String requestBody = "{\"message\":\"Spring AI란 무엇인가요?\"}";
        HttpEntity<String> request = new HttpEntity<>(requestBody, headers);
        
        // when
        ChatResponse response = restTemplate.postForObject(
                "http://localhost:" + port + "/api/chat",
                request,
                ChatResponse.class);
        
        // then
        assertThat(response).isNotNull();
        assertThat(response.getResponse()).isNotEmpty();
    }
    
    static class ChatResponse {
        private String response;
        
        // getters, setters
        public String getResponse() {
            return response;
        }
        
        public void setResponse(String response) {
            this.response = response;
        }
    }
}
```

## 함수 호출(Function Calling) 테스팅

Spring AI의 함수 호출 기능을 테스트하는 방법을 살펴보겠습니다.

### Mock 함수 호출 응답 설정

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.tool.Tool;
import org.springframework.ai.chat.tool.ToolParameterValue;
import org.springframework.ai.chat.tool.ToolSpec;
import org.springframework.ai.chat.tool.ToolUtils;
import org.springframework.ai.chat.tool.DefaultToolParameterValue;
import org.springframework.ai.test.chat.MockChatClient;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

public class FunctionCallingTest {

    @Test
    void testFunctionCallingWithMock() {
        // Function Calling 응답을 가진 Mock 클라이언트 생성
        MockChatClient mockChatClient = new MockChatClient(prompt -> {
            // 사용자 메시지 확인
            String userMessage = prompt.getMessages().stream()
                    .filter(m -> m instanceof UserMessage)
                    .map(m -> ((UserMessage) m).getContent())
                    .findFirst()
                    .orElse("");
            
            // 날씨 정보 요청 감지
            if (userMessage.toLowerCase().contains("날씨")) {
                // 함수 호출 응답 생성
                ToolSpec weatherTool = new ToolSpec(
                        "getWeather",
                        "Get current weather information",
                        Map.of(
                                "location", Map.of(
                                        "type", "string",
                                        "description", "City name"
                                )
                        )
                );
                
                // 함수 호출 파라미터 설정
                Map<String, ToolParameterValue> parameters = Map.of(
                        "location", new DefaultToolParameterValue("서울")
                );
                
                // 함수 호출 응답 생성
                return ToolUtils.convertToolCallToText(
                        "getWeather",
                        weatherTool,
                        parameters
                );
            }
            
            // 기본 응답
            return "무엇을 도와드릴까요?";
        });
        
        // 서비스에 Mock 클라이언트 주입
        WeatherService weatherService = new WeatherService(mockChatClient);
        
        // 테스트 실행
        WeatherInfo weatherInfo = weatherService.getWeatherInfo("서울의 날씨는 어때요?");
        
        // 결과 확인
        assertThat(weatherInfo).isNotNull();
        assertThat(weatherInfo.getLocation()).isEqualTo("서울");
    }
    
    // 테스트 대상 서비스
    static class WeatherService {
        private final ChatClient chatClient;
        
        public WeatherService(ChatClient chatClient) {
            this.chatClient = chatClient;
        }
        
        public WeatherInfo getWeatherInfo(String query) {
            Prompt prompt = new Prompt(new UserMessage(query));
            
            // 함수 호출 도구 정의
            List<Tool> tools = List.of(
                    new Tool(
                            "getWeather",
                            "Get current weather information",
                            Map.of(
                                    "location", Map.of(
                                            "type", "string",
                                            "description", "City name"
                                    )
                            )
                    )
            );
            
            // 함수 호출 처리
            var response = chatClient.call(prompt, tools);
            var toolCall = response.getToolCalls().get(0);
            
            // 함수 호출 결과 파싱
            String location = (String) toolCall.getArguments().get("location");
            
            // 실제 구현에서는 외부 날씨 API 호출
            return new WeatherInfo(location, "맑음", 25.5);
        }
    }
    
    static class WeatherInfo {
        private final String location;
        private final String condition;
        private final double temperature;
        
        public WeatherInfo(String location, String condition, double temperature) {
            this.location = location;
            this.condition = condition;
            this.temperature = temperature;
        }
        
        // getters
        public String getLocation() {
            return location;
        }
        
        public String getCondition() {
            return condition;
        }
        
        public double getTemperature() {
            return temperature;
        }
    }
}
```

## 성능 및 로드 테스트

AI 애플리케이션의 성능과 확장성을 테스트하는 방법을 살펴보겠습니다.

### AI 응답 지연 시뮬레이션

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.test.chat.MockChatClient;

import java.time.Duration;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertTimeoutPreemptively;

public class PerformanceTest {

    @Test
    void testResponseTime() {
        // 지연 시간이 있는 Mock 클라이언트 생성
        MockChatClient mockChatClient = new MockChatClient(prompt -> {
            try {
                // 2초 지연 시뮬레이션
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "지연된 응답입니다.";
        });
        
        // 테스트 대상 서비스
        ChatService chatService = new ChatService(mockChatClient);
        
        // 타임아웃이 있는 테스트 (5초)
        assertTimeoutPreemptively(Duration.ofSeconds(5), () -> {
            String response = chatService.ask("테스트 질문");
            assertThat(response).isEqualTo("지연된 응답입니다.");
        });
    }
    
    @Test
    void testTimeout() {
        // 매우 긴 지연이 있는 Mock 클라이언트
        MockChatClient mockChatClient = new MockChatClient(prompt -> {
            try {
                // 10초 지연 시뮬레이션
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "매우 지연된 응답입니다.";
        });
        
        // 타임아웃 처리가 있는 서비스
        TimeoutChatService chatService = new TimeoutChatService(mockChatClient, 3);
        
        // 서비스 호출
        String response = chatService.askWithTimeout("타임아웃 테스트");
        
        // 타임아웃으로 인한 기본 응답 확인
        assertThat(response).isEqualTo("응답 시간이 초과되었습니다.");
    }
    
    // 타임아웃 처리가 있는 서비스
    static class TimeoutChatService {
        private final ChatClient chatClient;
        private final long timeoutSeconds;
        
        public TimeoutChatService(ChatClient chatClient, long timeoutSeconds) {
            this.chatClient = chatClient;
            this.timeoutSeconds = timeoutSeconds;
        }
        
        public String askWithTimeout(String question) {
            CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
                Prompt prompt = new Prompt(new UserMessage(question));
                return chatClient.call(prompt).getResult().getOutput().getContent();
            });
            
            try {
                return future.get(timeoutSeconds, TimeUnit.SECONDS);
            } catch (InterruptedException | ExecutionException | TimeoutException e) {
                return "응답 시간이 초과되었습니다.";
            }
        }
    }
}
```

### 병렬 요청 처리 테스트

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.test.chat.MockChatClient;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.stream.Collectors;
import java.util.stream.IntStream;

import static org.assertj.core.api.Assertions.assertThat;

public class ConcurrencyTest {

    @Test
    void testConcurrentRequests() throws Exception {
        // Random 지연이 있는 Mock 클라이언트
        MockChatClient mockChatClient = new MockChatClient(prompt -> {
            try {
                // 100ms ~ 1000ms 랜덤 지연
                Thread.sleep((long) (Math.random() * 900 + 100));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "응답 #" + System.currentTimeMillis() % 1000;
        });
        
        // 테스트 대상 서비스
        ChatService chatService = new ChatService(mockChatClient);
        
        // 동시 요청 수
        int concurrentRequests = 10;
        
        // 스레드 풀 생성
        ExecutorService executorService = Executors.newFixedThreadPool(concurrentRequests);
        
        // 동시 요청 실행
        List<CompletableFuture<String>> futures = new ArrayList<>();
        
        for (int i = 0; i < concurrentRequests; i++) {
            final int requestId = i;
            CompletableFuture<String> future = CompletableFuture.supplyAsync(
                    () -> chatService.ask("동시 요청 #" + requestId),
                    executorService
            );
            futures.add(future);
        }
        
        // 모든 요청 결과 수집
        CompletableFuture<Void> allFutures = CompletableFuture.allOf(
                futures.toArray(new CompletableFuture[0])
        );
        
        // 모든 응답 대기
        allFutures.join();
        
        // 결과 확인
        List<String> responses = futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
        
        // 모든 응답 검증
        assertThat(responses).hasSize(concurrentRequests);
        assertThat(responses).allMatch(r -> r.startsWith("응답 #"));
        
        // 스레드 풀 종료
        executorService.shutdown();
    }
    
    @Test
    void testRateLimiting() {
        // 요청 수 카운팅 Mock 클라이언트
        int[] requestCount = {0};
        
        MockChatClient mockChatClient = new MockChatClient(prompt -> {
            requestCount[0]++;
            return "응답 #" + requestCount[0];
        });
        
        // 속도 제한 서비스
        RateLimitedChatService chatService = new RateLimitedChatService(mockChatClient, 5); // 최대 5개 요청
        
        // 10개 요청 시도
        List<String> responses = IntStream.range(0, 10)
                .mapToObj(i -> chatService.ask("요청 #" + i))
                .collect(Collectors.toList());
        
        // 처리된 요청 수 확인
        assertThat(requestCount[0]).isEqualTo(5); // 최대 5개만 처리됨
        
        // 응답 확인
        assertThat(responses)
                .filteredOn(r -> r.startsWith("응답 #")).hasSize(5); // 정상 응답 5개
        
        assertThat(responses)
                .filteredOn(r -> r.equals("요청 한도를 초과했습니다.")).hasSize(5); // 제한 메시지 5개
    }
    
    // 속도 제한 서비스
    static class RateLimitedChatService {
        private final ChatClient chatClient;
        private final int maxRequests;
        private int requestCount = 0;
        
        public RateLimitedChatService(ChatClient chatClient, int maxRequests) {
            this.chatClient = chatClient;
            this.maxRequests = maxRequests;
        }
        
        public synchronized String ask(String question) {
            if (requestCount >= maxRequests) {
                return "요청 한도를 초과했습니다.";
            }
            
            requestCount++;
            Prompt prompt = new Prompt(new UserMessage(question));
            return chatClient.call(prompt).getResult().getOutput().getContent();
        }
    }
}
```

## 실제 사용 사례: RAG 시스템 테스트

마지막으로, Retrieval-Augmented Generation(RAG) 시스템과 같은 복잡한 AI 애플리케이션을 테스트하는 방법을 살펴보겠습니다.

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.document.Document;
import org.springframework.ai.test.chat.MockChatClient;
import org.springframework.ai.vectorstore.VectorStore;

import java.util.List;
import java.util.function.Function;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyInt;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

public class RagSystemTest {

    private VectorStore vectorStore;
    private ChatClient chatClient;
    private RagService ragService;
    
    @BeforeEach
    void setUp() {
        // Mock VectorStore 설정
        vectorStore = mock(VectorStore.class);
        
        // 문서 검색 결과 설정
        List<Document> mockDocuments = List.of(
                new Document("1", "Spring AI는 다양한 AI 서비스를 통합하는 프레임워크입니다."),
                new Document("2", "Spring AI는 OpenAI, Anthropic 등 다양한 모델을 지원합니다.")
        );
        
        when(vectorStore.similaritySearch(any(String.class), anyInt())).thenReturn(mockDocuments);
        
        // Mock ChatClient 설정 - 검색 결과를 포함하는 지능적인 응답 생성
        Function<Prompt, String> responseFunction = prompt -> {
            // 프롬프트에서 시스템 메시지 확인
            String systemMessage = prompt.getMessages().stream()
                    .filter(m -> m instanceof SystemMessage)
                    .map(Message::getContent)
                    .findFirst()
                    .orElse("");
            
            // 시스템 메시지에 문서 내용이 포함되어 있는지 확인
            if (systemMessage.contains("Spring AI는 다양한 AI 서비스를 통합하는 프레임워크입니다")) {
                return "Spring AI는 Spring 생태계에서 AI 서비스를 쉽게 통합할 수 있게 해주는 프레임워크입니다. " +
                       "OpenAI, Anthropic 등 다양한 AI 제공업체를 지원합니다.";
            }
            
            return "질문에 답변할 수 있는 정보가 없습니다.";
        };
        
        chatClient = new MockChatClient(responseFunction);
        
        // RAG 서비스 생성
        ragService = new RagService(vectorStore, chatClient);
    }
    
    @Test
    void testRagQueryWithRelevantDocuments() {
        // RAG 쿼리 실행
        String response = ragService.query("Spring AI란 무엇인가요?");
        
        // 응답 확인
        assertThat(response).contains("Spring AI는");
        assertThat(response).contains("프레임워크");
        assertThat(response).contains("OpenAI");
        assertThat(response).contains("Anthropic");
    }
    
    @Test
    void testRagQueryWithNoRelevantDocuments() {
        // 빈 문서 목록 반환하도록 Mock 설정
        when(vectorStore.similaritySearch(any(String.class), anyInt())).thenReturn(List.of());
        
        // RAG 쿼리 실행
        String response = ragService.query("무관한 질문입니다.");
        
        // 응답 확인
        assertThat(response).isEqualTo("질문에 답변할 수 있는 정보가 없습니다.");
    }
    
    // 테스트 대상 RAG 서비스
    static class RagService {
        private final VectorStore vectorStore;
        private final ChatClient chatClient;
        
        public RagService(VectorStore vectorStore, ChatClient chatClient) {
            this.vectorStore = vectorStore;
            this.chatClient = chatClient;
        }
        
        public String query(String question) {
            // 관련 문서 검색
            List<Document> relevantDocs = vectorStore.similaritySearch(question, 3);
            
            if (relevantDocs.isEmpty()) {
                return "질문에 답변할 수 있는 정보가 없습니다.";
            }
            
            // 검색된 문서로 프롬프트 구성
            StringBuilder contextBuilder = new StringBuilder();
            contextBuilder.append("다음 정보를 바탕으로 질문에.답변하세요:\n\n");
            
            for (Document doc : relevantDocs) {
                contextBuilder.append("- ").append(doc.getContent()).append("\n");
            }
            
            // 프롬프트 생성
            List<Message> messages = List.of(
                    new SystemMessage(contextBuilder.toString()),
                    new UserMessage(question)
            );
            
            Prompt prompt = new Prompt(messages);
            
            // AI 호출 및 응답 반환
            return chatClient.call(prompt).getResult().getOutput().getContent();
        }
    }
}
```

## 결론

이 장에서는 Spring AI 애플리케이션을 효과적으로 테스트하기 위한 다양한 전략과 패턴을 살펴보았습니다. AI 애플리케이션 테스트의 특수한 과제와 Spring AI가 제공하는 테스트 도구를 이해하고, Mock 클라이언트와 녹화/재생 메커니즘을 활용하는 방법을 배웠습니다.

테스트 전략 요약:
1. **Mock 클라이언트 활용**: 외부 API 의존성 없이 빠르고 결정적인 테스트 실행
2. **녹화 및 재생**: 실제 AI 응답을 기록하고 재생하여 현실적인 테스트 수행
3. **통합 테스트**: 전체 애플리케이션 흐름에 대한 테스트 구현
4. **함수 호출 테스트**: AI 모델의 함수 호출 기능에 대한 테스트
5. **성능 및 로드 테스트**: 애플리케이션 성능과 확장성 테스트

이러한 테스트 전략을 활용하면 Spring AI 애플리케이션의 안정성, 신뢰성, 그리고 품질을 크게 향상시킬 수 있습니다. 다음 장에서는 AI 애플리케이션의 데이터 보안과 비용 관리에 대해 알아보겠습니다.