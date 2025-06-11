# 임베딩과 벡터 스토어

## 개요

이 장에서는 Spring AI의 핵심 기능 중 하나인 임베딩(Embeddings)과 벡터 스토어(Vector Stores)에 대해 알아봅니다. 임베딩은 텍스트, 이미지, 비디오를 고차원 벡터로 변환하는 기술이며, 벡터 스토어는 이러한 벡터를 저장하고 유사성 검색을 수행하는 특수한 데이터베이스입니다. 이들은 RAG(Retrieval Augmented Generation) 패턴의 핵심 구성 요소로, AI 애플리케이션에 맥락 정보를 제공하는 데 필수적입니다.

## 임베딩의 이해

### 임베딩이란?

임베딩은 텍스트, 이미지, 또는 비디오와 같은 데이터를 수치적 표현으로 변환한 것입니다. 구체적으로는 부동 소수점 숫자의 배열(벡터)로 표현됩니다. 이러한 벡터는 원본 데이터의 의미를 포착하도록 설계되어 있습니다.

주요 특징:
- **벡터 차원**: 임베딩 배열의 길이를 벡터의 차원이라고 합니다
- **의미적 유사성**: 두 벡터 간의 수치적 거리가 가까울수록 의미적으로 유사합니다
- **범용성**: 다양한 AI 작업(의미 분석, 텍스트 분류 등)에 활용됩니다

### 임베딩의 작동 원리

임베딩 모델은 다음과 같은 과정을 거쳐 작동합니다:

1. **입력 처리**: 텍스트를 토큰으로 분할
2. **벡터 변환**: 각 토큰을 수치 벡터로 변환
3. **의미 포착**: 문맥과 관계를 반영한 최종 벡터 생성

```java
// 임베딩 생성 예시
String text = "Spring AI는 AI 애플리케이션 개발을 간소화합니다.";
float[] embedding = embeddingModel.embed(text);
// embedding은 [0.1, -0.2, 0.3, ...] 형태의 벡터
```

## Spring AI Embedding API

### EmbeddingModel 인터페이스

Spring AI는 다양한 임베딩 모델과의 통합을 위해 `EmbeddingModel` 인터페이스를 제공합니다:

```java
public interface EmbeddingModel extends Model<EmbeddingRequest, EmbeddingResponse> {
    
    @Override
    EmbeddingResponse call(EmbeddingRequest request);
    
    // 단일 문서 임베딩
    float[] embed(Document document);
    
    // 단일 텍스트 임베딩
    default float[] embed(String text) {
        return this.embed(List.of(text)).iterator().next();
    }
    
    // 배치 텍스트 임베딩
    default List<float[]> embed(List<String> texts) {
        return this.call(new EmbeddingRequest(texts, EmbeddingOptions.EMPTY))
            .getResults()
            .stream()
            .map(Embedding::getOutput)
            .toList();
    }
    
    // 벡터 차원 확인
    default int dimensions() {
        return embed("Test String").length;
    }
}
```

### 지원되는 임베딩 모델

Spring AI 1.0.0 GA는 다음과 같은 임베딩 모델을 지원합니다:

1. **OpenAI Embeddings** - text-embedding-ada-002 등
2. **Azure OpenAI Embeddings** - Azure에서 호스팅되는 OpenAI 모델
3. **Ollama Embeddings** - 로컬 실행 가능한 오픈소스 모델
4. **Transformers (ONNX) Embeddings** - ONNX 런타임 기반 모델
5. **PostgresML Embeddings** - PostgreSQL 내장 ML 모델
6. **Bedrock Cohere/Titan Embeddings** - AWS Bedrock 모델
7. **VertexAI Embeddings** - Google Cloud 모델
8. **Mistral AI Embeddings** - Mistral AI 모델

## 임베딩 모델 구현

### OpenAI 임베딩 설정

가장 널리 사용되는 OpenAI 임베딩 모델을 설정해보겠습니다:

#### 의존성 추가

```gradle
dependencies {
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
}
```

#### 설정

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-ada-002
```

#### 사용 예제

```java
package com.example.embedding;

import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.embedding.EmbeddingResponse;
import org.springframework.ai.embedding.EmbeddingRequest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class EmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    
    @Autowired
    public EmbeddingService(EmbeddingModel embeddingModel) {
        this.embeddingModel = embeddingModel;
    }
    
    public float[] generateEmbedding(String text) {
        return embeddingModel.embed(text);
    }
    
    public List<float[]> generateBatchEmbeddings(List<String> texts) {
        return embeddingModel.embed(texts);
    }
    
    public EmbeddingResponse generateEmbeddingWithMetadata(List<String> texts) {
        return embeddingModel.embedForResponse(texts);
    }
    
    public void printEmbeddingInfo(String text) {
        float[] embedding = generateEmbedding(text);
        System.out.println("텍스트: " + text);
        System.out.println("임베딩 차원: " + embedding.length);
        System.out.println("처음 5개 값: " + 
            java.util.Arrays.toString(
                java.util.Arrays.copyOfRange(embedding, 0, 5)
            ));
    }
}
```

### 로컬 임베딩 모델 (Ollama)

프라이버시가 중요하거나 오프라인 환경에서는 Ollama를 사용할 수 있습니다:

```java
package com.example.embedding;

import org.springframework.ai.ollama.OllamaEmbeddingModel;
import org.springframework.ai.ollama.api.OllamaApi;
import org.springframework.ai.ollama.api.OllamaOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OllamaEmbeddingConfig {
    
    @Bean
    public OllamaEmbeddingModel ollamaEmbeddingModel() {
        var ollamaApi = new OllamaApi("http://localhost:11434");
        return new OllamaEmbeddingModel(ollamaApi, 
            OllamaOptions.builder()
                .withModel("nomic-embed-text")
                .build());
    }
}
```

## 벡터 스토어

### 벡터 스토어란?

벡터 스토어는 벡터 임베딩을 저장하고 유사성 검색을 수행하는 특수한 데이터베이스입니다. 전통적인 데이터베이스와 달리:

- **유사성 검색**: 정확한 일치가 아닌 유사도 기반 검색
- **고차원 벡터 지원**: 수백에서 수천 차원의 벡터 처리
- **효율적인 인덱싱**: 대규모 벡터 집합에서 빠른 검색

### VectorStore 인터페이스

Spring AI는 다양한 벡터 스토어 구현체를 위한 통일된 인터페이스를 제공합니다:

```java
public interface VectorStore extends DocumentWriter {
    
    default String getName() {
        return this.getClass().getSimpleName();
    }
    
    // 문서 추가
    void add(List<Document> documents);
    
    // ID로 삭제
    void delete(List<String> idList);
    
    // 필터 표현식으로 삭제
    void delete(Filter.Expression filterExpression);
    
    // 유사성 검색
    List<Document> similaritySearch(String query);
    
    // 고급 검색 옵션
    List<Document> similaritySearch(SearchRequest request);
    
    // 네이티브 클라이언트 접근
    default <T> Optional<T> getNativeClient() {
        return Optional.empty();
    }
}
```

### SearchRequest 빌더

유사성 검색을 위한 상세한 옵션을 설정할 수 있습니다:

```java
SearchRequest request = SearchRequest.builder()
    .query("Spring AI의 주요 기능은 무엇인가요?")
    .topK(5)  // 상위 5개 결과
    .similarityThreshold(0.7)  // 유사도 임계값
    .filterExpression("category == 'documentation'")  // 메타데이터 필터
    .build();

List<Document> results = vectorStore.similaritySearch(request);
```

## 벡터 스토어 구현

### PGVector 설정 및 사용

PostgreSQL의 pgvector 확장을 사용한 벡터 스토어 구현:

#### 의존성 추가

```gradle
dependencies {
    implementation 'org.springframework.ai:spring-ai-pgvector-store-spring-boot-starter'
    implementation 'org.postgresql:postgresql'
}
```

#### 설정

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/vectordb
    username: postgres
    password: postgres
  ai:
    vectorstore:
      pgvector:
        initialize-schema: true
        dimensions: 1536  # OpenAI text-embedding-ada-002 차원
        distance-type: COSINE_DISTANCE
```

#### 구현 예제

```java
package com.example.vectorstore;

import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class DocumentService {
    
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;
    
    @Autowired
    public DocumentService(VectorStore vectorStore, EmbeddingModel embeddingModel) {
        this.vectorStore = vectorStore;
        this.embeddingModel = embeddingModel;
    }
    
    public void addDocuments(List<String> contents, Map<String, Object> metadata) {
        List<Document> documents = contents.stream()
            .map(content -> new Document(content, metadata))
            .toList();
        
        vectorStore.add(documents);
    }
    
    public List<Document> searchSimilarDocuments(String query, int topK) {
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(topK)
            .similarityThreshold(0.7)
            .build();
            
        return vectorStore.similaritySearch(request);
    }
    
    public void deleteDocumentsByFilter(String filterExpression) {
        vectorStore.delete(filterExpression);
    }
}
```

### 문서 로딩과 처리

실제 문서를 벡터 스토어에 저장하는 완전한 예제:

```java
package com.example.vectorstore;

import org.springframework.ai.document.Document;
import org.springframework.ai.reader.TextReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.Resource;
import org.springframework.core.io.ResourceLoader;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class DocumentIngestionService {
    
    private final VectorStore vectorStore;
    private final ResourceLoader resourceLoader;
    
    @Autowired
    public DocumentIngestionService(VectorStore vectorStore, ResourceLoader resourceLoader) {
        this.vectorStore = vectorStore;
        this.resourceLoader = resourceLoader;
    }
    
    public void ingestDocument(String filePath, Map<String, Object> metadata) {
        // 문서 읽기
        Resource resource = resourceLoader.getResource(filePath);
        TextReader textReader = new TextReader(resource);
        List<Document> documents = textReader.get();
        
        // 문서 분할 (토큰 기반)
        TokenTextSplitter splitter = new TokenTextSplitter();
        List<Document> splitDocuments = splitter.apply(documents);
        
        // 메타데이터 추가
        splitDocuments.forEach(doc -> doc.getMetadata().putAll(metadata));
        
        // 벡터 스토어에 저장
        vectorStore.add(splitDocuments);
        
        System.out.println("문서 수집 완료: " + splitDocuments.size() + "개 청크 저장됨");
    }
}
```

## 배치 처리 전략

### TokenCountBatchingStrategy

대량의 문서를 처리할 때는 토큰 제한을 고려한 배치 처리가 필요합니다:

```java
package com.example.vectorstore;

import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.BatchingStrategy;
import org.springframework.ai.embedding.TokenCountBatchingStrategy;
import org.springframework.ai.tokenizer.JTokkitTokenCountEstimator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class BatchingConfig {
    
    @Bean
    public BatchingStrategy customBatchingStrategy() {
        return new TokenCountBatchingStrategy(
            EncodingType.CL100K_BASE,  // 인코딩 타입
            8000,  // 최대 입력 토큰 수
            0.1    // 예약 비율 (10%)
        );
    }
}
```

### 사용자 정의 배치 전략

특수한 요구사항을 위한 사용자 정의 배치 전략:

```java
public class CustomBatchingStrategy implements BatchingStrategy {
    
    private static final int MAX_BATCH_SIZE = 100;
    private static final int MAX_CONTENT_LENGTH = 1000;
    
    @Override
    public List<List<Document>> batch(List<Document> documents) {
        List<List<Document>> batches = new ArrayList<>();
        List<Document> currentBatch = new ArrayList<>();
        int currentLength = 0;
        
        for (Document doc : documents) {
            int docLength = doc.getContent().length();
            
            if (currentBatch.size() >= MAX_BATCH_SIZE || 
                currentLength + docLength > MAX_CONTENT_LENGTH) {
                if (!currentBatch.isEmpty()) {
                    batches.add(new ArrayList<>(currentBatch));
                    currentBatch.clear();
                    currentLength = 0;
                }
            }
            
            currentBatch.add(doc);
            currentLength += docLength;
        }
        
        if (!currentBatch.isEmpty()) {
            batches.add(currentBatch);
        }
        
        return batches;
    }
}
```

## 메타데이터 필터링

### Filter.Expression 사용

복잡한 메타데이터 필터링을 위한 표현식 빌더:

```java
package com.example.vectorstore;

import org.springframework.ai.vectorstore.filter.Filter;
import org.springframework.ai.vectorstore.filter.FilterExpressionBuilder;
import org.springframework.stereotype.Component;

@Component
public class FilterExamples {
    
    public void demonstrateFilters() {
        FilterExpressionBuilder builder = new FilterExpressionBuilder();
        
        // 단순 필터
        Filter.Expression simpleFilter = builder
            .eq("category", "documentation")
            .build();
        
        // 복합 필터
        Filter.Expression complexFilter = builder
            .and(
                builder.eq("type", "tutorial"),
                builder.gte("year", 2024)
            )
            .build();
        
        // IN 연산자 사용
        Filter.Expression inFilter = builder
            .in("tags", "spring", "ai", "embeddings")
            .build();
        
        // NOT 연산자 사용
        Filter.Expression notFilter = builder
            .not(builder.eq("status", "deprecated"))
            .build();
    }
}
```

### 문자열 기반 필터

SQL과 유사한 문자열 기반 필터 표현식:

```java
// 단순 필터
String filter1 = "category == 'documentation'";

// AND 조건
String filter2 = "type == 'tutorial' && year >= 2024";

// IN 연산자
String filter3 = "tags in ['spring', 'ai', 'embeddings']";

// 복합 조건
String filter4 = "category == 'documentation' && status != 'deprecated' && year >= 2023";

List<Document> results = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("Spring AI 튜토리얼")
        .filterExpression(filter4)
        .build()
);
```

## RAG 패턴 구현

### 기본 RAG 구현

임베딩과 벡터 스토어를 활용한 RAG 패턴 구현:

```java
package com.example.rag;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class RAGService {
    
    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    
    @Autowired
    public RAGService(ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {
        this.chatClient = chatClientBuilder.build();
        this.vectorStore = vectorStore;
    }
    
    public String askWithContext(String question) {
        // 1. 관련 문서 검색
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(3)
                .similarityThreshold(0.7)
                .build()
        );
        
        // 2. 컨텍스트 구성
        String context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));
        
        // 3. 프롬프트 생성 및 응답
        return chatClient.prompt()
            .system("당신은 주어진 컨텍스트를 기반으로 질문에 답변하는 도우미입니다. " +
                   "컨텍스트에 없는 정보는 추측하지 마세요.")
            .user(u -> u.text("""
                컨텍스트:
                {context}
                
                질문: {question}
                """)
                .param("context", context)
                .param("question", question))
            .call()
            .content();
    }
}
```

### 고급 RAG with 메타데이터

메타데이터를 활용한 더 정교한 RAG 구현:

```java
package com.example.rag;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.util.List;
import java.util.stream.Collectors;

@Service
public class AdvancedRAGService {
    
    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    
    public AdvancedRAGService(ChatClient.Builder chatClientBuilder, 
                              VectorStore vectorStore) {
        this.chatClient = chatClientBuilder.build();
        this.vectorStore = vectorStore;
    }
    
    public String askWithMetadataFilter(String question, String category, int year) {
        // 메타데이터 필터를 포함한 검색
        String filterExpression = String.format(
            "category == '%s' && year >= %d", category, year
        );
        
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(5)
                .similarityThreshold(0.75)
                .filterExpression(filterExpression)
                .build()
        );
        
        // 문서 정보와 함께 컨텍스트 구성
        String context = relevantDocs.stream()
            .map(doc -> String.format("[출처: %s, 날짜: %s]\n%s",
                doc.getMetadata().get("source"),
                doc.getMetadata().get("date"),
                doc.getContent()))
            .collect(Collectors.joining("\n\n---\n\n"));
        
        return chatClient.prompt()
            .system("제공된 컨텍스트를 기반으로 정확하게 답변하세요. " +
                   "각 정보의 출처를 명시하세요.")
            .user(u -> u.text("""
                컨텍스트:
                {context}
                
                질문: {question}
                
                답변 시 출처를 명시하고, 최신 정보를 우선적으로 사용하세요.
                """)
                .param("context", context)
                .param("question", question))
            .call()
            .content();
    }
}
```

## 성능 최적화

### 벡터 스토어 인덱싱

효율적인 검색을 위한 인덱스 설정:

```java
@Configuration
public class VectorStoreOptimization {
    
    @Bean
    public PgVectorStore optimizedPgVectorStore(
            JdbcTemplate jdbcTemplate, 
            EmbeddingModel embeddingModel) {
        
        return PgVectorStore.builder(jdbcTemplate, embeddingModel)
            .tableName("document_embeddings")
            .schemaName("public")
            .dimensions(1536)
            .distanceType(PgVectorStore.PgDistanceType.COSINE_DISTANCE)
            .removeExistingVectorStoreTable(false)
            .indexType(PgVectorStore.PgIndexType.HNSW)  // 고성능 인덱스
            .build();
    }
}
```

### 캐싱 전략

자주 사용되는 임베딩 캐싱:

```java
@Service
public class CachedEmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    private final Map<String, float[]> embeddingCache = new ConcurrentHashMap<>();
    
    public float[] getEmbedding(String text) {
        return embeddingCache.computeIfAbsent(text, 
            key -> embeddingModel.embed(key));
    }
    
    @Scheduled(fixedDelay = 3600000)  // 1시간마다
    public void clearCache() {
        embeddingCache.clear();
    }
}
```

## 문제 해결

### 일반적인 문제와 해결책

1. **토큰 제한 초과**
   - 문제: "Maximum token limit exceeded" 오류
   - 해결: TokenTextSplitter를 사용하여 문서를 작은 청크로 분할

2. **벡터 차원 불일치**
   - 문제: "Vector dimension mismatch" 오류
   - 해결: 임베딩 모델과 벡터 스토어의 차원 설정 확인

3. **유사성 검색 결과 없음**
   - 문제: 검색 결과가 항상 비어있음
   - 해결: similarityThreshold 값을 낮추거나 인덱스 재구성

4. **메모리 부족**
   - 문제: 대량 문서 처리 시 OutOfMemoryError
   - 해결: 배치 처리와 스트리밍 방식 활용

## 결론

이 장에서는 Spring AI의 임베딩과 벡터 스토어 기능을 살펴보았습니다. 주요 학습 내용:

1. **임베딩 모델**: 텍스트를 벡터로 변환하는 다양한 모델 지원
2. **벡터 스토어**: 19개 이상의 벡터 데이터베이스 통합
3. **유사성 검색**: 메타데이터 필터링을 포함한 고급 검색 기능
4. **배치 처리**: 대량 문서 처리를 위한 효율적인 전략
5. **RAG 패턴**: 임베딩과 벡터 스토어를 활용한 컨텍스트 기반 AI 응답

Spring AI 1.0.0 GA는 프로덕션 환경에서 사용할 수 있는 강력하고 유연한 임베딩 및 벡터 스토어 기능을 제공합니다. 이러한 기능을 활용하면 기업 데이터를 AI 모델과 효과적으로 통합할 수 있습니다.

다음 장에서는 이러한 기반 기술을 활용한 고급 RAG 패턴 구현에 대해 더 자세히 알아보겠습니다.