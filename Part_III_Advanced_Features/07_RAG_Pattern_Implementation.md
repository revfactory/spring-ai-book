# Retrieval-Augmented Generation(RAG) 패턴 구현

## 개요

Retrieval-Augmented Generation(RAG)은 대규모 언어 모델(LLM)의 한계를 극복하기 위한 핵심 패턴으로, 외부 데이터 소스에서 관련 정보를 검색하여 AI의 응답을 강화하는 기법입니다. 이 장에서는 Spring AI 1.0.0 GA를 사용하여 RAG 패턴을 구현하는 방법을 자세히 살펴보겠습니다.

RAG 패턴은 다음과 같은 상황에서 특히 유용합니다:
- LLM이 학습하지 않은 최신 정보가 필요할 때
- 조직 내부의 비공개 데이터를 활용해야 할 때
- 정확한 인용과 출처가 중요한 응답이 필요할 때
- 환각(hallucination) 문제를 최소화하고자 할 때

Spring AI는 모듈식 아키텍처를 통해 사용자 정의 RAG 플로우를 구축하거나 `Advisor` API를 사용하여 즉시 사용 가능한 RAG 플로우를 제공합니다.

## RAG 아키텍처 이해하기

### RAG의 핵심 구성 요소

RAG 시스템은 크게 다음 네 가지 핵심 구성 요소로 이루어집니다:

1. **사전 검색(Pre-Retrieval)**
   - 쿼리 변환(Query Transformation)
   - 쿼리 확장(Query Expansion)
   - 최적의 검색 결과를 위한 쿼리 처리

2. **검색(Retrieval)**
   - 문서 검색(Document Search)
   - 문서 결합(Document Join)
   - 유사도 기반 문서 추출

3. **사후 검색(Post-Retrieval)**
   - 문서 후처리(Document Post-Processing)
   - 관련성 평가 및 순위 재조정
   - 노이즈 및 중복 제거

4. **생성(Generation)**
   - 쿼리 증강(Query Augmentation)
   - 검색 결과와 사용자 쿼리 결합
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

#### Gradle (build.gradle)

```gradle
repositories {
    mavenCentral()
}

dependencies {
    implementation platform('org.springframework.ai:spring-ai-bom:1.0.0')
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
    implementation 'org.springframework.ai:spring-ai-pgvector-store-spring-boot-starter'
    
    // RAG 및 Advisor 지원
    implementation 'org.springframework.ai:spring-ai-advisors-vector-store'
    implementation 'org.springframework.ai:spring-ai-rag'
    
    // 문서 처리용 의존성
    implementation 'org.springframework.ai:spring-ai-pdf-document-reader'
    implementation 'org.springframework.ai:spring-ai-tika-document-reader'
    
    // 테스트용 의존성
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
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-advisors-vector-store</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-rag</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pdf-document-reader</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-tika-document-reader</artifactId>
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
  datasource:
    url: jdbc:postgresql://localhost:5432/vectordb
    username: postgres
    password: postgres
  ai:
    vectorstore:
      pgvector:
        initialize-schema: true
        dimensions: 1536   # OpenAI Ada 임베딩 차원
        distance-type: COSINE_DISTANCE

logging:
  level:
    org.springframework.ai: INFO
```

### 1. QuestionAnswerAdvisor를 사용한 간단한 RAG 구현

가장 간단한 방법은 Spring AI가 제공하는 `QuestionAnswerAdvisor`를 사용하는 것입니다:

```java
package com.example.ragdemo.rag;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.QuestionAnswerAdvisor;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class SimpleRAGService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    
    @Autowired
    public SimpleRAGService(ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {
        this.vectorStore = vectorStore;
        
        // QuestionAnswerAdvisor를 기본 Advisor로 설정
        this.chatClient = chatClientBuilder
            .defaultAdvisors(QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(SearchRequest.builder()
                    .topK(5)
                    .similarityThreshold(0.7)
                    .build())
                .build())
            .build();
    }
    
    public String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
    
    // 동적 필터를 사용한 질의
    public String askWithFilter(String question, String filterExpression) {
        return chatClient.prompt()
            .user(question)
            .advisors(a -> a.param(QuestionAnswerAdvisor.FILTER_EXPRESSION, filterExpression))
            .call()
            .content();
    }
}
```

### 2. RetrievalAugmentationAdvisor를 사용한 고급 RAG 구현

Spring AI의 모듈식 RAG 아키텍처를 활용하여 더 정교한 RAG 플로우를 구현할 수 있습니다:

```java
package com.example.ragdemo.rag;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.RetrievalAugmentationAdvisor;
import org.springframework.ai.rag.retrieval.search.VectorStoreDocumentRetriever;
import org.springframework.ai.rag.augmentation.ContextualQueryAugmenter;
import org.springframework.ai.rag.preretrieval.RewriteQueryTransformer;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class AdvancedRAGService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    
    @Autowired
    public AdvancedRAGService(ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {
        this.vectorStore = vectorStore;
        
        // 고급 RAG 설정 with 쿼리 재작성
        RetrievalAugmentationAdvisor ragAdvisor = RetrievalAugmentationAdvisor.builder()
            .queryTransformers(RewriteQueryTransformer.builder()
                .chatClientBuilder(chatClientBuilder.build().mutate())
                .build())
            .documentRetriever(VectorStoreDocumentRetriever.builder()
                .vectorStore(vectorStore)
                .similarityThreshold(0.75)
                .topK(5)
                .build())
            .queryAugmenter(ContextualQueryAugmenter.builder()
                .allowEmptyContext(false)  // 빈 컨텍스트 허용 안함
                .build())
            .build();
            
        this.chatClient = chatClientBuilder
            .defaultAdvisors(ragAdvisor)
            .build();
    }
    
    public String askWithAdvancedRAG(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

### 3. 문서 처리 파이프라인 구현

문서를 로드하고 벡터 스토어에 저장하는 파이프라인을 구현합니다:

```java
package com.example.ragdemo.document;

import org.springframework.ai.document.Document;
import org.springframework.ai.reader.pdf.PagePdfDocumentReader;
import org.springframework.ai.reader.tika.TikaDocumentReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@Service
public class DocumentProcessingService {

    private final VectorStore vectorStore;
    
    @Value("classpath:documents/**/*.*")
    private Resource[] resources;
    
    @Autowired
    public DocumentProcessingService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }
    
    public void processDocuments() {
        List<Document> allDocuments = new ArrayList<>();
        
        for (Resource resource : resources) {
            String filename = resource.getFilename();
            List<Document> documents = new ArrayList<>();
            
            if (filename != null && filename.endsWith(".pdf")) {
                // PDF 문서 처리
                PagePdfDocumentReader pdfReader = new PagePdfDocumentReader(resource);
                documents = pdfReader.get();
            } else {
                // 기타 문서 처리 (Tika 사용)
                TikaDocumentReader tikaReader = new TikaDocumentReader(resource);
                documents = tikaReader.get();
            }
            
            // 메타데이터 추가
            documents.forEach(doc -> {
                Map<String, Object> metadata = doc.getMetadata();
                metadata.put("source", filename);
                metadata.put("type", getDocumentType(filename));
            });
            
            allDocuments.addAll(documents);
        }
        
        // 문서 분할
        TokenTextSplitter splitter = new TokenTextSplitter();
        List<Document> splitDocuments = splitter.apply(allDocuments);
        
        // 벡터 스토어에 저장
        vectorStore.add(splitDocuments);
        
        System.out.println("Processed " + splitDocuments.size() + " document chunks");
    }
    
    private String getDocumentType(String filename) {
        if (filename.endsWith(".pdf")) return "PDF";
        if (filename.endsWith(".md")) return "Markdown";
        if (filename.endsWith(".txt")) return "Text";
        return "Other";
    }
}
```

### 4. 모듈식 RAG 구성 요소 활용

Spring AI의 모듈식 RAG 아키텍처를 활용하여 각 단계를 커스터마이징할 수 있습니다:

#### 쿼리 변환기 (Query Transformers)

```java
package com.example.ragdemo.rag.transformers;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.rag.Query;
import org.springframework.ai.rag.preretrieval.query.transformation.*;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.messages.AssistantMessage;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class QueryTransformationService {

    private final ChatClient.Builder chatClientBuilder;
    
    public QueryTransformationService(ChatClient.Builder chatClientBuilder) {
        this.chatClientBuilder = chatClientBuilder;
    }
    
    // 대화 압축 변환기
    public Query compressConversation(String followUpQuery, List<Object> conversationHistory) {
        Query query = Query.builder()
            .text(followUpQuery)
            .history(conversationHistory.toArray())
            .build();
            
        CompressionQueryTransformer transformer = CompressionQueryTransformer.builder()
            .chatClientBuilder(chatClientBuilder)
            .build();
            
        return transformer.transform(query);
    }
    
    // 쿼리 재작성 변환기
    public Query rewriteQuery(String originalQuery) {
        RewriteQueryTransformer transformer = RewriteQueryTransformer.builder()
            .chatClientBuilder(chatClientBuilder)
            .build();
            
        return transformer.transform(new Query(originalQuery));
    }
    
    // 번역 변환기
    public Query translateQuery(String query, String targetLanguage) {
        TranslationQueryTransformer transformer = TranslationQueryTransformer.builder()
            .chatClientBuilder(chatClientBuilder)
            .targetLanguage(targetLanguage)
            .build();
            
        return transformer.transform(new Query(query));
    }
    
    // 다중 쿼리 확장
    public List<Query> expandQuery(String query) {
        MultiQueryExpander expander = MultiQueryExpander.builder()
            .chatClientBuilder(chatClientBuilder)
            .numberOfQueries(3)
            .includeOriginal(true)
            .build();
            
        return expander.expand(new Query(query));
    }
}
```

#### 문서 검색 및 결합

```java
package com.example.ragdemo.rag.retrieval;

import org.springframework.ai.document.Document;
import org.springframework.ai.rag.Query;
import org.springframework.ai.rag.retrieval.search.VectorStoreDocumentRetriever;
import org.springframework.ai.rag.retrieval.join.ConcatenationDocumentJoiner;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.filter.FilterExpressionBuilder;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;
import java.util.function.Supplier;

@Component
public class DocumentRetrievalService {

    private final VectorStore vectorStore;
    
    public DocumentRetrievalService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }
    
    // 기본 문서 검색
    public List<Document> retrieveDocuments(String query) {
        VectorStoreDocumentRetriever retriever = VectorStoreDocumentRetriever.builder()
            .vectorStore(vectorStore)
            .similarityThreshold(0.75)
            .topK(5)
            .build();
            
        return retriever.retrieve(new Query(query));
    }
    
    // 메타데이터 필터링을 사용한 검색
    public List<Document> retrieveWithFilter(String query, String category) {
        VectorStoreDocumentRetriever retriever = VectorStoreDocumentRetriever.builder()
            .vectorStore(vectorStore)
            .similarityThreshold(0.75)
            .topK(5)
            .filterExpression(new FilterExpressionBuilder()
                .eq("category", category)
                .build())
            .build();
            
        return retriever.retrieve(new Query(query));
    }
    
    // 동적 필터를 사용한 검색 (예: 멀티테넌트 환경)
    public List<Document> retrieveWithDynamicFilter(String query, Supplier<String> tenantSupplier) {
        VectorStoreDocumentRetriever retriever = VectorStoreDocumentRetriever.builder()
            .vectorStore(vectorStore)
            .filterExpression(() -> new FilterExpressionBuilder()
                .eq("tenant", tenantSupplier.get())
                .build())
            .build();
            
        return retriever.retrieve(new Query(query));
    }
    
    // 다중 쿼리 결과 결합
    public List<Document> combineMultiQueryResults(Map<Query, List<List<Document>>> documentsForQuery) {
        ConcatenationDocumentJoiner joiner = new ConcatenationDocumentJoiner();
        return joiner.join(documentsForQuery);
    }
}
```
    
### 5. 사용자 정의 프롬프트 템플릿

RAG 시스템에서 사용할 프롬프트 템플릿을 커스터마이징할 수 있습니다:

```java
package com.example.ragdemo.rag;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.QuestionAnswerAdvisor;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.ai.template.TemplateRenderer;
import org.springframework.ai.template.st.StTemplateRenderer;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

@Service
public class CustomPromptRAGService {

    private final ChatClient chatClient;
    
    public CustomPromptRAGService(ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {
        // 사용자 정의 프롬프트 템플릿 생성
        PromptTemplate customPromptTemplate = PromptTemplate.builder()
            .renderer(StTemplateRenderer.builder()
                .startDelimiterToken('<')
                .endDelimiterToken('>')
                .build())
            .template("""
                <query>
                
                다음의 컨텍스트 정보를 참고하여 질문에 답변해주세요.
                
                ---------------------
                <question_answer_context>
                ---------------------
                
                주어진 컨텍스트 정보와 사전 지식 없이 질문에 답변하세요.
                
                다음 규칙을 따르세요:
                
                1. 답변이 컨텍스트에 없으면 "제공된 정보에서 답을 찾을 수 없습니다"라고 말하세요.
                2. "컨텍스트에 따르면..." 또는 "제공된 정보에 의하면..." 같은 표현은 피하세요.
                3. 가능한 한 구체적이고 정확한 답변을 제공하세요.
                4. 답변 끝에 출처를 명시하세요.
                """)
            .build();
            
        QuestionAnswerAdvisor qaAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
            .promptTemplate(customPromptTemplate)
            .build();
            
        this.chatClient = chatClientBuilder
            .defaultAdvisors(qaAdvisor)
            .build();
    }
    
    public String askWithCustomPrompt(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

### 6. REST API 엔드포인트

```java
package com.example.ragdemo.controller;

import com.example.ragdemo.document.DocumentProcessingService;
import com.example.ragdemo.rag.SimpleRAGService;
import com.example.ragdemo.rag.AdvancedRAGService;
import com.example.ragdemo.rag.transformers.QueryTransformationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/rag")
public class RagController {

    private final SimpleRAGService simpleRAGService;
    private final AdvancedRAGService advancedRAGService;
    private final DocumentProcessingService documentProcessingService;
    private final QueryTransformationService queryTransformationService;
    
    @Autowired
    public RagController(
            SimpleRAGService simpleRAGService,
            AdvancedRAGService advancedRAGService,
            DocumentProcessingService documentProcessingService,
            QueryTransformationService queryTransformationService) {
        this.simpleRAGService = simpleRAGService;
        this.advancedRAGService = advancedRAGService;
        this.documentProcessingService = documentProcessingService;
        this.queryTransformationService = queryTransformationService;
    }
    
    @PostMapping("/query")
    public ResponseEntity<QueryResponse> query(@RequestBody QueryRequest request) {
        String answer = simpleRAGService.ask(request.getQuery());
        return ResponseEntity.ok(new QueryResponse(answer));
    }
    
    @PostMapping("/query-with-filter")
    public ResponseEntity<QueryResponse> queryWithFilter(@RequestBody FilteredQueryRequest request) {
        String answer = simpleRAGService.askWithFilter(
            request.getQuery(), 
            request.getFilterExpression()
        );
        return ResponseEntity.ok(new QueryResponse(answer));
    }
    
    @PostMapping("/query-advanced")
    public ResponseEntity<QueryResponse> queryAdvanced(@RequestBody QueryRequest request) {
        String answer = advancedRAGService.askWithAdvancedRAG(request.getQuery());
        return ResponseEntity.ok(new QueryResponse(answer));
    }
    
    @PostMapping("/query-multilingual")
    public ResponseEntity<QueryResponse> queryMultilingual(@RequestBody MultilingualQueryRequest request) {
        // 쿼리를 영어로 번역
        var translatedQuery = queryTransformationService.translateQuery(
            request.getQuery(), 
            "english"
        );
        
        // 번역된 쿼리로 검색
        String answer = simpleRAGService.ask(translatedQuery.text());
        return ResponseEntity.ok(new QueryResponse(answer));
    }
    
    @PostMapping("/process-documents")
    public ResponseEntity<String> processDocuments() {
        try {
            documentProcessingService.processDocuments();
            return ResponseEntity.ok("Document processing completed successfully");
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body("Error during document processing: " + e.getMessage());
        }
    }
    
    // 요청 및 응답 DTO
    public static class QueryRequest {
        private String query;
        
        // getter, setter
        public String getQuery() { return query; }
        public void setQuery(String query) { this.query = query; }
    }
    
    public static class FilteredQueryRequest extends QueryRequest {
        private String filterExpression;
        
        // getter, setter
        public String getFilterExpression() { return filterExpression; }
        public void setFilterExpression(String filterExpression) { 
            this.filterExpression = filterExpression; 
        }
    }
    
    public static class MultilingualQueryRequest extends QueryRequest {
        private String sourceLanguage;
        
        // getter, setter
        public String getSourceLanguage() { return sourceLanguage; }
        public void setSourceLanguage(String sourceLanguage) { 
            this.sourceLanguage = sourceLanguage; 
        }
    }
    
    public static class QueryResponse {
        private String answer;
        
        public QueryResponse(String answer) {
            this.answer = answer;
        }
        
        // getter, setter
        public String getAnswer() { return answer; }
        public void setAnswer(String answer) { this.answer = answer; }
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

volumes:
  pgdata:
```

Docker Compose를 사용하여 컨테이너를 시작합니다:

```bash
docker-compose up -d
```

Spring AI는 `initialize-schema: true` 설정을 통해 자동으로 필요한 테이블과 인덱스를 생성합니다.

## 실제 사용 시나리오: 기술 문서 Q&A 시스템

이제 지금까지 구현한 RAG 시스템을 활용하여 실제 사용 사례를 살펴보겠습니다. 이 예제에서는 기술 문서에 대한 질의응답 시스템을 구현합니다.

### 문서 준비 및 메타데이터 관리

먼저, 문서 폴더를 생성하고 기술 문서를 저장합니다:

```bash
mkdir -p src/main/resources/documents
```

다음과 같은 유형의 문서를 추가할 수 있습니다:
- PDF 기술 문서
- Markdown 파일
- HTML 페이지
- 텍스트 파일
- Word 문서 (DOCX)
- PowerPoint 프레젠테이션 (PPTX)

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

### 성능 최적화 및 모니터링

RAG 시스템의 성능을 최적화하고 모니터링하는 방법을 구현합니다:

```java
package com.example.ragdemo.monitoring;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

@Component
public class OptimizedRAGService {

    private final VectorStore vectorStore;
    private final MeterRegistry meterRegistry;
    private final Map<String, CachedResult> queryCache = new ConcurrentHashMap<>();
    
    // 캐시 설정
    private static final int CACHE_SIZE = 100;
    private static final long CACHE_TTL_MS = 3600000; // 1시간
    
    public OptimizedRAGService(VectorStore vectorStore, MeterRegistry meterRegistry) {
        this.vectorStore = vectorStore;
        this.meterRegistry = meterRegistry;
    }
    
    public List<Document> searchWithCache(String query) {
        // 캐시 확인
        CachedResult cached = queryCache.get(query);
        if (cached != null && !cached.isExpired()) {
            meterRegistry.counter("rag.cache.hits").increment();
            return cached.documents;
        }
        
        meterRegistry.counter("rag.cache.misses").increment();
        
        // 타이머로 검색 시간 측정
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            SearchRequest request = SearchRequest.builder()
                .query(query)
                .topK(5)
                .similarityThreshold(0.75)
                .build();
                
            List<Document> results = vectorStore.similaritySearch(request);
            
            // 캐시에 저장
            if (queryCache.size() >= CACHE_SIZE) {
                // 가장 오래된 항목 제거
                queryCache.entrySet().removeIf(entry -> 
                    entry.getValue().isExpired()
                );
            }
            
            queryCache.put(query, new CachedResult(results));
            
            // 검색 결과 수 기록
            meterRegistry.gauge("rag.search.results.count", results.size());
            
            return results;
            
        } finally {
            sample.stop(Timer.builder("rag.search.duration")
                .description("RAG search duration")
                .register(meterRegistry));
        }
    }
    
    // 캐시된 결과 클래스
    private static class CachedResult {
        final List<Document> documents;
        final long timestamp;
        
        CachedResult(List<Document> documents) {
            this.documents = documents;
            this.timestamp = System.currentTimeMillis();
        }
        
        boolean isExpired() {
            return System.currentTimeMillis() - timestamp > CACHE_TTL_MS;
        }
    }
    
    // 메트릭 확인을 위한 엔드포인트
    public Map<String, Object> getMetrics() {
        return Map.of(
            "cacheSize", queryCache.size(),
            "cacheHitRate", calculateCacheHitRate(),
            "activeCacheEntries", countActiveCacheEntries()
        );
    }
    
    private double calculateCacheHitRate() {
        double hits = meterRegistry.counter("rag.cache.hits").count();
        double misses = meterRegistry.counter("rag.cache.misses").count();
        double total = hits + misses;
        return total > 0 ? hits / total : 0.0;
    }
    
    private long countActiveCacheEntries() {
        return queryCache.values().stream()
            .filter(result -> !result.isExpired())
            .count();
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

## 테스트 및 검증

RAG 시스템의 품질을 테스트하고 검증하는 방법을 살펴보겠습니다.

### 통합 테스트 작성

```java
package com.example.ragdemo.rag;

import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.TestPropertySource;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;

@SpringBootTest
@TestPropertySource(properties = {
    "spring.ai.openai.api-key=test-key",
    "spring.ai.openai.chat.options.model=gpt-4o"
})
public class RAGIntegrationTest {

    @Autowired
    private SimpleRAGService simpleRAGService;
    
    @MockBean
    private VectorStore vectorStore;
    
    @Test
    void testRAGWithQuestionAnswerAdvisor() {
        // Given: 벡터 스토어에 테스트 문서 설정
        List<Document> testDocuments = List.of(
            new Document("Spring AI는 AI 애플리케이션 개발을 위한 프레임워크입니다.",
                Map.of("source", "spring-ai-docs.pdf", "page", 1)),
            new Document("RAG 패턴은 검색 증강 생성을 통해 AI 응답을 개선합니다.",
                Map.of("source", "rag-guide.pdf", "page", 5))
        );
        
        when(vectorStore.similaritySearch(any())).thenReturn(testDocuments);
        
        // When: RAG 질의 수행
        String question = "Spring AI의 RAG 패턴은 무엇인가요?";
        String answer = simpleRAGService.ask(question);
        
        // Then: 응답 검증
        assertThat(answer).isNotNull();
        assertThat(answer).isNotEmpty();
        // 실제 응답은 모킹된 ChatClient에 따라 달라짐
    }
    
    @Test
    void testRAGWithMetadataFilter() {
        // Given: 특정 소스로 필터링된 문서
        List<Document> filteredDocuments = List.of(
            new Document("Spring Boot 3.0의 새로운 기능들",
                Map.of("source", "spring-boot-3.pdf", "version", "3.0"))
        );
        
        when(vectorStore.similaritySearch(any())).thenReturn(filteredDocuments);
        
        // When: 필터가 적용된 RAG 질의
        String answer = simpleRAGService.askWithFilter(
            "Spring Boot 3.0의 새로운 기능은?",
            "version == '3.0'"
        );
        
        // Then: 응답 검증
        assertThat(answer).isNotNull();
    }
}
```

### RAG 품질 평가 메트릭

```java
package com.example.ragdemo.evaluation;

import org.springframework.ai.document.Document;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

@Component
public class RAGEvaluationService {

    // 검색 정확도 평가
    public double evaluateRetrievalAccuracy(
            List<Document> retrievedDocs,
            Set<String> expectedDocIds) {
        
        Set<String> retrievedIds = retrievedDocs.stream()
            .map(Document::getId)
            .collect(Collectors.toSet());
            
        long correctRetrievals = retrievedIds.stream()
            .filter(expectedDocIds::contains)
            .count();
            
        return (double) correctRetrievals / expectedDocIds.size();
    }
    
    // 응답 관련성 평가 (0-1 점수)
    public double evaluateResponseRelevance(
            String query,
            String response,
            List<String> expectedKeywords) {
        
        long matchedKeywords = expectedKeywords.stream()
            .filter(keyword -> response.toLowerCase()
                .contains(keyword.toLowerCase()))
            .count();
            
        return (double) matchedKeywords / expectedKeywords.size();
    }
    
    // 컨텍스트 활용도 평가
    public Map<String, Double> evaluateContextUtilization(
            String response,
            List<Document> retrievedDocs) {
        
        int totalDocs = retrievedDocs.size();
        int usedDocs = 0;
        
        for (Document doc : retrievedDocs) {
            // 간단한 휴리스틱: 문서 내용의 일부가 응답에 포함되었는지 확인
            String[] contentWords = doc.getContent().split("\\s+");
            for (String word : contentWords) {
                if (word.length() > 5 && response.contains(word)) {
                    usedDocs++;
                    break;
                }
            }
        }
        
        double utilizationRate = (double) usedDocs / totalDocs;
        
        return Map.of(
            "totalDocuments", (double) totalDocs,
            "usedDocuments", (double) usedDocs,
            "utilizationRate", utilizationRate
        );
    }
}
```

### 성능 최적화 모범 사례

RAG 시스템의 성능을 최적화하기 위한 모범 사례:

1. **청크 크기 최적화**
   - 적절한 청크 크기 선택 (일반적으로 200-500 토큰)
   - 청크 간 겹침 설정 (10-20%)
   - 문서 유형별 다른 전략 적용

2. **검색 매개변수 튜닝**
   - Top-K 값 조정 (일반적으로 3-10)
   - 유사도 임계값 설정 (0.7-0.85)
   - 필터 표현식 최적화

3. **쿼리 전처리**
   - 쿼리 재작성 및 확장
   - 다국어 지원을 위한 번역
   - 문맥 압축 기술 활용

4. **캐싱 전략**
   - 쿼리 결과 캐싱
   - 임베딩 캐싱
   - TTL 기반 캐시 무효화

5. **인프라 최적화**
   - 벡터 인덱스 선택 (HNSW, IVF)
   - 배치 처리 활용
   - 비동기 처리 구현

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

이 장에서는 Spring AI 1.0.0 GA를 사용하여 Retrieval-Augmented Generation(RAG) 패턴을 구현하는 방법을 살펴보았습니다. 

주요 학습 내용:

1. **Advisor API 활용**: `QuestionAnswerAdvisor`와 `RetrievalAugmentationAdvisor`를 통한 즉시 사용 가능한 RAG 구현

2. **모듈식 RAG 아키텍처**: 사전 검색, 검색, 사후 검색, 생성 단계별 커스터마이징 가능한 구성 요소

3. **쿼리 변환 및 확장**: 다국어 지원, 쿼리 재작성, 다중 쿼리 확장을 통한 검색 품질 향상

4. **성능 최적화**: 캐싱, 메트릭 모니터링, 배치 처리를 통한 시스템 성능 개선

5. **품질 평가**: RAG 시스템의 검색 정확도와 응답 품질을 측정하는 평가 메트릭

Spring AI의 RAG 기능은 프로덕션 환경에서 안정적으로 사용할 수 있는 엔터프라이즈급 솔루션을 제공합니다. 모듈식 아키텍처를 통해 각 조직의 요구사항에 맞게 유연하게 커스터마이징할 수 있으며, Advisor API를 통해 간단하게 시작할 수 있습니다.

다음 장에서는 Function Calling과 Tool 어노테이션을 활용하여 AI가 실제 시스템 기능을 호출하고 활용할 수 있는 방법에 대해 알아보겠습니다.