# Retrieval-Augmented Generation(RAG) 패턴 구현

## 개요

Retrieval-Augmented Generation(RAG)은 대규모 언어 모델(LLM)의 한계를 극복하기 위한 핵심 패턴으로, 외부 데이터 소스에서 관련 정보를 검색하여 AI의 응답을 강화하는 기법입니다. 이 장에서는 Spring AI를 사용하여 RAG 패턴을 구현하는 방법을 자세히 살펴보겠습니다.

RAG 패턴은 다음과 같은 상황에서 특히 유용합니다:
- LLM이 학습하지 않은 최신 정보가 필요할 때
- 조직 내부의 비공개 데이터를 활용해야 할 때
- 정확한 인용과 출처가 중요한 응답이 필요할 때
- 환각(hallucination) 문제를 최소화하고자 할 때

## RAG 아키텍처 이해하기

### RAG의 핵심 구성 요소

RAG 시스템은 크게 다음 세 가지 핵심 구성 요소로 이루어집니다:

1. **문서 처리 파이프라인**
   - 문서 수집 및 로드
   - 청크(chunk) 분할
   - 임베딩 생성
   - 벡터 스토어 저장

2. **검색 시스템**
   - 쿼리 임베딩 생성
   - 유사도 검색
   - 관련 문서 추출

3. **생성 시스템**
   - 검색 결과와 사용자 쿼리 결합
   - 프롬프트 생성
   - LLM을 통한 응답 생성

### RAG 워크플로우

RAG 패턴의 일반적인 워크플로우는 다음과 같습니다:

1. **인덱싱 단계** (사전 처리)
   - 문서를 의미 있는 청크로 분할
   - 각 청크를 벡터로 변환(임베딩)
   - 벡터 데이터베이스에 저장

2. **검색 및 생성 단계** (런타임)
   - 사용자 질의 수신
   - 질의를 임베딩으로 변환
   - 관련성 높은 문서 청크 검색
   - 검색된 문서와 원본 질의를 결합하여 새로운 프롬프트 생성
   - LLM에 프롬프트 전송하여 최종 응답 생성

## Spring AI로 RAG 파이프라인 구축하기

Spring AI는 RAG 패턴 구현을 위한 포괄적인 도구를 제공합니다. 이제 Spring Boot 애플리케이션에서 RAG 시스템을 구축하는 방법을 단계별로 살펴보겠습니다.

### 프로젝트 설정

먼저 필요한 의존성을 추가합니다:

#### Gradle (build.gradle.kts)

```kotlin
repositories {
    mavenCentral()
    maven { url = uri("https://repo.spring.io/milestone") }
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.ai:spring-ai-openai-spring-boot-starter:1.0.0-M6")
    implementation("org.springframework.ai:spring-ai-document-spring-boot-starter:1.0.0-M6")
    implementation("org.springframework.ai:spring-ai-pgvector-store-spring-boot-starter:1.0.0-M6")
    
    // 문서 처리용 의존성
    implementation("org.springframework.ai:spring-ai-document-parser-tika:1.0.0-M6")
    
    // 테스트용 의존성
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.ai:spring-ai-test:1.0.0-M6")
}
```

#### Maven (pom.xml)

```xml
<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        <version>1.0.0-M6</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-document-spring-boot-starter</artifactId>
        <version>1.0.0-M6</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
        <version>1.0.0-M6</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-document-parser-tika</artifactId>
        <version>1.0.0-M6</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-test</artifactId>
        <version>1.0.0-M6</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 애플리케이션 구성

`application.yml` 파일에 필요한 설정을 추가합니다:

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-ada-002
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
    pgvector:
      url: jdbc:postgresql://localhost:5432/vectordb
      username: postgres
      password: postgres
      schema: public
      dimension: 1536   # OpenAI Ada 임베딩 차원

logging:
  level:
    org.springframework.ai: INFO
```

### 1. 문서 처리 파이프라인 구현

#### 문서 로더 및 변환기 구현

```java
package com.example.ragdemo.document;

import org.springframework.ai.document.Document;
import org.springframework.ai.document.DocumentReader;
import org.springframework.ai.document.tika.TikaDocumentReader;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.List;

@Service
public class DocumentLoaderService {

    @Value("classpath:documents/**/*.*")
    private Resource[] resources;
    
    private final DocumentReader documentReader;
    
    public DocumentLoaderService() {
        this.documentReader = new TikaDocumentReader();
    }
    
    public List<Document> loadDocuments() throws IOException {
        return documentReader.read(resources);
    }
}
```

#### 문서 분할 및 청킹

```java
package com.example.ragdemo.document;

import org.springframework.ai.document.Document;
import org.springframework.ai.document.splitter.TextSplitter;
import org.springframework.ai.document.splitter.TokenTextSplitter;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class DocumentSplitterService {

    private final TextSplitter textSplitter;
    
    public DocumentSplitterService() {
        // 청크 크기와 겹침 설정
        this.textSplitter = new TokenTextSplitter(1000, 200);
    }
    
    public List<Document> splitDocuments(List<Document> documents) {
        return documents.stream()
                .flatMap(document -> textSplitter.split(document).stream())
                .collect(Collectors.toList());
    }
}
```

#### 임베딩 및 벡터 스토어 저장

```java
package com.example.ragdemo.vectorstore;

import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.ai.vectorstore.PgVectorStore;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class VectorStoreService {

    private final EmbeddingClient embeddingClient;
    private final VectorStore vectorStore;
    
    @Autowired
    public VectorStoreService(EmbeddingClient embeddingClient, PgVectorStore vectorStore) {
        this.embeddingClient = embeddingClient;
        this.vectorStore = vectorStore;
    }
    
    public void storeDocuments(List<Document> documents) {
        vectorStore.add(documents);
    }
    
    public List<Document> similaritySearch(String query, int k) {
        return vectorStore.similaritySearch(query, k);
    }
}
```

### 2. 데이터 색인 구성 요소

데이터 색인 작업을 위한 서비스 클래스를 구현합니다:

```java
package com.example.ragdemo.indexing;

import com.example.ragdemo.document.DocumentLoaderService;
import com.example.ragdemo.document.DocumentSplitterService;
import com.example.ragdemo.vectorstore.VectorStoreService;
import jakarta.annotation.PostConstruct;
import org.springframework.ai.document.Document;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.List;

@Service
public class IndexingService {

    private final DocumentLoaderService documentLoaderService;
    private final DocumentSplitterService documentSplitterService;
    private final VectorStoreService vectorStoreService;
    
    @Autowired
    public IndexingService(
            DocumentLoaderService documentLoaderService,
            DocumentSplitterService documentSplitterService,
            VectorStoreService vectorStoreService) {
        this.documentLoaderService = documentLoaderService;
        this.documentSplitterService = documentSplitterService;
        this.vectorStoreService = vectorStoreService;
    }
    
    // 애플리케이션 시작 시 자동으로 색인 작업 수행
    @PostConstruct
    public void indexDocuments() throws IOException {
        List<Document> documents = documentLoaderService.loadDocuments();
        List<Document> splitDocuments = documentSplitterService.splitDocuments(documents);
        vectorStoreService.storeDocuments(splitDocuments);
        
        System.out.println("Indexed " + splitDocuments.size() + " document chunks");
    }
    
    // 수동 색인 작업을 위한 메서드
    public void reindexDocuments() throws IOException {
        indexDocuments();
    }
}
```

### 3. RAG 질의 처리 서비스

```java
package com.example.ragdemo.rag;

import com.example.ragdemo.vectorstore.VectorStoreService;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.document.Document;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class RagQueryService {

    private final VectorStoreService vectorStoreService;
    private final ChatClient chatClient;
    
    @Autowired
    public RagQueryService(VectorStoreService vectorStoreService, ChatClient chatClient) {
        this.vectorStoreService = vectorStoreService;
        this.chatClient = chatClient;
    }
    
    public String query(String userQuery) {
        // 1. 벡터 스토어에서 관련 문서 검색
        List<Document> relevantDocs = vectorStoreService.similaritySearch(userQuery, 5);
        
        // 2. 검색된 문서 컨텍스트 추출
        String context = relevantDocs.stream()
                .map(Document::getContent)
                .collect(Collectors.joining("\n\n"));
        
        // 3. 시스템 프롬프트 생성
        String systemPromptTemplate = """
                You are a helpful assistant that answers questions based on the provided context.
                If the answer cannot be found in the context, just say "I don't have enough information to answer this question."
                Do not make up or infer information that is not directly provided in the context.
                
                Context:
                %s
                """;
        
        SystemMessage systemMessage = new SystemMessage(String.format(systemPromptTemplate, context));
        
        // 4. 메시지 목록 구성
        List<Message> messages = new ArrayList<>();
        messages.add(systemMessage);
        messages.add(new UserMessage(userQuery));
        
        // 5. LLM 호출하여 응답 생성
        Prompt prompt = new Prompt(messages);
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
    
    public String queryWithSources(String userQuery) {
        // 1. 벡터 스토어에서 관련 문서 검색
        List<Document> relevantDocs = vectorStoreService.similaritySearch(userQuery, 5);
        
        // 2. 검색된 문서 컨텍스트 및 출처 정보 추출
        StringBuilder contextBuilder = new StringBuilder();
        for (int i = 0; i < relevantDocs.size(); i++) {
            Document doc = relevantDocs.get(i);
            contextBuilder.append("Document ").append(i + 1).append(":\n");
            contextBuilder.append("Content: ").append(doc.getContent()).append("\n");
            
            // 메타데이터에서 출처 정보 추출
            Map<String, Object> metadata = doc.getMetadata();
            if (metadata.containsKey("source")) {
                contextBuilder.append("Source: ").append(metadata.get("source")).append("\n");
            }
            
            contextBuilder.append("\n");
        }
        
        // 3. 시스템 프롬프트 생성 (출처 정보 포함)
        String systemPromptTemplate = """
                You are a helpful assistant that answers questions based on the provided context.
                If the answer cannot be found in the context, just say "I don't have enough information to answer this question."
                Do not make up or infer information that is not directly provided in the context.
                
                Always cite your sources at the end of your answer, referring to the document numbers in the context.
                For example: "Source: Document 1, Document 3"
                
                Context:
                %s
                """;
        
        SystemMessage systemMessage = new SystemMessage(String.format(systemPromptTemplate, contextBuilder.toString()));
        
        // 4. 메시지 목록 구성
        List<Message> messages = new ArrayList<>();
        messages.add(systemMessage);
        messages.add(new UserMessage(userQuery));
        
        // 5. LLM 호출하여 응답 생성
        Prompt prompt = new Prompt(messages);
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
}
```

### 4. REST API 엔드포인트

```java
package com.example.ragdemo.controller;

import com.example.ragdemo.indexing.IndexingService;
import com.example.ragdemo.rag.RagQueryService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;

@RestController
@RequestMapping("/api/rag")
public class RagController {

    private final RagQueryService ragQueryService;
    private final IndexingService indexingService;
    
    @Autowired
    public RagController(RagQueryService ragQueryService, IndexingService indexingService) {
        this.ragQueryService = ragQueryService;
        this.indexingService = indexingService;
    }
    
    @PostMapping("/query")
    public ResponseEntity<QueryResponse> query(@RequestBody QueryRequest request) {
        String answer = ragQueryService.query(request.getQuery());
        return ResponseEntity.ok(new QueryResponse(answer));
    }
    
    @PostMapping("/query-with-sources")
    public ResponseEntity<QueryResponse> queryWithSources(@RequestBody QueryRequest request) {
        String answer = ragQueryService.queryWithSources(request.getQuery());
        return ResponseEntity.ok(new QueryResponse(answer));
    }
    
    @PostMapping("/reindex")
    public ResponseEntity<String> reindex() {
        try {
            indexingService.reindexDocuments();
            return ResponseEntity.ok("Reindexing completed successfully");
        } catch (IOException e) {
            return ResponseEntity.internalServerError().body("Error during reindexing: " + e.getMessage());
        }
    }
    
    // 요청 및 응답 DTO
    public static class QueryRequest {
        private String query;
        
        public String getQuery() {
            return query;
        }
        
        public void setQuery(String query) {
            this.query = query;
        }
    }
    
    public static class QueryResponse {
        private String answer;
        
        public QueryResponse(String answer) {
            this.answer = answer;
        }
        
        public String getAnswer() {
            return answer;
        }
        
        public void setAnswer(String answer) {
            this.answer = answer;
        }
    }
}
```

## PostgreSQL pgvector 설정

RAG 시스템 구현을 위해 벡터 데이터베이스로 PostgreSQL과 pgvector 확장을 사용하는 방법을 알아보겠습니다.

### Docker를 이용한 pgvector 설정

Docker와 Docker Compose를 사용하여 pgvector가 설치된 PostgreSQL 인스턴스를 쉽게 실행할 수 있습니다:

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: vectordb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d

volumes:
  pgdata:
```

다음으로, 초기화 스크립트를 작성하여 필요한 테이블과 확장을 설정합니다:

```sql
-- init-scripts/01-init.sql
CREATE EXTENSION IF NOT EXISTS vector;

-- 벡터 저장을 위한 테이블 생성
CREATE TABLE IF NOT EXISTS vector_store (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    metadata JSONB,
    embedding vector(1536) -- OpenAI Ada 임베딩 차원
);

-- 벡터 검색을 위한 인덱스 생성
CREATE INDEX IF NOT EXISTS vector_store_embedding_idx ON vector_store USING ivfflat (embedding vector_l2_ops) WITH (lists = 100);
```

Docker Compose를 사용하여 컨테이너를 시작합니다:

```bash
docker-compose up -d
```

## 실제 사용 시나리오: 기술 문서 Q&A 시스템

이제 지금까지 구현한 RAG 시스템을 활용하여 실제 사용 사례를 살펴보겠습니다. 이 예제에서는 기술 문서에 대한 질의응답 시스템을 구현합니다.

### 문서 준비

먼저, 문서 폴더를 생성하고 기술 문서를 저장합니다:

```bash
mkdir -p src/main/resources/documents
```

다음과 같은 유형의 문서를 추가할 수 있습니다:
- PDF 기술 문서
- Markdown 파일
- HTML 페이지
- 텍스트 파일

### 문서 메타데이터 추가

문서 로드 과정에서 메타데이터를 추가하여 출처 추적을 향상시킬 수 있습니다:

```java
package com.example.ragdemo.document;

import org.springframework.ai.document.Document;
import org.springframework.ai.document.DocumentReader;
import org.springframework.ai.document.tika.TikaDocumentReader;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class DocumentLoaderService {

    @Value("classpath:documents/**/*.*")
    private Resource[] resources;
    
    private final DocumentReader documentReader;
    
    public DocumentLoaderService() {
        this.documentReader = new TikaDocumentReader();
    }
    
    public List<Document> loadDocuments() throws IOException {
        List<Document> documents = documentReader.read(resources);
        
        // 문서에 메타데이터 추가
        return documents.stream()
                .map(this::enrichMetadata)
                .collect(Collectors.toList());
    }
    
    private Document enrichMetadata(Document document) {
        Map<String, Object> metadata = new HashMap<>(document.getMetadata());
        
        // 파일 이름을 소스로 추가
        if (metadata.containsKey("filename")) {
            String filename = (String) metadata.get("filename");
            metadata.put("source", filename);
        }
        
        // 문서 유형 추가
        if (metadata.containsKey("Content-Type")) {
            String contentType = (String) metadata.get("Content-Type");
            metadata.put("document_type", contentType);
        }
        
        // 문서 생성 날짜 추가
        if (metadata.containsKey("Creation-Date")) {
            String creationDate = (String) metadata.get("Creation-Date");
            metadata.put("created_at", creationDate);
        }
        
        return new Document(document.getId(), document.getContent(), metadata);
    }
}
```

### 고급 청킹 전략

보다 지능적인 텍스트 분할이 필요한 경우, 다음과 같은 고급 청킹 전략을 구현할 수 있습니다:

```java
package com.example.ragdemo.document;

import org.springframework.ai.document.Document;
import org.springframework.ai.document.splitter.TextSplitter;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

@Service
public class AdvancedDocumentSplitterService {

    public List<Document> splitByHeadings(Document document) {
        List<Document> chunks = new ArrayList<>();
        String content = document.getContent();
        
        // 제목 패턴 (Markdown 형식)
        Pattern headingPattern = Pattern.compile("^(#{1,3})\\s+(.+)$", Pattern.MULTILINE);
        Matcher matcher = headingPattern.matcher(content);
        
        int lastIndex = 0;
        String currentHeading = "Introduction";
        int currentHeadingLevel = 0;
        
        while (matcher.find()) {
            // 새 제목을 발견했을 때 이전 섹션 저장
            if (lastIndex > 0) {
                String sectionContent = content.substring(lastIndex, matcher.start()).trim();
                if (!sectionContent.isEmpty()) {
                    Map<String, Object> metadata = new HashMap<>(document.getMetadata());
                    metadata.put("heading", currentHeading);
                    metadata.put("heading_level", currentHeadingLevel);
                    
                    chunks.add(new Document(
                            document.getId() + "-" + currentHeading.replace(" ", "-").toLowerCase(),
                            sectionContent,
                            metadata
                    ));
                }
            }
            
            // 현재 제목 정보 업데이트
            currentHeadingLevel = matcher.group(1).length();
            currentHeading = matcher.group(2);
            lastIndex = matcher.end();
        }
        
        // 마지막 섹션 처리
        if (lastIndex < content.length()) {
            String sectionContent = content.substring(lastIndex).trim();
            if (!sectionContent.isEmpty()) {
                Map<String, Object> metadata = new HashMap<>(document.getMetadata());
                metadata.put("heading", currentHeading);
                metadata.put("heading_level", currentHeadingLevel);
                
                chunks.add(new Document(
                        document.getId() + "-" + currentHeading.replace(" ", "-").toLowerCase(),
                        sectionContent,
                        metadata
                ));
            }
        }
        
        return chunks;
    }
}
```

### 하이브리드 검색 구현

의미적 검색과 키워드 검색을 결합한 하이브리드 검색 방식을 구현할 수 있습니다:

```java
package com.example.ragdemo.vectorstore;

import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Service
public class HybridSearchService {

    private final VectorStore vectorStore;
    private final JdbcTemplate jdbcTemplate;
    private final EmbeddingClient embeddingClient;
    
    @Autowired
    public HybridSearchService(
            VectorStore vectorStore,
            JdbcTemplate jdbcTemplate,
            EmbeddingClient embeddingClient) {
        this.vectorStore = vectorStore;
        this.jdbcTemplate = jdbcTemplate;
        this.embeddingClient = embeddingClient;
    }
    
    public List<Document> hybridSearch(String query, int semanticK, int keywordK) {
        // 1. 의미적 검색 (벡터 유사도)
        List<Document> semanticResults = vectorStore.similaritySearch(query, semanticK);
        
        // 2. 키워드 검색 (전체 텍스트 검색)
        List<Document> keywordResults = keywordSearch(query, keywordK);
        
        // 3. 결과 병합 및 중복 제거
        Map<String, Document> combinedResults = new HashMap<>();
        
        // 의미적 검색 결과에 가중치 부여
        for (Document doc : semanticResults) {
            combinedResults.put(doc.getId(), doc);
        }
        
        // 키워드 검색 결과 추가
        for (Document doc : keywordResults) {
            combinedResults.putIfAbsent(doc.getId(), doc);
        }
        
        return new ArrayList<>(combinedResults.values());
    }
    
    private List<Document> keywordSearch(String query, int k) {
        // PostgreSQL에서 전체 텍스트 검색 수행
        String sql = """
                SELECT id, content, metadata
                FROM vector_store
                WHERE to_tsvector('english', content) @@ plainto_tsquery('english', ?)
                LIMIT ?
                """;
        
        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            String id = rs.getString("id");
            String content = rs.getString("content");
            Map<String, Object> metadata = new HashMap<>(); // JSONB 파싱 로직 필요
            
            return new Document(id, content, metadata);
        }, query, k);
    }
}
```

## Q&A 웹 인터페이스 구현

마지막으로, 사용자가 문서에 질문을 하고 RAG 시스템의 응답을 받을 수 있는 간단한 웹 인터페이스를 구현해 보겠습니다.

### HTML 템플릿

`src/main/resources/static/index.html` 파일을 생성합니다:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>기술 문서 Q&A</title>
    <link rel="stylesheet" href="/css/style.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>기술 문서 Q&A 시스템</h1>
            <p>질문을 입력하면 기술 문서를 기반으로 답변해 드립니다.</p>
        </header>
        
        <main>
            <div class="query-section">
                <textarea id="queryInput" placeholder="질문을 입력하세요..."></textarea>
                <div class="buttons">
                    <button id="askButton">질문하기</button>
                    <button id="askWithSourcesButton">출처와 함께 질문하기</button>
                </div>
            </div>
            
            <div class="result-section" id="resultSection">
                <div class="loading" id="loading">
                    <div class="spinner"></div>
                    <p>답변을 생성하고 있습니다...</p>
                </div>
                <div class="answer" id="answer"></div>
            </div>
        </main>
    </div>
    
    <script src="/js/app.js"></script>
</body>
</html>
```

### CSS 스타일

`src/main/resources/static/css/style.css` 파일을 생성합니다:

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

body {
    background-color: #f5f5f5;
    color: #333;
    line-height: 1.6;
}

.container {
    max-width: 1000px;
    margin: 2rem auto;
    padding: 1rem;
}

header {
    text-align: center;
    margin-bottom: 2rem;
}

header h1 {
    color: #2c3e50;
    margin-bottom: 0.5rem;
}

.query-section {
    background: white;
    padding: 1.5rem;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    margin-bottom: 2rem;
}

textarea {
    width: 100%;
    height: 120px;
    padding: 0.8rem;
    border: 1px solid #ddd;
    border-radius: 4px;
    resize: vertical;
    font-size: 1rem;
    margin-bottom: 1rem;
}

.buttons {
    display: flex;
    gap: 1rem;
}

button {
    padding: 0.8rem 1.5rem;
    background-color: #3498db;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1rem;
    transition: background-color 0.3s;
}

button:hover {
    background-color: #2980b9;
}

#askWithSourcesButton {
    background-color: #2ecc71;
}

#askWithSourcesButton:hover {
    background-color: #27ae60;
}

.result-section {
    background: white;
    padding: 1.5rem;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    min-height: 200px;
}

.loading {
    display: none;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    height: 200px;
}

.spinner {
    border: 4px solid rgba(0, 0, 0, 0.1);
    width: 36px;
    height: 36px;
    border-radius: 50%;
    border-left-color: #3498db;
    animation: spin 1s linear infinite;
    margin-bottom: 1rem;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

.answer {
    font-size: 1.1rem;
    color: #333;
    line-height: 1.8;
}

code {
    background-color: #f1f1f1;
    padding: 0.2rem 0.4rem;
    border-radius: 3px;
    font-family: 'Courier New', monospace;
    font-size: 0.9em;
}

pre {
    background-color: #f1f1f1;
    padding: 1rem;
    border-radius: 4px;
    overflow-x: auto;
    margin: 1rem 0;
}

blockquote {
    border-left: 4px solid #3498db;
    padding-left: 1rem;
    margin: 1rem 0;
    color: #555;
}
```

### JavaScript 로직

`src/main/resources/static/js/app.js` 파일을 생성합니다:

```javascript
document.addEventListener('DOMContentLoaded', function() {
    const queryInput = document.getElementById('queryInput');
    const askButton = document.getElementById('askButton');
    const askWithSourcesButton = document.getElementById('askWithSourcesButton');
    const loading = document.getElementById('loading');
    const answer = document.getElementById('answer');
    
    // 마크다운 렌더링 라이브러리 로드
    const markdownIt = window.markdownit();
    
    askButton.addEventListener('click', function() {
        queryRag('/api/rag/query');
    });
    
    askWithSourcesButton.addEventListener('click', function() {
        queryRag('/api/rag/query-with-sources');
    });
    
    function queryRag(endpoint) {
        const query = queryInput.value.trim();
        
        if (!query) {
            alert('질문을 입력해주세요.');
            return;
        }
        
        // 로딩 표시
        loading.style.display = 'flex';
        answer.innerHTML = '';
        
        fetch(endpoint, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ query: query }),
        })
        .then(response => {
            if (!response.ok) {
                throw new Error('Server responded with an error: ' + response.statusText);
            }
            return response.json();
        })
        .then(data => {
            // 마크다운을 HTML로 변환
            const renderedHtml = markdownIt.render(data.answer);
            answer.innerHTML = renderedHtml;
        })
        .catch(error => {
            console.error('Error:', error);
            answer.textContent = '오류가 발생했습니다: ' + error.message;
        })
        .finally(() => {
            loading.style.display = 'none';
        });
    }
});
```

### 컨트롤러 수정

정적 리소스를 제공하기 위한 컨트롤러를 추가합니다:

```java
package com.example.ragdemo.controller;

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

## 테스트 및 최적화

RAG 시스템의 성능을 테스트하고 최적화하는 방법을 살펴보겠습니다.

### 단위 테스트 작성

```java
package com.example.ragdemo.rag;

import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import java.util.Arrays;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyInt;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

@SpringBootTest
public class RagQueryServiceTest {

    @MockBean
    private VectorStore vectorStore;
    
    @MockBean
    private ChatClient chatClient;
    
    @Test
    void testQuery() {
        // 벡터 스토어 모의 객체 설정
        List<Document> mockDocs = Arrays.asList(
            new Document("1", "Spring Boot는 스프링 프레임워크 기반의 애플리케이션을 쉽게 만들 수 있도록 도와주는 도구입니다."),
            new Document("2", "Spring AI는 다양한 AI 모델을 스프링 애플리케이션에 통합할 수 있게 해줍니다.")
        );
        when(vectorStore.similaritySearch(anyString(), anyInt())).thenReturn(mockDocs);
        
        // 채팅 클라이언트 모의 객체 설정
        ChatResponse mockResponse = mock(ChatResponse.class);
        when(mockResponse.getResult()).thenReturn(
            mock(org.springframework.ai.chat.ChatResponse.Generation.class)
        );
        when(mockResponse.getResult().getOutput()).thenReturn(
            mock(Message.class)
        );
        when(mockResponse.getResult().getOutput().getContent()).thenReturn(
            "Spring Boot는 스프링 기반 애플리케이션을 쉽게 개발할 수 있는 도구이며, Spring AI는 AI 모델을 통합할 수 있게 해줍니다."
        );
        when(chatClient.call(any(Prompt.class))).thenReturn(mockResponse);
        
        // 테스트 대상 서비스 생성
        RagQueryService ragQueryService = new RagQueryService(vectorStore, chatClient);
        
        // 테스트 실행
        String result = ragQueryService.query("스프링 부트란 무엇인가요?");
        
        // 검증
        assertTrue(result.contains("Spring Boot"));
        assertTrue(result.contains("Spring AI"));
    }
}
```

### 성능 최적화 팁

RAG 시스템의 성능을 최적화하기 위한 몇 가지 팁을 살펴보겠습니다:

1. **청크 크기 최적화**: 문서 청크 크기와 겹침을 조정하여 검색 정확도를 높입니다.
2. **검색 매개변수 튜닝**: k 값(검색할 문서 수)을 조정하여 관련성과 처리 시간의 균형을 맞춥니다.
3. **프롬프트 엔지니어링**: 시스템 프롬프트를 최적화하여 LLM이 검색된 정보를 효과적으로 활용하도록 합니다.
4. **캐싱 구현**: 자주 사용되는 쿼리에 대한 결과를 캐싱하여 응답 시간을 단축합니다.
5. **벡터 데이터베이스 인덱싱**: 적절한 벡터 인덱스를 구성하여 검색 성능을 향상시킵니다.

### 캐싱 구현 예시

```java
package com.example.ragdemo.rag;

import org.springframework.stereotype.Component;

import java.util.LinkedHashMap;
import java.util.Map;

@Component
public class QueryCache {
    private final Map<String, CacheEntry> cache;
    private final int maxSize;
    private final long expirationTimeMs;
    
    public QueryCache() {
        this(100, 1000 * 60 * 60); // 기본값: 100개 항목, 1시간 만료
    }
    
    public QueryCache(int maxSize, long expirationTimeMs) {
        this.maxSize = maxSize;
        this.expirationTimeMs = expirationTimeMs;
        
        // 간단한 LRU 캐시 구현
        this.cache = new LinkedHashMap<String, CacheEntry>(maxSize, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, CacheEntry> eldest) {
                return size() > maxSize;
            }
        };
    }
    
    public synchronized void put(String query, String answer) {
        cache.put(query, new CacheEntry(answer, System.currentTimeMillis()));
    }
    
    public synchronized String get(String query) {
        CacheEntry entry = cache.get(query);
        if (entry == null) {
            return null;
        }
        
        // 만료 확인
        if (System.currentTimeMillis() - entry.timestamp > expirationTimeMs) {
            cache.remove(query);
            return null;
        }
        
        return entry.answer;
    }
    
    private static class CacheEntry {
        final String answer;
        final long timestamp;
        
        CacheEntry(String answer, long timestamp) {
            this.answer = answer;
            this.timestamp = timestamp;
        }
    }
}
```

## 결론

이 장에서는 Spring AI를 사용하여 Retrieval-Augmented Generation(RAG) 패턴을 구현하는 방법을 살펴보았습니다. RAG는 LLM의 한계를 극복하고, 최신 정보와 도메인 특화 데이터를 활용하여 보다 정확하고 신뢰할 수 있는 AI 응답을 생성하는 강력한 패턴입니다.

Spring AI의 문서 처리, 임베딩, 벡터 스토어 기능을 활용하여 완전한 RAG 시스템을 구축하는 과정을 단계별로 살펴보았습니다. 또한 시스템 최적화 및 성능 향상을 위한 다양한 기법도 소개했습니다.

다음 장에서는 Function Calling과 `@Tool` 어노테이션을 활용하여 AI가 실제 시스템 기능을 활용할 수 있는 방법에 대해 알아보겠습니다.