# 데이터 보안·비용 관리

## 개요

AI 애플리케이션을 개발하고 운영하는 데 있어 데이터 보안과 비용 관리는 핵심적인 고려사항입니다. 특히 Spring AI와 같은 프레임워크를 사용해 외부 AI 서비스를 연동할 때는 데이터 유출 위험과 예측 불가능한 비용 발생에 주의해야 합니다. 이 장에서는 Spring AI 애플리케이션의 데이터 보안을 강화하고 API 호출 비용을 효과적으로 관리하는 방법을 살펴보겠습니다.

## 데이터 보안 고려사항

### AI 애플리케이션의 보안 위험

AI 애플리케이션, 특히 외부 API에 의존하는 애플리케이션은 다음과 같은 보안 위험에 노출될 수 있습니다:

1. **데이터 전송 중 유출**: API 호출 과정에서 민감한 데이터가 유출될 위험
2. **프롬프트 주입 공격**: 사용자 입력을 통한 악의적인 프롬프트 주입
3. **개인정보 노출**: 프롬프트나 응답에 포함된 개인식별정보(PII) 유출
4. **API 키 노출**: 인증 자격 증명의 부적절한 관리로 인한 무단 접근
5. **트레이닝 데이터 유출**: 미세 조정이나 특정 용도 학습 과정에서의 데이터 유출
6. **악의적인 응답 생성**: AI 모델이 유해하거나 부적절한 콘텐츠를 생성할 위험

### 신뢰할 수 있는 AI 제공자 선택

Spring AI는 다양한 AI 서비스 제공자를 지원합니다. 제공자를 선택할 때 고려해야 할 보안 측면은 다음과 같습니다:

1. **데이터 처리 약관**: 제공자가 API를 통해 받은 데이터를 어떻게 처리하는지 확인
2. **규정 준수**: GDPR, HIPAA 등 관련 규정 준수 여부
3. **데이터 저장 위치**: 데이터가 저장되는 지리적 위치 및 해당 지역의 데이터 보호법
4. **데이터 보존 정책**: 요청 및 응답 데이터 보존 기간
5. **암호화 정책**: 전송 중 및 저장 시 암호화 방식

다음은 Spring AI에서 지원하는 주요 AI 제공자의 보안 정책 비교표입니다:

| 제공자 | 전송 중 암호화 | 저장 시 암호화 | 트레이닝 데이터 사용 | 데이터 보존 기간 | SOC 2 준수 |
|--------|----------------|----------------|----------------------|------------------|------------|
| OpenAI | TLS 1.2+ | AES-256 | 옵트아웃 가능 | 30일 | Yes |
| Anthropic | TLS 1.2+ | AES-256 | 옵트아웃 가능 | 30일 | Yes |
| Azure OpenAI | TLS 1.2+ | AES-256 | 사용 안 함 | 구성 가능 | Yes |
| Amazon Bedrock | TLS 1.2+ | AES-256 | 사용 안 함 | 구성 가능 | Yes |
| Google Vertex AI | TLS 1.2+ | AES-256 | 옵트아웃 가능 | 구성 가능 | Yes |

### Spring AI 애플리케이션의 보안 구성

Spring AI 애플리케이션에서 데이터 보안을 강화하기 위한 기본 구성 방법을 살펴보겠습니다.

#### API 키 보호

API 키는 코드나 버전 관리 시스템에 직접 포함해서는 안 됩니다. 대신 다음과 같은 방법을 사용해야 합니다:

```yaml
# application.yml
spring:
  ai:
    openai:
      # 직접 API 키를 포함하지 않음
      api-key: ${OPENAI_API_KEY}
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
```

프로덕션 환경에서는 다음 방법을 고려하세요:

1. **환경 변수 사용**
2. **외부 구성 서버** (Spring Cloud Config)
3. **시크릿 관리 도구** (AWS Secrets Manager, HashiCorp Vault)
4. **Kubernetes Secrets**

다음은 HashiCorp Vault를 사용한 API 키 관리 예시입니다:

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.vault.core.VaultTemplate;
import org.springframework.ai.openai.OpenAiChatClient;
import org.springframework.ai.openai.api.OpenAiApi;

@Configuration
public class AiSecurityConfig {

    private final VaultTemplate vaultTemplate;
    
    public AiSecurityConfig(VaultTemplate vaultTemplate) {
        this.vaultTemplate = vaultTemplate;
    }
    
    @Bean
    public OpenAiApi openAiApi() {
        // Vault에서 API 키 가져오기
        String apiKey = vaultTemplate
            .read("secret/ai-keys/openai")
            .getData()
            .get("api-key").toString();
        
        return new OpenAiApi(apiKey);
    }
    
    @Bean
    public OpenAiChatClient openAiChatClient(OpenAiApi openAiApi) {
        return new OpenAiChatClient(openAiApi);
    }
}
```

#### 전송 중 데이터 보호

Spring AI API 호출에서 전송 중인 데이터를 보호하려면 다음을 확인하세요:

1. **HTTPS 사용**: 모든 API 통신이 TLS/SSL을 통해 이루어지는지 확인
2. **최신 TLS 버전**: TLS 1.2 이상 사용
3. **인증서 검증**: 엄격한 인증서 검증 활성화

RestTemplate이나 WebClient를 직접 구성하는 경우 다음과 같이 설정할 수 있습니다:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.apache.hc.client5.http.impl.io.PoolingHttpClientConnectionManagerBuilder;
import org.apache.hc.client5.http.ssl.DefaultHostnameVerifier;
import org.apache.hc.client5.http.ssl.SSLConnectionSocketFactory;
import org.apache.hc.core5.ssl.SSLContexts;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;

import javax.net.ssl.SSLContext;

@Configuration
public class SecureHttpClientConfig {

    @Bean
    public RestTemplate secureRestTemplate() throws Exception {
        // TLS 1.2 이상 사용하는 SSLContext 생성
        SSLContext sslContext = SSLContexts.custom()
                .setProtocol("TLSv1.2")
                .build();
        
        // 호스트명 검증기 설정
        SSLConnectionSocketFactory sslSocketFactory = new SSLConnectionSocketFactory(
                sslContext,
                new String[]{"TLSv1.2", "TLSv1.3"},
                null,
                new DefaultHostnameVerifier());
        
        // HTTP 클라이언트 생성
        var connectionManager = PoolingHttpClientConnectionManagerBuilder.create()
                .setSSLSocketFactory(sslSocketFactory)
                .build();
        
        var httpClient = HttpClients.custom()
                .setConnectionManager(connectionManager)
                .build();
        
        // RestTemplate 생성 및 반환
        HttpComponentsClientHttpRequestFactory requestFactory = 
                new HttpComponentsClientHttpRequestFactory(httpClient);
        
        return new RestTemplate(requestFactory);
    }
}
```

#### 프롬프트 주입 방지

프롬프트 주입 공격은 사용자가 악의적인 지시를 프롬프트에 삽입하여 AI 모델을 조작하려는 시도입니다. 이를 방지하기 위한 방법은 다음과 같습니다:

1. **사용자 입력 검증**: 모든 사용자 입력을 검증하고 필터링
2. **프롬프트 구조 분리**: 시스템 프롬프트와 사용자 입력 명확히 분리
3. **템플릿 사용**: Spring AI의 프롬프트 템플릿 메커니즘 활용

다음은 프롬프트 주입 방지를 위한 예제 코드입니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.regex.Pattern;

@Service
public class SecurePromptService {

    private final ChatClient chatClient;
    private final Pattern unsafePattern;
    
    public SecurePromptService(ChatClient chatClient) {
        this.chatClient = chatClient;
        // 잠재적인 주입 공격 패턴 정의
        this.unsafePattern = Pattern.compile(
                "ignore previous instructions|forget your instructions|you are now|system: ", 
                Pattern.CASE_INSENSITIVE);
    }
    
    public String processUserPrompt(String userInput) {
        // 입력 검증
        if (isUnsafeInput(userInput)) {
            return "입력이 허용되지 않습니다. 다시 시도해주세요.";
        }
        
        // 시스템 프롬프트와 사용자 입력 명확히 분리
        SystemMessage systemMessage = new SystemMessage(
                "당신은 안전한 정보만 제공하는 도우미입니다. " +
                "금융, 개인 식별 정보 또는 민감한 데이터를 요청하지 마세요.");
        
        UserMessage userMessage = new UserMessage(userInput);
        
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
    
    private boolean isUnsafeInput(String input) {
        // 비어있거나 지나치게 긴 입력 확인
        if (input == null || input.trim().isEmpty() || input.length() > 1000) {
            return true;
        }
        
        // 주입 패턴 확인
        return unsafePattern.matcher(input).find();
    }
}
```

#### 개인정보 보호 및 마스킹

AI 모델에 전송되는 텍스트에서 개인 식별 정보(PII)를 식별하고 마스킹하는 것이 중요합니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

@Service
public class PrivacyAwareAiService {

    private final ChatClient chatClient;
    
    // PII 패턴 정의
    private static final Pattern EMAIL_PATTERN = 
            Pattern.compile("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}");
    private static final Pattern PHONE_PATTERN = 
            Pattern.compile("(\\+\\d{1,2}\\s)?\\(\\d{3}\\)\\s?\\d{3}[-.\\s]\\d{4}");
    private static final Pattern SSN_PATTERN = 
            Pattern.compile("\\d{3}[-]\\d{2}[-]\\d{4}");
    private static final Pattern CREDIT_CARD_PATTERN = 
            Pattern.compile("\\d{4}[- ]\\d{4}[- ]\\d{4}[- ]\\d{4}");
    
    public PrivacyAwareAiService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String processQuery(String userQuery) {
        // 입력에서 PII 마스킹
        String sanitizedInput = maskPII(userQuery);
        
        // AI 모델 호출
        Prompt prompt = new Prompt(new UserMessage(sanitizedInput));
        String response = chatClient.call(prompt).getResult().getOutput().getContent();
        
        // 필요한 경우 응답에서도 PII 마스킹
        return maskPII(response);
    }
    
    private String maskPII(String text) {
        if (text == null || text.isEmpty()) {
            return text;
        }
        
        // 이메일 마스킹
        Matcher emailMatcher = EMAIL_PATTERN.matcher(text);
        while (emailMatcher.find()) {
            String email = emailMatcher.group();
            String maskedEmail = maskEmail(email);
            text = text.replace(email, maskedEmail);
        }
        
        // 전화번호 마스킹
        Matcher phoneMatcher = PHONE_PATTERN.matcher(text);
        while (phoneMatcher.find()) {
            String phone = phoneMatcher.group();
            text = text.replace(phone, "XXX-XXX-XXXX");
        }
        
        // 주민등록번호/SSN 마스킹
        Matcher ssnMatcher = SSN_PATTERN.matcher(text);
        while (ssnMatcher.find()) {
            String ssn = ssnMatcher.group();
            text = text.replace(ssn, "XXX-XX-XXXX");
        }
        
        // 신용카드 번호 마스킹
        Matcher ccMatcher = CREDIT_CARD_PATTERN.matcher(text);
        while (ccMatcher.find()) {
            String cc = ccMatcher.group();
            text = text.replace(cc, "XXXX-XXXX-XXXX-XXXX");
        }
        
        return text;
    }
    
    private String maskEmail(String email) {
        int atIndex = email.indexOf('@');
        if (atIndex <= 1) {
            return "***@" + email.substring(atIndex + 1);
        }
        
        String username = email.substring(0, atIndex);
        String domain = email.substring(atIndex);
        
        int charsToShow = Math.min(3, username.length());
        return username.substring(0, charsToShow) + "***" + domain;
    }
}
```

### 데이터 보안을 위한 아키텍처 패턴

데이터 보안을 위한 몇 가지 효과적인 아키텍처 패턴을 살펴보겠습니다.

#### 민감한 데이터를 위한 프록시 패턴

민감한 데이터가 외부 AI 제공자에게 전송되는 것을 방지하기 위해 프록시 패턴을 사용할 수 있습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

@Service
public class DataProxyService {

    private final ChatClient chatClient;
    private final Map<String, String> sensitiveDataMap = new HashMap<>();
    
    public DataProxyService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String processWithDataProtection(String userInput) {
        // 민감한 데이터 식별 및 대체
        Pattern sensitivePattern = Pattern.compile("(PII:\\s*)([^\\s]+)");
        Matcher matcher = sensitivePattern.matcher(userInput);
        
        StringBuffer sanitizedBuffer = new StringBuffer();
        
        while (matcher.find()) {
            String sensitiveData = matcher.group(2);
            String token = generateToken();
            
            // 토큰과 실제 데이터 매핑 저장
            sensitiveDataMap.put(token, sensitiveData);
            
            matcher.appendReplacement(sanitizedBuffer, "PII: " + token);
        }
        matcher.appendTail(sanitizedBuffer);
        
        String sanitizedInput = sanitizedBuffer.toString();
        
        // 대체된 입력으로 AI 호출
        Prompt prompt = new Prompt(new UserMessage(sanitizedInput));
        String aiResponse = chatClient.call(prompt).getResult().getOutput().getContent();
        
        // 응답에서 토큰을 원래 데이터로 복원
        for (Map.Entry<String, String> entry : sensitiveDataMap.entrySet()) {
            aiResponse = aiResponse.replace(entry.getKey(), entry.getValue());
        }
        
        // 사용된 매핑 제거
        sensitiveDataMap.clear();
        
        return aiResponse;
    }
    
    private String generateToken() {
        return "TOKEN_" + UUID.randomUUID().toString().substring(0, 8);
    }
}
```

#### 로컬 LLM 대체 패턴

특히 민감한 정보에 대해서는 로컬에서 실행되는 LLM으로 처리하는 패턴을 사용할 수 있습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.classification.TextClassifier;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class HybridAiService {

    private final ChatClient remoteChatClient; // OpenAI, Anthropic 등
    private final ChatClient localChatClient;  // Ollama와 같은 로컬 LLM
    private final TextClassifier sensitivityClassifier;
    
    public HybridAiService(
            @Qualifier("remoteChatClient") ChatClient remoteChatClient,
            @Qualifier("localChatClient") ChatClient localChatClient,
            TextClassifier sensitivityClassifier) {
        this.remoteChatClient = remoteChatClient;
        this.localChatClient = localChatClient;
        this.sensitivityClassifier = sensitivityClassifier;
    }
    
    public String processQuery(String userQuery) {
        // 쿼리의 민감도 분석
        float sensitivityScore = sensitivityClassifier.classify(userQuery);
        
        // 민감도에 따라 적절한 ChatClient 선택
        ChatClient selectedClient = 
                (sensitivityScore > 0.7) ? localChatClient : remoteChatClient;
        
        // 선택된 클라이언트로 쿼리 처리
        Prompt prompt = new Prompt(new UserMessage(userQuery));
        return selectedClient.call(prompt).getResult().getOutput().getContent();
    }
}
```

### 윤리적 고려사항

AI 애플리케이션 개발에서 보안뿐만 아니라 윤리적 측면도 중요합니다. 다음과 같은 사항을 고려해야 합니다:

1. **투명성**: AI 모델이 사용되고 있음을 사용자에게 공개적으로 알리기
2. **공정성**: AI 응답에서 편향을 모니터링하고 완화하기
3. **책임성**: AI 시스템이 생성한 콘텐츠에 대한 명확한 책임 할당
4. **옵트아웃 옵션**: 사용자에게 AI 처리를 거부할 수 있는 옵션 제공

다음은 윤리적 가이드라인을 구현하는 예시입니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class EthicalAiService {

    private final ChatClient chatClient;
    
    public EthicalAiService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String processEthicalQuery(String userQuery, boolean aiDisclosureRequired) {
        // 윤리적 가이드라인이 포함된 시스템 프롬프트
        SystemMessage systemMessage = new SystemMessage(
                "당신은 공정하고 편향되지 않은 AI 도우미입니다. 답변할 때 다음 원칙을 따르세요:\n" +
                "1. 모든 사람을 존중하고 공정하게 대우하세요.\n" +
                "2. 불확실할 때는 그 사실을 인정하세요.\n" +
                "3. 유해하거나 불법적인 콘텐츠 요청은 거부하세요.\n" +
                "4. 문화적, 사회적 배경의 다양성을 고려하세요.\n" +
                "5. 사실과 의견을 명확히 구분하세요."
        );
        
        UserMessage userMessage = new UserMessage(userQuery);
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        
        String response = chatClient.call(prompt).getResult().getOutput().getContent();
        
        // AI 사용 공개
        if (aiDisclosureRequired) {
            response += "\n\n[이 응답은 AI 시스템에 의해 생성되었습니다.]";
        }
        
        return response;
    }
    
    public boolean needsModeration(String content) {
        // 콘텐츠 모더레이션 로직
        // 실제 구현에서는 별도의 모더레이션 API 또는 모델 사용
        return content.toLowerCase().contains("harmful") || 
               content.toLowerCase().contains("illegal");
    }
}
```

## 비용 관리 전략

AI API 호출은 예측 불가능하고 상당한 비용이 발생할 수 있습니다. 다음은 Spring AI 애플리케이션에서 비용을 관리하는 효과적인 전략입니다.

### AI 비용 모니터링 및 추적

비용을 관리하려면 먼저 측정해야 합니다. Spring AI 애플리케이션에서 API 사용을 모니터링하는 방법을 살펴보겠습니다.

#### 토큰 사용량 추적

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.tokenizer.TokenCountingService;
import org.springframework.stereotype.Service;

@Service
public class TokenTrackingService {

    private final ChatClient chatClient;
    private final TokenCountingService tokenizer;
    private final TokenUsageRepository usageRepository;
    
    public TokenTrackingService(
            ChatClient chatClient, 
            TokenCountingService tokenizer,
            TokenUsageRepository usageRepository) {
        this.chatClient = chatClient;
        this.tokenizer = tokenizer;
        this.usageRepository = usageRepository;
    }
    
    public String processPromptWithTracking(Prompt prompt, String userId) {
        // 프롬프트 토큰 수 계산
        int promptTokens = tokenizer.count(prompt);
        
        // AI 호출
        var response = chatClient.call(prompt);
        String content = response.getResult().getOutput().getContent();
        
        // 응답 토큰 수 계산
        int completionTokens = tokenizer.count(content);
        
        // 사용량 저장
        usageRepository.recordUsage(
                userId, 
                promptTokens, 
                completionTokens, 
                calculateCost(promptTokens, completionTokens));
        
        return content;
    }
    
    private double calculateCost(int promptTokens, int completionTokens) {
        // OpenAI GPT-4의 경우 (2023년 기준 예시 가격)
        double promptCostPer1000 = 0.03;
        double completionCostPer1000 = 0.06;
        
        return (promptTokens / 1000.0 * promptCostPer1000) + 
               (completionTokens / 1000.0 * completionCostPer1000);
    }
}

// 토큰 사용량 저장소 인터페이스
interface TokenUsageRepository {
    void recordUsage(String userId, int promptTokens, int completionTokens, double cost);
}
```

#### 비용 모니터링 대시보드

Spring Boot Actuator와 Micrometer를 사용하여 비용 메트릭을 수집하고 모니터링할 수 있습니다:

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

@Service
public class MetricsTrackingChatService {

    private final ChatClient chatClient;
    private final Counter tokenCounter;
    private final Counter costCounter;
    
    public MetricsTrackingChatService(ChatClient chatClient, MeterRegistry registry) {
        this.chatClient = chatClient;
        this.tokenCounter = Counter.builder("ai.tokens.total")
                .description("Total number of tokens used")
                .register(registry);
        this.costCounter = Counter.builder("ai.cost.total")
                .description("Total cost of AI API calls in USD")
                .register(registry);
    }
    
    public String processPrompt(Prompt prompt) {
        var response = chatClient.call(prompt);
        
        // 토큰 사용량 (OpenAI만 가능, 다른 제공자는 응답에서 이 정보를 제공하지 않을 수 있음)
        var usage = response.getMetadata().get("usage");
        if (usage instanceof Map) {
            Map<String, Object> usageMap = (Map<String, Object>) usage;
            int totalTokens = (int) usageMap.getOrDefault("total_tokens", 0);
            
            // 메트릭 업데이트
            tokenCounter.increment(totalTokens);
            
            // 비용 계산 및 업데이트 (예시)
            double cost = totalTokens / 1000.0 * 0.05; // $0.05 per 1K tokens
            costCounter.increment(cost);
        }
        
        return response.getResult().getOutput().getContent();
    }
}
```

### 비용 최적화 기법

다음은 AI API 호출 비용을 최적화하는 여러 기법입니다.

#### 캐싱 구현

동일한 쿼리에 대해 반복적인 API 호출을 피하기 위해 응답 캐싱을 구현합니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class CachingAiService {

    private final ChatClient chatClient;
    
    public CachingAiService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    @Cacheable(value = "aiResponses", key = "#prompt.toString()")
    public String getAiResponse(Prompt prompt) {
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
}
```

더 세밀한 제어가 필요한 경우 Spring의 캐시 추상화를 확장할 수 있습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.cache.CacheManager;
import org.springframework.stereotype.Service;

import java.util.Objects;

@Service
public class AdvancedCachingAiService {

    private final ChatClient chatClient;
    private final CacheManager cacheManager;
    
    public AdvancedCachingAiService(ChatClient chatClient, CacheManager cacheManager) {
        this.chatClient = chatClient;
        this.cacheManager = cacheManager;
    }
    
    public String getAiResponse(Prompt prompt, boolean allowCached) {
        String promptKey = generatePromptKey(prompt);
        
        // 캐시 조회가 허용된 경우 캐시에서 확인
        if (allowCached) {
            var cache = cacheManager.getCache("aiResponses");
            if (cache != null) {
                String cachedResponse = cache.get(promptKey, String.class);
                if (cachedResponse != null) {
                    return cachedResponse;
                }
            }
        }
        
        // 캐시에 없거나 캐시 사용이 허용되지 않은 경우 API 호출
        String response = chatClient.call(prompt).getResult().getOutput().getContent();
        
        // 응답 캐싱
        var cache = cacheManager.getCache("aiResponses");
        if (cache != null) {
            cache.put(promptKey, response);
        }
        
        return response;
    }
    
    private String generatePromptKey(Prompt prompt) {
        // 프롬프트 정규화 및 해싱
        return Objects.hash(prompt.toString()) + "";
    }
}
```

#### 토큰 사용량 최적화

토큰 사용량을 최적화하여 비용을 절감하는 방법을 살펴보겠습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Service
public class TokenOptimizationService {

    private final ChatClient chatClient;
    private final int maxChatHistory;
    
    public TokenOptimizationService(ChatClient chatClient, int maxChatHistory) {
        this.chatClient = chatClient;
        this.maxChatHistory = maxChatHistory;
    }
    
    public String chat(String userInput, List<ChatMessage> chatHistory) {
        // 시스템 프롬프트 최적화: 간결하고 명확하게
        SystemMessage systemMessage = new SystemMessage(
                "간결하고 정확하게 답변하세요. 불필요한 인사말이나 장식적 표현을 사용하지 마세요.");
        
        // 채팅 기록 최적화: 토큰 제한을 위해 최근 N개 메시지만 유지
        List<ChatMessage> optimizedHistory = optimizeChatHistory(chatHistory);
        
        // 메시지 목록 구성
        List<Message> messages = new ArrayList<>();
        messages.add(systemMessage);
        
        // 최적화된 기록 변환
        for (ChatMessage historicalMessage : optimizedHistory) {
            if ("user".equals(historicalMessage.getRole())) {
                messages.add(new UserMessage(historicalMessage.getContent()));
            } else if ("assistant".equals(historicalMessage.getRole())) {
                messages.add(new org.springframework.ai.chat.messages.AssistantMessage(
                        historicalMessage.getContent()));
            }
        }
        
        // 현재 사용자 입력 추가
        messages.add(new UserMessage(userInput));
        
        // AI 호출
        Prompt prompt = new Prompt(messages);
        String response = chatClient.call(prompt).getResult().getOutput().getContent();
        
        return response;
    }
    
    private List<ChatMessage> optimizeChatHistory(List<ChatMessage> history) {
        if (history.size() <= maxChatHistory) {
            return new ArrayList<>(history);
        }
        
        // 최근 N개 메시지만 유지
        return new ArrayList<>(history.subList(history.size() - maxChatHistory, history.size()));
    }
    
    // 채팅 메시지 클래스
    public static class ChatMessage {
        private String role;
        private String content;
        
        // getter, setter
        public String getRole() {
            return role;
        }
        
        public String getContent() {
            return content;
        }
    }
}
```

#### 모델 계층화

비용 대비 성능을 최적화하기 위해 다양한 모델을 계층적으로 활용할 수 있습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class TieredModelService {

    private final ChatClient basicModelClient;    // 저비용 모델 (GPT-3.5 등)
    private final ChatClient advancedModelClient; // 고비용 모델 (GPT-4 등)
    private final ComplexityAnalyzer complexityAnalyzer;
    
    public TieredModelService(
            @Qualifier("basicModelClient") ChatClient basicModelClient,
            @Qualifier("advancedModelClient") ChatClient advancedModelClient,
            ComplexityAnalyzer complexityAnalyzer) {
        this.basicModelClient = basicModelClient;
        this.advancedModelClient = advancedModelClient;
        this.complexityAnalyzer = complexityAnalyzer;
    }
    
    public String processQuery(String userQuery) {
        // 쿼리 복잡성 분석
        QueryComplexity complexity = complexityAnalyzer.analyzeComplexity(userQuery);
        
        // 복잡성에 따라 적절한 모델 선택
        ChatClient selectedClient = (complexity == QueryComplexity.HIGH) 
                ? advancedModelClient 
                : basicModelClient;
        
        // 선택된 모델로 쿼리 처리
        Prompt prompt = new Prompt(new UserMessage(userQuery));
        return selectedClient.call(prompt).getResult().getOutput().getContent();
    }
    
    // 복잡성 분석기 인터페이스
    interface ComplexityAnalyzer {
        QueryComplexity analyzeComplexity(String query);
    }
    
    // 쿼리 복잡성 열거형
    enum QueryComplexity {
        LOW, MEDIUM, HIGH
    }
}
```

### 비용 제어 메커니즘

API 호출 비용이 예산을 초과하지 않도록 하는 제어 메커니즘을 구현하는 방법을 살펴보겠습니다.

#### 할당량 및 제한 설정

사용자 또는 애플리케이션별 할당량을 설정하여 API 사용량을 제한합니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

@Service
public class QuotaManagementService {

    private final ChatClient chatClient;
    private final QuotaRepository quotaRepository;
    
    public QuotaManagementService(ChatClient chatClient, QuotaRepository quotaRepository) {
        this.chatClient = chatClient;
        this.quotaRepository = quotaRepository;
    }
    
    public String processPromptWithQuota(String userId, Prompt prompt) {
        // 사용자 할당량 확인
        UserQuota quota = quotaRepository.getUserQuota(userId);
        
        if (quota.isExceeded()) {
            return "월간 AI 사용 할당량을 초과했습니다. 다음 달까지 기다리거나 요금제를 업그레이드하세요.";
        }
        
        if (!quota.hasAvailableTokens(estimateTokenCount(prompt))) {
            return "현재 요청을 처리하기에 충분한 토큰이 남아있지 않습니다.";
        }
        
        // AI 호출
        String response = chatClient.call(prompt).getResult().getOutput().getContent();
        
        // 사용량 업데이트
        quota.consumeTokens(estimateTokenCount(prompt) + estimateTokenCount(response));
        quotaRepository.updateQuota(userId, quota);
        
        return response;
    }
    
    private int estimateTokenCount(Object text) {
        // 토큰 수 추정 (간단한 구현)
        String content = text.toString();
        // 대략 1 토큰 = 4 글자 (영어 기준)
        return content.length() / 4;
    }
    
    // 인터페이스 및 클래스
    interface QuotaRepository {
        UserQuota getUserQuota(String userId);
        void updateQuota(String userId, UserQuota quota);
    }
    
    static class UserQuota {
        private int monthlyTokenLimit;
        private int tokensUsed;
        
        public boolean isExceeded() {
            return tokensUsed >= monthlyTokenLimit;
        }
        
        public boolean hasAvailableTokens(int requiredTokens) {
            return (tokensUsed + requiredTokens) <= monthlyTokenLimit;
        }
        
        public void consumeTokens(int tokenCount) {
            this.tokensUsed += tokenCount;
        }
    }
}
```

#### 경보 및 자동 셧다운

비용이 특정 임계값에 도달하면 경보를 발생시키고 필요한 경우 AI 기능을 자동으로 비활성화합니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;

@Service
public class CostAlertingService {

    private final ChatClient chatClient;
    private final CostAlertConfig alertConfig;
    private final NotificationService notificationService;
    
    // 비용 추적 변수
    private final AtomicInteger currentMonthCost = new AtomicInteger(0);
    private final AtomicBoolean aiServicesEnabled = new AtomicBoolean(true);
    private LocalDate lastResetDate = LocalDate.now().withDayOfMonth(1);
    
    public CostAlertingService(
            ChatClient chatClient, 
            CostAlertConfig alertConfig,
            NotificationService notificationService) {
        this.chatClient = chatClient;
        this.alertConfig = alertConfig;
        this.notificationService = notificationService;
    }
    
    public String processPromptWithCostControl(Prompt prompt) {
        // 새 달이 시작되면 카운터 재설정
        resetCounterIfNewMonth();
        
        // AI 서비스가 비활성화되었는지 확인
        if (!aiServicesEnabled.get()) {
            return "비용 제한에 도달하여 AI 서비스가 비활성화되었습니다. 관리자에게 문의하세요.";
        }
        
        // AI 호출
        String response = chatClient.call(prompt).getResult().getOutput().getContent();
        
        // 비용 추정 및 업데이트
        int tokenCount = estimateTokenCount(prompt) + estimateTokenCount(response);
        double cost = calculateCost(tokenCount);
        int newTotalCost = currentMonthCost.addAndGet((int) (cost * 100)); // 센트 단위로 저장
        
        // 경보 임계값 확인
        checkAlertThresholds(newTotalCost / 100.0);
        
        return response;
    }
    
    private void checkAlertThresholds(double currentCost) {
        // 경고 임계값
        if (currentCost >= alertConfig.getWarningThreshold() && 
                currentCost < alertConfig.getWarningThreshold() + 10) {
            notificationService.sendAlert(
                    "AI 비용 경고",
                    String.format("월간 AI 비용이 %.2f$에 도달했습니다", currentCost));
        }
        
        // 중요 임계값
        if (currentCost >= alertConfig.getCriticalThreshold() && 
                currentCost < alertConfig.getCriticalThreshold() + 10) {
            notificationService.sendAlert(
                    "AI 비용 중요 경고", 
                    String.format("월간 AI 비용이 %.2f$에 도달했습니다", currentCost));
        }
        
        // 셧다운 임계값
        if (currentCost >= alertConfig.getShutdownThreshold() && aiServicesEnabled.get()) {
            aiServicesEnabled.set(false);
            notificationService.sendAlert(
                    "AI 서비스 비활성화됨",
                    String.format("비용 임계값(%.2f$)에 도달하여 AI 서비스가 비활성화되었습니다", 
                            alertConfig.getShutdownThreshold()));
        }
    }
    
    private void resetCounterIfNewMonth() {
        LocalDate today = LocalDate.now();
        LocalDate firstDayOfMonth = today.withDayOfMonth(1);
        
        if (ChronoUnit.DAYS.between(lastResetDate, firstDayOfMonth) > 0) {
            currentMonthCost.set(0);
            aiServicesEnabled.set(true);
            lastResetDate = firstDayOfMonth;
        }
    }
    
    private int estimateTokenCount(Object text) {
        // 토큰 수 추정 구현
        String content = text.toString();
        return content.length() / 4;
    }
    
    private double calculateCost(int tokenCount) {
        // 비용 계산 구현 (예: $0.03 per 1K tokens)
        return tokenCount / 1000.0 * 0.03;
    }
    
    // 설정 및 인터페이스
    static class CostAlertConfig {
        private double warningThreshold = 100.0;  // 월 $100
        private double criticalThreshold = 300.0; // 월 $300
        private double shutdownThreshold = 500.0; // 월 $500
        
        // getter
        public double getWarningThreshold() {
            return warningThreshold;
        }
        
        public double getCriticalThreshold() {
            return criticalThreshold;
        }
        
        public double getShutdownThreshold() {
            return shutdownThreshold;
        }
    }
    
    interface NotificationService {
        void sendAlert(String title, String message);
    }
}
```

### 비용 대비 품질 최적화

비용을 절감하면서도 품질을 유지하는 몇 가지 최적화 기법을 살펴보겠습니다.

#### 프롬프트 엔지니어링을 통한 효율성 향상

효과적인 프롬프트를 사용하여 더 적은 비용으로 더 나은 결과를 얻을 수 있습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class EfficientPromptService {

    private final ChatClient chatClient;
    
    public EfficientPromptService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String generateSummary(String longText) {
        // 비효율적인 방법: 직접 전체 텍스트 전송
        // return generateSimpleSummary(longText);
        
        // 효율적인 방법: 청크로 분할하고 단계적 요약
        return generateHierarchicalSummary(longText);
    }
    
    private String generateSimpleSummary(String text) {
        // 비효율적 접근 방식: 전체 텍스트를 한 번에 처리
        UserMessage userMessage = new UserMessage(
                "다음 텍스트를 간결하게 요약해주세요:\n\n" + text);
        
        Prompt prompt = new Prompt(userMessage);
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
    
    private String generateHierarchicalSummary(String text) {
        // 효율적 접근 방식: 텍스트를 청크로 분할하고 계층적 요약
        
        // 1. 텍스트를 여러 청크로 분할
        List<String> chunks = splitIntoChunks(text, 4000);
        
        // 2. 각 청크 요약
        StringBuilder intermediateResults = new StringBuilder();
        for (int i = 0; i < chunks.size(); i++) {
            UserMessage chunkMessage = new UserMessage(
                    "다음은 더 큰 문서의 섹션 " + (i+1) + "/" + chunks.size() + "입니다. " +
                    "3-5문장으로 핵심 요점만 요약하세요:\n\n" + chunks.get(i));
            
            Prompt chunkPrompt = new Prompt(chunkMessage);
            String chunkSummary = chatClient.call(chunkPrompt).getResult().getOutput().getContent();
            
            intermediateResults.append("섹션 ").append(i+1).append(" 요약: ")
                    .append(chunkSummary).append("\n\n");
        }
        
        // 3. 최종 요약 생성
        SystemMessage systemMessage = new SystemMessage(
                "당신은 문서 요약 전문가입니다. 각 섹션의 요약을 기반으로 전체 문서의 포괄적인 요약을 작성하세요.");
        
        UserMessage finalMessage = new UserMessage(
                "다음은 긴 문서의 섹션별 요약입니다. 이를 바탕으로 전체 문서의 일관된 요약을 작성해주세요:\n\n" + 
                intermediateResults.toString());
        
        Prompt finalPrompt = new Prompt(List.of(systemMessage, finalMessage));
        return chatClient.call(finalPrompt).getResult().getOutput().getContent();
    }
    
    private List<String> splitIntoChunks(String text, int chunkSize) {
        // 텍스트를 청크로 분할하는 로직
        // (구현 생략)
        return List.of(); // 실제 구현 필요
    }
}
```

#### 하이브리드 접근 방식: AI와 규칙 기반 시스템 결합

모든 작업에 AI를 사용하는 대신, 일부 작업은 규칙 기반 시스템으로 처리하여 비용을 절감할 수 있습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.regex.Pattern;

@Service
public class HybridProcessingService {

    private final ChatClient chatClient;
    private final Map<Pattern, String> faqResponses = new HashMap<>();
    
    public HybridProcessingService(ChatClient chatClient) {
        this.chatClient = chatClient;
        
        // FAQ 패턴과 응답 초기화
        faqResponses.put(
                Pattern.compile("(?i).*영업\\s*시간.*|.*opening\\s*hours.*"), 
                "저희 영업시간은 평일 오전 9시부터 오후 6시까지입니다.");
        faqResponses.put(
                Pattern.compile("(?i).*위치.*어디.*|.*address.*|.*location.*"), 
                "저희 주소는 서울시 강남구 테헤란로 123입니다.");
        faqResponses.put(
                Pattern.compile("(?i).*환불.*정책.*|.*refund.*policy.*"), 
                "구매 후 7일 이내에 미사용 제품에 한해 전액 환불이 가능합니다.");
        // 추가 FAQ 패턴
    }
    
    public String processQuery(String userQuery) {
        // 1. 먼저 FAQ 패턴 확인
        for (Map.Entry<Pattern, String> entry : faqResponses.entrySet()) {
            if (entry.getKey().matcher(userQuery).matches()) {
                return entry.getValue();
            }
        }
        
        // 2. 간단한 인사말 처리
        if (isGreeting(userQuery)) {
            return "안녕하세요! 무엇을 도와드릴까요?";
        }
        
        // 3. 기타 간단한 규칙 기반 처리
        if (userQuery.length() < 5) {
            return "더 자세한 질문을 해주시면 더 잘 도와드릴 수 있습니다.";
        }
        
        // 4. 위의 모든 규칙에 해당하지 않는 경우 AI 모델 사용
        Prompt prompt = new Prompt(new UserMessage(userQuery));
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
    
    private boolean isGreeting(String text) {
        String lowerText = text.toLowerCase();
        return lowerText.matches(".*안녕.*|.*hi.*|.*hello.*|.*hey.*");
    }
}
```

## 실제 사용 사례: 의료 데이터 기반 AI 어시스턴트

실제 사용 사례를 통해 보안 및 비용 관리 기법을 통합적으로 살펴보겠습니다.

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.regex.Pattern;

@Service
public class MedicalAiAssistantService {

    private final ChatClient primaryClient;
    private final ChatClient fallbackClient;
    private final AiUsageMonitor usageMonitor;
    private final PrivacyFilter privacyFilter;
    
    // 비용 관리를 위한 카운터
    private final AtomicInteger dailyApiCalls = new AtomicInteger(0);
    private LocalDateTime lastResetTime = LocalDateTime.now();
    
    // 민감한 의료 정보 탐지를 위한 패턴
    private final Pattern piiPattern = Pattern.compile(
            "\\b(\\d{3}-\\d{2}-\\d{4}|(?i)ssn|주민\\s*등록\\s*번호|환자\\s*id|patient\\s*id)\\b");
    
    public MedicalAiAssistantService(
            @Qualifier("primaryChatClient") ChatClient primaryClient,
            @Qualifier("fallbackChatClient") ChatClient fallbackClient,
            AiUsageMonitor usageMonitor,
            PrivacyFilter privacyFilter) {
        this.primaryClient = primaryClient;
        this.fallbackClient = fallbackClient;
        this.usageMonitor = usageMonitor;
        this.privacyFilter = privacyFilter;
    }
    
    @PreAuthorize("hasRole('DOCTOR') or hasRole('NURSE')")
    @Cacheable(value = "medicalQueries", key = "#query", condition = "#allowCaching == true")
    public String getMedicalInformation(String query, boolean allowCaching) {
        // 일일 API 호출 제한 확인 및 초기화
        checkAndResetDailyCounter();
        if (dailyApiCalls.get() >= 1000) { // 일일 1000회 제한
            return "일일 AI 쿼리 한도에 도달했습니다. 내일 다시 시도하거나 관리자에게 문의하세요.";
        }
        
        // 개인 식별 정보 확인
        if (piiPattern.matcher(query).find()) {
            return "요청에 민감한 환자 정보가 포함되어 있습니다. 개인식별정보 없이 다시 문의해주세요.";
        }
        
        // 민감한 정보 마스킹
        String sanitizedQuery = privacyFilter.maskSensitiveInformation(query);
        
        // 시스템 프롬프트 설정 (의료 전문 지식과 제한사항 포함)
        SystemMessage systemMessage = new SystemMessage(
                "당신은 의료 전문가를 돕는 AI 어시스턴트입니다. " +
                "의학적 사실에 근거하여 정확한 정보를 제공하세요. " +
                "진단을 내리거나 처방을 제안하지 말고, 의학 문헌에 기반한 정보만 제공하세요. " +
                "불확실한 사항에 대해서는 명확히 그렇다고 인정하세요. " +
                "모든 답변에 관련 의학 문헌이나 가이드라인을 인용하세요.");
        
        UserMessage userMessage = new UserMessage(sanitizedQuery);
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        
        try {
            // API 호출 카운터 증가
            dailyApiCalls.incrementAndGet();
            
            // 주 클라이언트로 API 호출
            String response = primaryClient.call(prompt).getResult().getOutput().getContent();
            
            // 사용량 기록
            usageMonitor.recordApiCall("medical", sanitizedQuery, response);
            
            return response;
        } catch (Exception e) {
            // 주 클라이언트 실패 시 폴백 클라이언트 사용
            try {
                return fallbackClient.call(prompt).getResult().getOutput().getContent() + 
                        "\n\n(참고: 이 응답은 백업 AI 시스템에서 생성되었습니다)";
            } catch (Exception fallbackEx) {
                return "죄송합니다. 현재 AI 서비스를 이용할 수 없습니다. 나중에 다시 시도해주세요.";
            }
        }
    }
    
    private void checkAndResetDailyCounter() {
        LocalDateTime now = LocalDateTime.now();
        if (now.getDayOfYear() != lastResetTime.getDayOfYear() || 
                now.getYear() != lastResetTime.getYear()) {
            dailyApiCalls.set(0);
            lastResetTime = now;
        }
    }
    
    // 인터페이스
    interface AiUsageMonitor {
        void recordApiCall(String category, String query, String response);
    }
    
    interface PrivacyFilter {
        String maskSensitiveInformation(String text);
    }
}
```

## 결론

이 장에서는 Spring AI 애플리케이션의 데이터 보안과 비용 관리 전략에 대해 살펴보았습니다. 데이터 보안 측면에서는 API 키 보호, 전송 중 데이터 보호, 프롬프트 주입 방지, 개인정보 마스킹, 다양한 보안 아키텍처 패턴을 알아보았습니다. 비용 관리 측면에서는 토큰 사용량 추적, 캐싱, 토큰 최적화, 모델 계층화, 할당량 설정, 경보 및 자동 셧다운과 같은 기법을 살펴보았습니다.

이러한 전략을 적용하면 Spring AI 애플리케이션의 보안을 강화하고 예측 가능한 비용으로 운영할 수 있습니다. 보안과 비용 관리는 프로젝트 초기 단계부터 고려해야 하는 중요한 요소이며, Spring AI는 이를 효과적으로 구현할 수 있는 다양한 메커니즘을 제공합니다.

다음 장에서는 Spring AI 애플리케이션의 관측성과 로깅에 대해 살펴볼 것입니다. 이는 애플리케이션의 동작을 모니터링하고 문제를 신속하게 식별하여 대응하기 위한 핵심 기능입니다.