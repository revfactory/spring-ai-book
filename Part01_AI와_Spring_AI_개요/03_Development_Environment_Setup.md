# 개발 환경 설정

## 개요

Spring AI 프로젝트를 시작하기 위해서는 적절한 개발 환경 설정이 필요합니다. 이 장에서는 Java 17 이상(권장: Java 21), Spring Boot 3.4.x, Spring AI 1.0.0을 기반으로 하는 개발 환경을 구축하는 방법을 단계별로 설명합니다. 또한 다양한 AI 모델 프로바이더와의 연동을 위한 API 키 설정 방법도 함께 알아봅니다.

## 필수 소프트웨어 요구사항

### Java 17 이상 설치

Spring AI는 Java 17 이상을 요구합니다. 최신 기능과 성능 향상을 위해 Java 21(LTS)을 권장합니다. Java 21은 가상 스레드(Virtual Threads)와 같은 새로운 기능을 제공합니다.

#### Windows에서 설치

1. [Oracle JDK 다운로드 페이지](https://www.oracle.com/java/technologies/downloads/#java21)에서 Windows용 인스톨러를 다운로드합니다.
2. 다운로드한 인스톨러를 실행하고 안내에 따라 설치를 완료합니다.
3. 설치 후 명령 프롬프트에서 `java -version` 명령어로 설치를 확인합니다.

#### macOS에서 설치

1. Homebrew를 사용하는 경우: `brew install openjdk@21` 명령으로 설치합니다.
2. [Oracle JDK 다운로드 페이지](https://www.oracle.com/java/technologies/downloads/#java21)에서 macOS용 패키지를 다운로드하여 설치할 수도 있습니다.
3. 설치 후 터미널에서 `java -version` 명령어로 설치를 확인합니다.

#### Linux에서 설치

1. Ubuntu/Debian: `sudo apt-get install openjdk-21-jdk` 명령으로 설치합니다.
2. RHEL/Fedora: `sudo dnf install java-21-openjdk-devel` 명령으로 설치합니다.
3. 설치 후 `java -version` 명령어로 설치를 확인합니다.

### Maven 또는 Gradle 설정

Spring AI 프로젝트는 Maven이나 Gradle을 사용하여 의존성을 관리합니다. 두 도구 모두 Spring AI와 잘 작동하지만, 이 책에서는 주로 Gradle을 사용합니다.

#### Gradle 설치

1. [Gradle 다운로드 페이지](https://gradle.org/install/)에서 최신 버전을 다운로드하거나, 패키지 관리자를 사용하여 설치합니다.
   - Windows: Scoop 또는 Chocolatey를 사용할 수 있습니다.
   - macOS: `brew install gradle` 명령으로 설치합니다.
   - Linux: 배포판별 패키지 관리자를 사용합니다.
2. 설치 후 `gradle -v` 명령어로 설치를 확인합니다.

#### Maven 설치

1. [Maven 다운로드 페이지](https://maven.apache.org/download.cgi)에서 최신 버전을 다운로드하고 설치합니다.
2. 설치 후 `mvn -v` 명령어로 설치를 확인합니다.

### IDE 설정

Spring AI 개발을 위해 IDE(통합 개발 환경)를 설정합니다. 권장하는 IDE는 다음과 같습니다:

#### IntelliJ IDEA

1. [IntelliJ IDEA](https://www.jetbrains.com/idea/download/)를 다운로드하고 설치합니다. Community Edition(무료)이나 Ultimate Edition 모두 사용 가능합니다.
2. Spring 개발을 위한 플러그인이 사전 설치되어 있는지 확인합니다.
   - Spring Initializr 플러그인
   - Spring Boot 플러그인

#### Visual Studio Code

1. [Visual Studio Code](https://code.visualstudio.com/download)를 다운로드하고 설치합니다.
2. Java 개발을 위한 확장 프로그램을 설치합니다.
   - Extension Pack for Java
   - Spring Boot Extension Pack
   - Gradle for Java

#### Eclipse

1. [Spring Tools 4 for Eclipse](https://spring.io/tools)를 다운로드하고 설치합니다.
2. 이 버전은 Spring 개발에 필요한 도구가 사전 설치되어 있습니다.

## Spring Boot 프로젝트 생성

### Spring Initializr 사용

1. [Spring Initializr 웹사이트](https://start.spring.io/)를 방문합니다.
2. 다음과 같이 프로젝트 설정을 구성합니다:
   - Project: Gradle Project 또는 Maven Project
   - Language: Java
   - Spring Boot: 3.4.x (Spring AI는 3.4.x를 지원하며, 3.5.x 릴리스 시 지원 예정)
   - Project Metadata: 자신의 프로젝트에 맞게 설정
   - Packaging: Jar
   - Java: 17 이상 (21 권장)
3. AI Models와 Vector Stores 선택:
   - 원하는 AI 모델 (OpenAI, Anthropic, Ollama 등)
   - 필요한 경우 Vector Store (PostgreSQL/PGVector, Chroma 등)
4. 'Generate' 버튼을 클릭하여 프로젝트를 다운로드합니다.

### 명령줄에서 생성

다음 명령을 사용하여 Spring Boot 프로젝트를 생성할 수도 있습니다:

```bash
spring init --build=gradle --java-version=21 --boot-version=3.4.0 \
  --dependencies=web,devtools --name=spring-ai-demo spring-ai-demo
```

### IDE에서 생성

대부분의 IDE는 Spring Initializr 통합 기능을 제공합니다:

#### IntelliJ IDEA에서 생성

1. 'File' > 'New' > 'Project...'를 선택합니다.
2. 'Spring Initializr'를 선택하고 위에서 언급한 설정을 적용합니다.

#### Eclipse(STS)에서 생성

1. 'File' > 'New' > 'Spring Starter Project'를 선택합니다.
2. 위에서 언급한 설정을 적용합니다.

## Spring AI 의존성 추가

### Spring AI BOM (Bill of Materials) 사용

Spring AI는 의존성 관리를 간소화하기 위해 BOM을 제공합니다. Spring AI 1.0.0 정식 버전은 Maven Central에서 바로 사용할 수 있으며, 스냅샷 버전이나 밀스톤 버전을 사용하려면 추가 리포지토리 설정이 필요합니다.

### Gradle에 Spring AI 의존성 추가

프로젝트의 build.gradle 파일에 다음 내용을 추가합니다:

#### 정식 릴리스 버전 (1.0.0)
```gradle
repositories {
    mavenCentral()
}

dependencies {
    implementation platform("org.springframework.ai:spring-ai-bom:1.0.0")
    
    // OpenAI 모델 사용 시
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
    
    // Anthropic Claude 모델 사용 시
    implementation 'org.springframework.ai:spring-ai-anthropic-spring-boot-starter'
    
    // 벡터 스토어 (선택적)
    implementation 'org.springframework.ai:spring-ai-pgvector-store-spring-boot-starter'
}
```

#### 스냅샷 버전
```gradle
repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/milestone' }
    maven { url 'https://repo.spring.io/snapshot' }
    maven {
        name = 'Central Portal Snapshots'
        url = 'https://central.sonatype.com/repository/maven-snapshots/'
    }
}

dependencies {
    implementation platform("org.springframework.ai:spring-ai-bom:1.0.0-SNAPSHOT")
    
    // 원하는 모듈 추가
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
}
```

### Maven에 Spring AI 의존성 추가

pom.xml 파일에 다음 내용을 추가합니다:

#### 정식 릴리스 버전 (1.0.0)
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
    <!-- OpenAI 모델 사용 시 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    
    <!-- Anthropic Claude 모델 사용 시 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

#### 스냅샷 버전
```xml
<repositories>
    <repository>
        <id>spring-snapshots</id>
        <name>Spring Snapshots</name>
        <url>https://repo.spring.io/snapshot</url>
        <releases>
            <enabled>false</enabled>
        </releases>
    </repository>
    <repository>
        <name>Central Portal Snapshots</name>
        <id>central-portal-snapshots</id>
        <url>https://central.sonatype.com/repository/maven-snapshots/</url>
        <releases>
            <enabled>false</enabled>
        </releases>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**주의사항**: Maven 미러 설정이 있는 경우, Spring 스냅샷 리포지토리 접근을 위해 `mirrorOf` 설정을 다음과 같이 수정해야 합니다:

```xml
<mirror>
    <id>my-mirror</id>
    <mirrorOf>*,!spring-snapshots,!central-portal-snapshots</mirrorOf>
    <url>https://my-company-repository.com/maven</url>
</mirror>
```

## AI 모델 프로바이더 설정

### 환경 변수 설정

AI 모델 프로바이더의 API 키를 환경 변수로 설정합니다. 이는 보안상의 이유로 권장되는 방법입니다.

#### Windows에서 설정

```
setx OPENAI_API_KEY "your-api-key"
setx ANTHROPIC_API_KEY "your-api-key"
```

#### macOS/Linux에서 설정

```bash
export OPENAI_API_KEY="your-api-key"
export ANTHROPIC_API_KEY="your-api-key"
```

### application.properties/application.yml 설정

`src/main/resources/application.properties` 또는 `application.yml` 파일에 다음 설정을 추가합니다:

#### application.properties

```properties
# OpenAI 설정
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4o
spring.ai.openai.chat.options.temperature=0.7

# Anthropic 설정
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-3-opus-20240229
spring.ai.anthropic.chat.options.temperature=0.7

# 로깅 설정
logging.level.org.springframework.ai=INFO
```

#### application.yml

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

logging:
  level:
    org.springframework.ai: INFO
```

## 프로젝트 구조 설정

### 권장 패키지 구조

Spring AI 프로젝트를 위한 권장 패키지 구조는 다음과 같습니다:

```
src/main/java/com/example/demo/
├── SpringAiDemoApplication.java
├── config/
│   └── AiConfig.java
├── controller/
│   └── AiController.java
├── service/
│   └── AiService.java
├── model/
│   └── AiRequest.java
│   └── AiResponse.java
└── util/
    └── PromptTemplates.java

src/main/resources/
├── application.properties
├── application.yml
└── prompts/
    └── system-message.st
```

### 기본 애플리케이션 클래스

최소한의 Spring AI 애플리케이션 설정을 위한 기본 클래스는 다음과 같습니다:

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringAiDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringAiDemoApplication.class, args);
    }

}
```

## 기본 설정 검증

### 간단한 테스트 서비스 구현

개발 환경이 올바르게 설정되었는지 확인하기 위해 간단한 테스트 서비스를 구현합니다:

```java
package com.example.demo.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.SystemPromptTemplate;
import org.springframework.ai.chat.prompt.UserPrompt;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class AiService {

    private final ChatClient chatClient;

    @Autowired
    public AiService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String generateResponse(String userInput) {
        SystemPromptTemplate systemPromptTemplate = 
            new SystemPromptTemplate("You are a helpful assistant. Your name is {name}.");
        
        Prompt prompt = new Prompt(
            systemPromptTemplate.create(Map.of("name", "Spring AI Assistant")),
            new UserPrompt(userInput)
        );
        
        ChatResponse response = chatClient.call(prompt);
        return response.getResult().getOutput().getContent();
    }
}
```

### 테스트 컨트롤러 구현

```java
package com.example.demo.controller;

import com.example.demo.service.AiService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AiController {

    private final AiService aiService;

    @Autowired
    public AiController(AiService aiService) {
        this.aiService = aiService;
    }

    @GetMapping("/ai/chat")
    public String chat(@RequestParam String message) {
        return aiService.generateResponse(message);
    }
}
```

### 애플리케이션 실행 및 테스트

1. 애플리케이션을 실행합니다:
   ```bash
   ./gradlew bootRun
   ```
   또는 Maven의 경우:
   ```bash
   ./mvnw spring-boot:run
   ```

2. 웹 브라우저나 API 클라이언트를 사용하여 다음 URL에 접속합니다:
   ```
   http://localhost:8080/ai/chat?message=Hello
   ```

3. AI 응답이 올바르게 반환되는지 확인합니다.

## 개발 환경 향상을 위한 추가 도구

### Docker Compose 통합

Spring AI는 Docker Compose를 통해 실행되는 모델 서비스나 벡터 스토어와의 연결을 자동으로 설정할 수 있습니다:

```gradle
dependencies {
    implementation 'org.springframework.ai:spring-ai-spring-boot-docker-compose'
}
```

지원되는 서비스:
- Ollama (로컬 LLM)
- Chroma (벡터 데이터베이스)
- MongoDB Atlas Local
- OpenSearch
- Qdrant
- Weaviate
- Typesense

### Testcontainers 통합

테스트 환경에서 Testcontainers를 사용하여 AI 서비스를 자동으로 프로비저닝할 수 있습니다:

```gradle
dependencies {
    testImplementation 'org.springframework.ai:spring-ai-spring-boot-testcontainers'
}
```

지원되는 컨테이너:
- OllamaContainer
- ChromaDBContainer
- MilvusContainer
- QdrantContainer
- WeaviateContainer
- 기타 벡터 데이터베이스 컨테이너

## 문제 해결

### API 키 오류

- **오류 메시지**: "Invalid API key" 또는 "Authentication error"
- **해결 방법**: 환경 변수가 올바르게 설정되었는지 확인합니다. IDE를 재시작하거나 새 터미널 세션을 시작해야 할 수도 있습니다.

### 의존성 문제

- **오류 메시지**: "Cannot resolve symbol" 또는 "ClassNotFoundException"
- **해결 방법**: Gradle이나 Maven 의존성이 올바르게 설정되었는지 확인합니다. `./gradlew clean build` 또는 `./mvnw clean install` 명령을 실행하여 프로젝트를 다시 빌드합니다.

### 연결 오류

- **오류 메시지**: "Connection timeout" 또는 "Network error"
- **해결 방법**: 인터넷 연결을 확인하고, 필요한 경우 프록시 설정을 구성합니다.

### 모델 이용 제한

- **오류 메시지**: "Rate limit exceeded" 또는 "You exceeded your current quota"
- **해결 방법**: AI 프로바이더의 사용량 제한을 확인하고, 필요한 경우 사용량 계획을 업그레이드합니다.

## 마이그레이션 가이드

### 기존 프로젝트 업그레이드

기존 Spring AI 프로젝트를 1.0.0 GA로 업그레이드하는 경우, OpenRewrite 레시피를 사용하여 자동화할 수 있습니다:

1. **OpenRewrite 플러그인 추가**:
   ```gradle
   plugins {
       id("org.openrewrite.rewrite") version "6.8.0"
   }
   ```

2. **레시피 적용**:
   ```bash
   ./gradlew rewriteRun
   ```

이 레시피는 다음과 같은 변경 사항을 자동으로 적용합니다:
- 패키지 이름 변경
- API 메서드 시그니처 업데이트
- 더 이상 사용되지 않는 클래스와 메서드 교체

자세한 내용은 Arconia Spring AI Migrations 문서를 참조하세요.

## 결론

이 장에서는 Spring AI 프로젝트 개발을 위한 환경 설정 방법을 살펴보았습니다. Java 17 이상(권장: Java 21), Spring Boot 3.4.x, Spring AI 1.0.0을 기반으로 하는 개발 환경을 구축하고, 다양한 AI 모델 프로바이더와의 연동을 설정하는 방법을 단계별로 알아보았습니다.

주요 특징:
- Spring AI 1.0.0 정식 버전(2025년 5월 20일 릴리스) Maven Central에서 바로 사용 가능
- Spring Boot 3.4.x 지원 (3.5.x 릴리스 시 지원 예정)
- Spring AI BOM을 통한 의존성 관리 간소화
- Docker Compose와 Testcontainers 통합 지원
- Spring Initializr에서 AI Models와 Vector Stores 직접 선택 가능
- OpenRewrite를 통한 자동 마이그레이션 지원

기본 설정이 완료되었으므로, 다음 장에서는 Spring AI의 아키텍처와 핵심 구성 요소에 대해 자세히 살펴보겠습니다.