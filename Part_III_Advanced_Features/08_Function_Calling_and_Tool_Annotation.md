# Tool Calling & `@Tool` 어노테이션

## 개요

Tool Calling(이전 명칭: Function Calling)은 최신 LLM이 제공하는 가장 강력한 기능 중 하나로, AI가 텍스트 응답을 생성하는 것을 넘어 정의된 도구를 호출할 수 있게 해줍니다. Spring AI 1.0.0 GA에서는 이 기능을 효과적으로 활용할 수 있도록 개선된 Tool API와 `@Tool` 어노테이션을 제공합니다.

이 장에서는 Spring AI의 Tool Calling 기능을 살펴보고, `@Tool` 어노테이션을 사용하여 Java 메서드를 AI 모델의 도구로 노출하는 방법을 배우게 됩니다. 이를 통해 AI가 외부 데이터를 조회하거나 애플리케이션의 특정 기능을 실행하는 복잡한 시스템을 구현할 수 있습니다.

## Tool Calling 이해하기

### Tool Calling의 개념

Tool Calling(도구 호출)은 LLM이 텍스트 응답을 생성하는 대신, 미리 정의된 도구(API, 함수 등)를 호출하여 필요한 정보를 얻거나 작업을 수행할 수 있게 하는 기능입니다. 이 기능을 통해 LLM은:

1. 사용자의 의도를 파악하여 적절한 도구를 선택할 수 있습니다.
2. 도구 호출에 필요한 매개변수를 사용자 입력에서 추출할 수 있습니다.
3. 도구의 실행 결과를 기반으로 추가적인 응답을 생성할 수 있습니다.

중요한 점은 보안상의 이유로 모델이 직접 API에 접근하지 않으며, 클라이언트 애플리케이션이 도구 호출 로직을 제공한다는 것입니다. 모델은 도구 호출을 요청하고 입력 인수를 제공할 뿐이며, 애플리케이션이 도구를 실행하고 결과를 반환합니다.

### Tool Calling의 주요 사용 사례

Tool Calling은 다음 두 가지 주요 사용 사례에서 활용됩니다:

1. **정보 검색(Information Retrieval)**: 
   - 데이터베이스, 웹 서비스, 파일 시스템, 검색 엔진 등에서 정보 검색
   - RAG(Retrieval Augmented Generation) 시나리오에서 활용
   - 예: 날씨 정보 조회, 최신 뉴스 검색, 데이터베이스 레코드 조회

2. **작업 수행(Taking Action)**:
   - 소프트웨어 시스템에서 특정 작업 실행
   - 이메일 발송, 데이터베이스 레코드 생성, 양식 제출, 워크플로우 트리거
   - 예: 고객 채팅봇에서 항공편 예약, 웹 페이지 양식 작성, TDD 시나리오에서 자동 코드 생성

### 주요 AI 모델 프로바이더의 Tool Calling 지원

Spring AI는 다양한 AI 모델 프로바이더의 Tool Calling 기능을 추상화하여 제공합니다:

| 프로바이더 | 지원 모델 | 특징 |
|------------|----------|------|
| OpenAI | GPT-3.5, GPT-4, GPT-4o | 최초로 Function Calling 개념 도입, 다중 도구 호출 지원 |
| Anthropic | Claude 3, Claude 3.5 | Tool Use로 불리며, 고급 도구 사용 기능 제공 |
| Amazon Bedrock | Claude, Titan, Llama | AWS를 통한 통합 지원, Converse API 활용 |
| Azure OpenAI | GPT-3.5, GPT-4, GPT-4o | Microsoft Azure 환경에서의 Tool Calling |
| Google Vertex AI | Gemini Pro, Gemini Ultra | 멀티모달 Tool Calling 지원 |
| Groq | Llama, Mixtral | 빠른 추론 속도로 도구 호출 처리 |
| Ollama | Llama 3, Mistral | 로컬 환경에서의 Tool Calling 지원 |

## Spring AI의 Tool API

Spring AI는 유연한 Tool 정의 방법을 제공합니다. 메서드나 함수를 도구로 변환하는 두 가지 주요 접근 방식이 있습니다:

1. **선언적 방식**: `@Tool` 어노테이션 사용
2. **프로그래밍 방식**: `MethodToolCallback` 또는 `FunctionToolCallback` 사용

### Quick Start - 간단한 예제

먼저 간단한 예제로 Tool Calling을 시작해보겠습니다:

```java
import java.time.LocalDateTime;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.context.i18n.LocaleContextHolder;

class DateTimeTools {

    @Tool(description = "Get the current date and time in the user's timezone")
    String getCurrentDateTime() {
        return LocalDateTime.now().atZone(LocaleContextHolder.getTimeZone().toZoneId()).toString();
    }
}
```

이제 이 도구를 ChatClient와 함께 사용해봅시다:

```java
ChatModel chatModel = ...

String response = ChatClient.create(chatModel)
        .prompt("What day is tomorrow?")
        .tools(new DateTimeTools())
        .call()
        .content();

System.out.println(response);
// 출력: Tomorrow is 2024-11-07.
```

### 선언적 방식: `@Tool` 어노테이션

`@Tool` 어노테이션을 사용하면 메서드를 간단하게 도구로 변환할 수 있습니다:

```java
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;

class WeatherTools {
    
    @Tool(description = "Get current weather information for a location")
    WeatherInfo getCurrentWeather(
        @ToolParam(description = "City name") String city,
        @ToolParam(description = "Country code (optional)", required = false) String countryCode
    ) {
        // 날씨 API 호출 및 정보 반환
        return weatherApi.getWeatherInfo(city, countryCode);
    }
}
```

`@Tool` 어노테이션의 주요 속성:
- `name`: 도구의 이름 (기본값: 메서드 이름)
- `description`: 도구의 설명 (AI 모델이 도구의 용도를 이해하는 데 중요)
- `returnDirect`: 결과를 모델에 전달하지 않고 직접 반환할지 여부 (기본값: false)
- `resultConverter`: 결과 변환에 사용할 `ToolCallResultConverter` 클래스

### `@ToolParam` 어노테이션

`@ToolParam` 어노테이션으로 매개변수에 대한 메타데이터를 제공할 수 있습니다:

```java
@Tool(description = "Set a user alarm for the given time")
void setAlarm(
    @ToolParam(description = "Time in ISO-8601 format") String time,
    @ToolParam(description = "Alarm label", required = false) String label
) {
    LocalDateTime alarmTime = LocalDateTime.parse(time, DateTimeFormatter.ISO_DATE_TIME);
    System.out.println("Alarm set for " + alarmTime + (label != null ? " - " + label : ""));
}
```

`@ToolParam` 속성:
- `description`: 매개변수 설명 (형식, 허용 값 등을 명시)
- `required`: 필수 여부 (기본값: true)

매개변수가 `@Nullable`로 표시되면 `@ToolParam`에서 명시적으로 required로 표시하지 않는 한 선택적으로 처리됩니다.

### 도구 매개변수 지원 타입

메서드 기반 도구는 다양한 타입을 지원합니다:

- **기본 타입**: `int`, `long`, `double`, `boolean`, `String` 등
- **배열 및 컬렉션**: `List<T>`, `Set<T>`, `Map<K,V>`, `T[]` 등
- **객체**: POJO, Record 타입 (직렬화 가능한 객체)
- **열거형**: `enum` 타입
- **날짜/시간**: `LocalDate`, `LocalDateTime`, `ZonedDateTime` 등

지원하지 않는 타입:
- `Optional` (대신 `@Nullable` 사용)
- 비동기 타입 (`CompletableFuture`, `Future`)
- 리액티브 타입 (`Mono`, `Flux`)
- 함수형 타입 (`Function`, `Supplier`, `Consumer`)

### JSON Schema 생성 및 검증

Spring AI는 자동으로 입력 매개변수의 JSON Schema를 생성합니다. 추가적인 검증을 위해 다양한 어노테이션을 사용할 수 있습니다:

```java
import jakarta.validation.constraints.*;
import com.fasterxml.jackson.annotation.JsonProperty;
import io.swagger.v3.oas.annotations.media.Schema;

@Tool(description = "Search for books in our catalog")
public List<Book> searchBooks(
    @ToolParam(description = "Search query") 
    @NotBlank String query,
    
    @ToolParam(description = "Maximum number of results") 
    @Min(1) @Max(50) 
    int limit,
    
    @ToolParam(description = "Filter by genre", required = false) 
    @Pattern(regexp = "^(fiction|non-fiction|sci-fi|mystery|biography)$") 
    String genre,
    
    @ToolParam(description = "Price range filter")
    PriceRange priceRange
) {
    return bookRepository.findBooks(query, genre, limit, priceRange);
}

record PriceRange(
    @Schema(description = "Minimum price in USD", minimum = "0") 
    @JsonProperty("min_price") 
    double minPrice,
    
    @Schema(description = "Maximum price in USD", maximum = "1000") 
    @JsonProperty("max_price") 
    double maxPrice
) {}

### 프로그래밍 방식: MethodToolCallback

프로그래밍 방식으로 메서드를 도구로 변환할 수도 있습니다:

```java
import org.springframework.ai.tool.MethodToolCallback;
import org.springframework.ai.tool.ToolDefinition;
import org.springframework.util.ReflectionUtils;

Method method = ReflectionUtils.findMethod(DateTimeTools.class, "getCurrentDateTime");
ToolCallback toolCallback = MethodToolCallback.builder()
    .toolDefinition(ToolDefinition.builder(method)
            .description("Get the current date and time in the user's timezone")
            .build())
    .toolMethod(method)
    .toolObject(new DateTimeTools())
    .build();
```

`MethodToolCallback.Builder`의 주요 옵션:
- `toolDefinition`: 도구 이름, 설명, 입력 스키마 정의
- `toolMethod`: 도구로 사용할 메서드
- `toolObject`: 메서드를 포함하는 객체 인스턴스 (정적 메서드의 경우 생략)
- `toolCallResultConverter`: 결과 변환기

### Function을 도구로 사용하기

Spring AI는 함수형 인터페이스(`Function`, `Supplier`, `Consumer`, `BiFunction`)도 도구로 사용할 수 있게 지원합니다:

```java
import org.springframework.ai.tool.FunctionToolCallback;

public class WeatherService implements Function<WeatherRequest, WeatherResponse> {
    public WeatherResponse apply(WeatherRequest request) {
        return new WeatherResponse(30.0, Unit.C);
    }
}

public enum Unit { C, F }
public record WeatherRequest(String location, Unit unit) {}
public record WeatherResponse(double temp, Unit unit) {}

// FunctionToolCallback으로 변환
ToolCallback toolCallback = FunctionToolCallback
    .builder("currentWeather", new WeatherService())
    .description("Get the weather in location")
    .inputType(WeatherRequest.class)
    .build();
```

### Spring Bean으로 도구 정의하기

도구를 Spring Bean으로 정의하고 동적으로 해결할 수도 있습니다:

```java
@Configuration(proxyBeanMethods = false)
class WeatherTools {

    @Bean
    @Description("Get the weather in location")
    Function<WeatherRequest, WeatherResponse> currentWeather() {
        return new WeatherService();
    }
}

## Tool Calling 통합 구현

이제 Spring AI에서 Tool Calling을 사용하는 방법을 단계별로 살펴보겠습니다.

### 1. ChatClient와 도구 사용하기

#### 런타임 도구 추가

가장 간단한 방법은 ChatClient 호출 시 도구를 직접 전달하는 것입니다:

```java
ChatModel chatModel = ...

// @Tool 어노테이션이 있는 클래스 인스턴스 전달
String response = ChatClient.create(chatModel)
    .prompt("서울의 현재 날씨는 어때요?")
    .tools(new WeatherTools())
    .call()
    .content();

// 또는 개별 ToolCallback 전달
ToolCallback weatherTool = FunctionToolCallback
    .builder("getCurrentWeather", new WeatherService())
    .description("Get the weather in location")
    .inputType(WeatherRequest.class)
    .build();

String response = ChatClient.create(chatModel)
    .prompt("서울의 현재 날씨는 어때요?")
    .tools(weatherTool)
    .call()
    .content();
```

#### 기본 도구 설정

자주 사용하는 도구는 ChatClient.Builder에 기본값으로 설정할 수 있습니다:

```java
ChatModel chatModel = ...
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultTools(new WeatherTools(), new DateTimeTools())
    .build();

// 이제 모든 요청에 기본 도구가 포함됩니다
String response = chatClient.prompt()
    .user("내일 날씨는 어떤가요?")
    .call()
    .content();
```

### 2. ChatModel과 도구 사용하기

ChatModel을 직접 사용할 때는 `ToolCallingChatOptions`를 통해 도구를 전달합니다:

```java
ChatModel chatModel = ...
ToolCallback[] weatherTools = ToolCallbacks.from(new WeatherTools());

ChatOptions chatOptions = ToolCallingChatOptions.builder()
    .toolCallbacks(weatherTools)
    .build();

Prompt prompt = new Prompt("서울의 날씨는 어때요?", chatOptions);
ChatResponse response = chatModel.call(prompt);
```

### 3. 동적 도구 해결 (Spring Bean 사용)

Spring Bean으로 정의된 도구는 이름으로 동적으로 해결할 수 있습니다:

```java
@Configuration
class ToolConfiguration {
    
    @Bean
    @Description("Get current weather information")
    Function<WeatherRequest, WeatherResponse> weatherTool() {
        return new WeatherService();
    }
}

// 도구 이름으로 참조
String response = ChatClient.create(chatModel)
    .prompt("날씨 어때요?")
    .toolNames("weatherTool")  // Bean 이름 사용
    .call()
    .content();
```

### 4. Tool Context 사용하기

도구에 추가적인 컨텍스트 정보를 전달할 수 있습니다:

```java
class CustomerTools {
    
    @Tool(description = "Retrieve customer information")
    Customer getCustomerInfo(Long id, ToolContext toolContext) {
        String tenantId = (String) toolContext.getContext().get("tenantId");
        return customerRepository.findById(id, tenantId);
    }
}

// 도구 호출 시 컨텍스트 전달
String response = ChatClient.create(chatModel)
    .prompt("고객 ID 42의 정보를 알려주세요")
    .tools(new CustomerTools())
    .toolContext(Map.of("tenantId", "acme"))
    .call()
    .content();
```

### 5. Tool 실행 제어

Spring AI는 도구 실행을 프레임워크가 자동으로 처리하거나 사용자가 직접 제어할 수 있는 옵션을 제공합니다.

#### 프레임워크 제어 방식 (기본값)

기본적으로 Spring AI가 도구 호출을 자동으로 처리합니다:

```java
// 프레임워크가 자동으로 도구를 호출하고 결과를 모델에 전달
String response = ChatClient.create(chatModel)
    .prompt("서울의 날씨를 알려주고 내일을 위한 옷차림을 추천해줘")
    .tools(new WeatherTools())
    .call()
    .content();
```

#### 사용자 제어 방식

도구 실행을 직접 제어하려면 `internalToolExecutionEnabled`를 false로 설정합니다:

```java
ChatModel chatModel = ...
ToolCallingManager toolCallingManager = ToolCallingManager.builder().build();

ChatOptions chatOptions = ToolCallingChatOptions.builder()
    .toolCallbacks(new CustomerTools())
    .internalToolExecutionEnabled(false)  // 수동 제어
    .build();

Prompt prompt = new Prompt("고객 ID 42의 정보를 알려주세요", chatOptions);
ChatResponse chatResponse = chatModel.call(prompt);

// 도구 호출 요청 확인 및 실행
while (chatResponse.hasToolCalls()) {
    ToolExecutionResult toolExecutionResult = toolCallingManager.executeToolCalls(prompt, chatResponse);
    
    prompt = new Prompt(toolExecutionResult.conversationHistory(), chatOptions);
    chatResponse = chatModel.call(prompt);
}

System.out.println(chatResponse.getResult().getOutput().getText());
```

### 6. Return Direct 패턴

도구 결과를 모델에 전달하지 않고 직접 클라이언트에 반환하는 패턴입니다:

```java
class SearchTools {
    
    @Tool(description = "Search documents in knowledge base", returnDirect = true)
    List<Document> searchDocuments(String query) {
        // RAG 시나리오에서 검색 결과를 직접 반환
        return vectorStore.similaritySearch(query);
    }
}
```

`returnDirect = true`로 설정하면:
- 도구 실행 결과가 모델로 전달되지 않음
- 결과가 직접 클라이언트에 반환됨
- 추가적인 모델 처리 없이 빠른 응답 가능

## 실용적인 도구 구현 예제

### 날씨 정보 도구

```java
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;
import java.time.LocalDateTime;

@Component
public class WeatherTools {
    
    private final RestClient restClient;
    private final String apiKey = "${weather.api.key}";
    
    public WeatherTools(RestClient.Builder restClientBuilder) {
        this.restClient = restClientBuilder
            .baseUrl("https://api.weatherapi.com/v1")
            .build();
    }
    
    @Tool(description = "Get current weather information for a location")
    public WeatherInfo getCurrentWeather(
        @ToolParam(description = "City name") String city,
        @ToolParam(description = "Country code (optional)", required = false) String countryCode
    ) {
        String location = countryCode != null ? city + "," + countryCode : city;
        
        WeatherApiResponse response = restClient.get()
            .uri("/current.json?key={key}&q={location}", apiKey, location)
            .retrieve()
            .body(WeatherApiResponse.class);
        
        return new WeatherInfo(
            response.location().name(),
            response.location().country(),
            response.current().tempC(),
            response.current().tempF(),
            response.current().condition().text(),
            response.current().humidity(),
            response.current().windKph(),
            LocalDateTime.now()
        );
    }
    
    @Tool(description = "Get weather forecast for next days")
    public ForecastInfo getWeatherForecast(
        @ToolParam(description = "City name") String city,
        @ToolParam(description = "Number of days (1-10)") int days
    ) {
        ForecastApiResponse response = restClient.get()
            .uri("/forecast.json?key={key}&q={city}&days={days}", apiKey, city, days)
            .retrieve()
            .body(ForecastApiResponse.class);
        
        return new ForecastInfo(
            response.location().name(),
            response.forecast().forecastday().stream()
                .map(day -> new DayForecast(
                    day.date(),
                    day.day().maxtempC(),
                    day.day().mintempC(),
                    day.day().condition().text(),
                    day.day().chanceOfRain()
                ))
                .toList()
        );
    }
    
    // Response DTOs
    public record WeatherInfo(
        String city,
        String country,
        double tempC,
        double tempF,
        String condition,
        int humidity,
        double windKph,
        LocalDateTime timestamp
    ) {}
    
    public record ForecastInfo(
        String city,
        List<DayForecast> forecast
    ) {}
    
    public record DayForecast(
        String date,
        double maxTempC,
        double minTempC,
        String condition,
        int chanceOfRain
    ) {}
}
```

### 데이터베이스 조회 도구

```java
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.ai.tool.ToolContext;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDate;
import java.util.List;

@Component
public class CustomerTools {
    
    private final CustomerRepository customerRepository;
    private final OrderRepository orderRepository;
    
    public CustomerTools(CustomerRepository customerRepository, OrderRepository orderRepository) {
        this.customerRepository = customerRepository;
        this.orderRepository = orderRepository;
    }
    
    @Tool(description = "Search for customers by name or email")
    @Transactional(readOnly = true)
    public List<CustomerInfo> searchCustomers(
        @ToolParam(description = "Search query (name or email)") String query,
        @ToolParam(description = "Maximum number of results", required = false) Integer limit,
        ToolContext toolContext
    ) {
        // 컨텍스트에서 사용자 권한 확인
        String userRole = (String) toolContext.getContext().get("userRole");
        if (!"ADMIN".equals(userRole) && !"SUPPORT".equals(userRole)) {
            throw new SecurityException("Insufficient permissions");
        }
        
        int maxResults = limit != null ? limit : 10;
        return customerRepository.searchByNameOrEmail(query, maxResults).stream()
            .map(customer -> new CustomerInfo(
                customer.getId(),
                customer.getName(),
                customer.getEmail(),
                customer.getRegistrationDate(),
                customer.getTier()
            ))
            .toList();
    }
    
    @Tool(description = "Get customer order history")
    @Transactional(readOnly = true)
    public OrderHistory getCustomerOrderHistory(
        @ToolParam(description = "Customer ID") Long customerId,
        @ToolParam(description = "Include only orders after this date", required = false) LocalDate afterDate
    ) {
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new IllegalArgumentException("Customer not found"));
        
        List<Order> orders = afterDate != null 
            ? orderRepository.findByCustomerIdAndDateAfter(customerId, afterDate)
            : orderRepository.findByCustomerId(customerId);
        
        double totalSpent = orders.stream()
            .mapToDouble(Order::getTotalAmount)
            .sum();
        
        return new OrderHistory(
            customer.getName(),
            orders.size(),
            totalSpent,
            orders.stream()
                .map(order -> new OrderSummary(
                    order.getId(),
                    order.getOrderDate(),
                    order.getTotalAmount(),
                    order.getStatus()
                ))
                .toList()
        );
    }
    
    // DTOs
    public record CustomerInfo(
        Long id,
        String name,
        String email,
        LocalDate registrationDate,
        String tier
    ) {}
    
    public record OrderHistory(
        String customerName,
        int totalOrders,
        double totalSpent,
        List<OrderSummary> recentOrders
    ) {}
    
    public record OrderSummary(
        Long orderId,
        LocalDate orderDate,
        double amount,
        String status
    ) {}
}
```

### 작업 수행 도구 (액션)

```java
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.beans.factory.annotation.Autowired;
import java.time.LocalDateTime;
import java.util.UUID;

@Component
public class TaskActionTools {
    
    private final EmailService emailService;
    private final NotificationService notificationService;
    private final TaskRepository taskRepository;
    
    @Autowired
    public TaskActionTools(EmailService emailService, 
                          NotificationService notificationService,
                          TaskRepository taskRepository) {
        this.emailService = emailService;
        this.notificationService = notificationService;
        this.taskRepository = taskRepository;
    }
    
    @Tool(description = "Send an email to a recipient")
    @Transactional
    public EmailResult sendEmail(
        @ToolParam(description = "Recipient email address") String to,
        @ToolParam(description = "Email subject") String subject,
        @ToolParam(description = "Email body content") String body,
        @ToolParam(description = "CC recipients (comma-separated)", required = false) String cc
    ) {
        try {
            String emailId = emailService.send(
                EmailRequest.builder()
                    .to(to)
                    .subject(subject)
                    .body(body)
                    .cc(cc != null ? List.of(cc.split(",")) : List.of())
                    .sentAt(LocalDateTime.now())
                    .build()
            );
            
            return new EmailResult(true, emailId, "Email sent successfully");
        } catch (Exception e) {
            return new EmailResult(false, null, "Failed to send email: " + e.getMessage());
        }
    }
    
    @Tool(description = "Create a task reminder")
    @Transactional
    public TaskReminder createReminder(
        @ToolParam(description = "Task description") String description,
        @ToolParam(description = "Due date and time in ISO-8601 format") String dueDateTime,
        @ToolParam(description = "Priority level (HIGH, MEDIUM, LOW)") String priority,
        @ToolParam(description = "Assignee email", required = false) String assignee
    ) {
        Task task = new Task();
        task.setId(UUID.randomUUID().toString());
        task.setDescription(description);
        task.setDueDateTime(LocalDateTime.parse(dueDateTime));
        task.setPriority(Priority.valueOf(priority.toUpperCase()));
        task.setAssignee(assignee);
        task.setStatus("PENDING");
        task.setCreatedAt(LocalDateTime.now());
        
        Task savedTask = taskRepository.save(task);
        
        // 알림 설정
        if (assignee != null) {
            notificationService.scheduleNotification(
                assignee,
                "Task Reminder: " + description,
                LocalDateTime.parse(dueDateTime).minusHours(1)
            );
        }
        
        return new TaskReminder(
            savedTask.getId(),
            savedTask.getDescription(),
            savedTask.getDueDateTime(),
            savedTask.getPriority().name(),
            savedTask.getAssignee(),
            "Reminder created successfully"
        );
    }
    
    @Tool(description = "Update task status")
    @Transactional
    public TaskUpdateResult updateTaskStatus(
        @ToolParam(description = "Task ID") String taskId,
        @ToolParam(description = "New status (PENDING, IN_PROGRESS, COMPLETED, CANCELLED)") String newStatus,
        @ToolParam(description = "Notes or comments", required = false) String notes
    ) {
        Task task = taskRepository.findById(taskId)
            .orElseThrow(() -> new IllegalArgumentException("Task not found"));
        
        String oldStatus = task.getStatus();
        task.setStatus(newStatus);
        task.setUpdatedAt(LocalDateTime.now());
        
        if (notes != null) {
            task.addNote(notes);
        }
        
        taskRepository.save(task);
        
        return new TaskUpdateResult(
            taskId,
            oldStatus,
            newStatus,
            LocalDateTime.now(),
            "Task status updated successfully"
        );
    }
    
    // Result DTOs
    public record EmailResult(
        boolean success,
        String emailId,
        String message
    ) {}
    
    public record TaskReminder(
        String taskId,
        String description,
        LocalDateTime dueDateTime,
        String priority,
        String assignee,
        String message
    ) {}
    
    public record TaskUpdateResult(
        String taskId,
        String oldStatus,
        String newStatus,
        LocalDateTime updatedAt,
        String message
    ) {}
}
```

## 복잡한 워크플로우 구현

여러 도구를 조합하여 복잡한 비즈니스 로직을 구현하는 방법을 살펴보겠습니다.

### 여행 계획 도우미 시스템

```java
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Service;
import java.time.LocalDate;
import java.util.List;

@Service
public class TravelAssistantService {
    
    private final ChatClient chatClient;
    
    public TravelAssistantService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder
            .defaultTools(
                new WeatherTools(),
                new FlightSearchTools(),
                new HotelBookingTools(),
                new ActivityRecommendationTools()
            )
            .build();
    }
    
    public String planTrip(String userRequest) {
        return chatClient.prompt()
            .system("""
                You are a helpful travel assistant that helps users plan their trips.
                Use the available tools to:
                1. Check weather conditions for the destination
                2. Search for flights
                3. Find suitable hotels
                4. Recommend activities and attractions
                
                Always consider the user's budget and preferences.
                Provide comprehensive travel plans with specific recommendations.
                """)
            .user(userRequest)
            .call()
            .content();
    }
}

@Component
class FlightSearchTools {
    
    @Tool(description = "Search for flights between two cities")
    public List<FlightOption> searchFlights(
        @ToolParam(description = "Departure city") String from,
        @ToolParam(description = "Destination city") String to,
        @ToolParam(description = "Departure date (YYYY-MM-DD)") LocalDate departureDate,
        @ToolParam(description = "Return date for round trip", required = false) LocalDate returnDate,
        @ToolParam(description = "Number of passengers") int passengers,
        @ToolParam(description = "Class (ECONOMY, BUSINESS, FIRST)") String flightClass
    ) {
        // 실제 항공편 검색 로직
        return flightSearchService.search(
            FlightSearchCriteria.builder()
                .from(from)
                .to(to)
                .departureDate(departureDate)
                .returnDate(returnDate)
                .passengers(passengers)
                .flightClass(FlightClass.valueOf(flightClass))
                .build()
        );
    }
    
    public record FlightOption(
        String airline,
        String flightNumber,
        LocalDateTime departure,
        LocalDateTime arrival,
        double price,
        String currency,
        int availableSeats
    ) {}
}

@Component
class HotelBookingTools {
    
    @Tool(description = "Search for hotels in a city")
    public List<HotelOption> searchHotels(
        @ToolParam(description = "City name") String city,
        @ToolParam(description = "Check-in date") LocalDate checkIn,
        @ToolParam(description = "Check-out date") LocalDate checkOut,
        @ToolParam(description = "Number of guests") int guests,
        @ToolParam(description = "Maximum price per night") double maxPrice,
        @ToolParam(description = "Minimum star rating (1-5)", required = false) Integer minStarRating
    ) {
        return hotelService.searchHotels(
            HotelSearchRequest.builder()
                .city(city)
                .checkIn(checkIn)
                .checkOut(checkOut)
                .guests(guests)
                .maxPricePerNight(maxPrice)
                .minStarRating(minStarRating != null ? minStarRating : 3)
                .build()
        );
    }
    
    @Tool(description = "Get detailed information about a specific hotel")
    public HotelDetails getHotelDetails(
        @ToolParam(description = "Hotel ID") String hotelId
    ) {
        return hotelService.getHotelDetails(hotelId);
    }
    
    public record HotelOption(
        String hotelId,
        String name,
        int starRating,
        double pricePerNight,
        String location,
        List<String> amenities,
        double userRating
    ) {}
    
    public record HotelDetails(
        String hotelId,
        String name,
        String description,
        String address,
        List<String> amenities,
        List<RoomType> availableRooms,
        CancellationPolicy cancellationPolicy
    ) {}
}

@Component
class ActivityRecommendationTools {
    
    @Tool(description = "Get activity recommendations for a destination")
    public List<Activity> recommendActivities(
        @ToolParam(description = "City or location") String location,
        @ToolParam(description = "Activity categories (comma-separated)") String categories,
        @ToolParam(description = "Maximum budget per person", required = false) Double maxBudget,
        @ToolParam(description = "Duration in hours", required = false) Integer duration
    ) {
        List<String> categoryList = List.of(categories.split(","));
        
        return activityService.findActivities(
            ActivitySearchCriteria.builder()
                .location(location)
                .categories(categoryList)
                .maxBudget(maxBudget)
                .duration(duration)
                .build()
        );
    }
    
    public record Activity(
        String name,
        String description,
        String category,
        double price,
        int durationHours,
        double rating,
        String bookingUrl
    ) {}
}
```

## 오류 처리 및 예외 관리

Tool Calling에서 발생할 수 있는 오류를 효과적으로 처리하는 방법을 알아보겠습니다.

### 도구 실행 예외 처리

```java
import org.springframework.ai.tool.ToolExecutionException;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;

@Component
public class RobustWeatherTools {
    
    @Tool(description = "Get current weather with error handling")
    public WeatherResult getWeatherSafely(
        @ToolParam(description = "City name") String city
    ) {
        try {
            // 날씨 API 호출
            WeatherInfo weather = weatherApi.getCurrentWeather(city);
            return new WeatherResult(true, weather, null);
            
        } catch (ApiException e) {
            // API 오류를 ToolExecutionException으로 변환
            throw new ToolExecutionException(
                "Weather API error: " + e.getMessage(), e
            );
        } catch (Exception e) {
            // 기타 예외 처리
            return new WeatherResult(false, null, 
                "Unable to fetch weather for " + city);
        }
    }
    
    public record WeatherResult(
        boolean success,
        WeatherInfo data,
        String errorMessage
    ) {}
}
```

### 사용자 정의 예외 처리기

```java
import org.springframework.ai.tool.ToolExecutionExceptionProcessor;
import org.springframework.ai.tool.ToolExecutionException;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ToolConfiguration {
    
    @Bean
    public ToolExecutionExceptionProcessor customExceptionProcessor() {
        return exception -> {
            // 예외 유형에 따른 사용자 친화적 메시지 생성
            if (exception.getCause() instanceof ApiLimitExceededException) {
                return "I'm currently unable to access that information due to rate limits. Please try again later.";
            } else if (exception.getCause() instanceof UnauthorizedException) {
                return "I don't have permission to access that resource.";
            } else {
                // 기본 오류 메시지
                return "An error occurred while processing your request: " + exception.getMessage();
            }
        };
    }
}
```

### 프로퍼티 설정

```yaml
spring:
  ai:
    tools:
      throw-exception-on-error: false  # 오류를 모델에 전달 (기본값)
```

`throw-exception-on-error`가 false이면 오류 메시지가 모델에 전달되어 사용자 친화적인 응답을 생성할 수 있습니다.

## 도구 테스트 및 디버깅

### 단위 테스트 작성

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.mockito.Mock;
import org.springframework.ai.tool.ToolCallback;
import org.springframework.ai.tool.ToolCallbacks;
import org.springframework.boot.test.context.SpringBootTest;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.when;

@SpringBootTest
class WeatherToolsTest {
    
    @Mock
    private WeatherApiClient weatherApiClient;
    
    private WeatherTools weatherTools;
    
    @BeforeEach
    void setUp() {
        weatherTools = new WeatherTools(weatherApiClient);
    }
    
    @Test
    void testWeatherToolExecution() {
        // Given
        when(weatherApiClient.getWeather("Seoul"))
            .thenReturn(new WeatherData(25.0, "Sunny", 60));
        
        // When
        WeatherInfo result = weatherTools.getCurrentWeather("Seoul", "KR");
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.city()).isEqualTo("Seoul");
        assertThat(result.tempC()).isEqualTo(25.0);
        assertThat(result.condition()).isEqualTo("Sunny");
    }
    
    @Test
    void testToolCallbackGeneration() {
        // When
        ToolCallback[] callbacks = ToolCallbacks.from(weatherTools);
        
        // Then
        assertThat(callbacks).hasSize(2); // getCurrentWeather와 getWeatherForecast
        assertThat(callbacks[0].getToolDefinition().name())
            .isEqualTo("getCurrentWeather");
        assertThat(callbacks[0].getToolDefinition().description())
            .contains("current weather");
    }
}
```

### 통합 테스트 작성

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@TestPropertySource(properties = {
    "spring.ai.openai.api-key=${OPENAI_API_KEY}",
    "spring.ai.openai.chat.options.model=gpt-4o"
})
class ToolCallingIntegrationTest {
    
    @Autowired
    private ChatClient.Builder chatClientBuilder;
    
    @Test
    void testWeatherToolCalling() {
        // Given
        ChatClient chatClient = chatClientBuilder
            .defaultTools(new WeatherTools())
            .build();
        
        // When
        String response = chatClient.prompt()
            .user("What's the weather like in Tokyo?")
            .call()
            .content();
        
        // Then
        assertThat(response).isNotBlank();
        assertThat(response.toLowerCase())
            .containsAnyOf("tokyo", "weather", "temperature", "°c");
    }
    
    @Test
    void testMultipleToolCalls() {
        // Given
        ChatClient chatClient = chatClientBuilder
            .defaultTools(
                new WeatherTools(),
                new CustomerTools()
            )
            .build();
        
        // When
        String response = chatClient.prompt()
            .user("Check the weather in Seoul and find customer John Doe")
            .call()
            .content();
        
        // Then
        assertThat(response).contains("Seoul");
        assertThat(response).containsIgnoringCase("john doe");
    }
}
```

### 도구 호출 모니터링

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class ToolMonitoringAspect {
    
    private static final Logger logger = LoggerFactory.getLogger(ToolMonitoringAspect.class);
    private final MeterRegistry meterRegistry;
    
    public ToolMonitoringAspect(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @Around("@annotation(org.springframework.ai.tool.annotation.Tool)")
    public Object monitorToolExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        String toolName = joinPoint.getSignature().getName();
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            logger.debug("Tool called: {} with args: {}", 
                toolName, joinPoint.getArgs());
            
            Object result = joinPoint.proceed();
            
            meterRegistry.counter("tool.calls", "name", toolName, "status", "success")
                .increment();
            
            logger.debug("Tool {} returned: {}", toolName, result);
            
            return result;
            
        } catch (Exception e) {
            meterRegistry.counter("tool.calls", "name", toolName, "status", "error")
                .increment();
            
            logger.error("Tool {} failed: {}", toolName, e.getMessage(), e);
            throw e;
            
        } finally {
            sample.stop(Timer.builder("tool.execution.time")
                .tag("name", toolName)
                .register(meterRegistry));
        }
    }
}
```

### 로깅 설정

```yaml
logging:
  level:
    org.springframework.ai: DEBUG
    org.springframework.ai.tool: TRACE
    com.example.tools: DEBUG
```

## 보안 고려사항

Tool Calling을 사용할 때 반드시 고려해야 할 보안 사항을 알아보겠습니다.

### 입력 검증 및 권한 제어

```java
import jakarta.validation.constraints.*;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.ai.tool.ToolContext;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Component;

@Component
public class SecureCustomerTools {
    
    @Tool(description = "Get customer information securely")
    @PreAuthorize("hasAnyRole('ADMIN', 'SUPPORT')")
    public CustomerInfo getCustomerInfo(
        @ToolParam(description = "Customer ID (alphanumeric, 6-20 chars)") 
        @Pattern(regexp = "^[a-zA-Z0-9]{6,20}$") 
        String customerId,
        
        @ToolParam(description = "Fields to return", required = false)
        @Pattern(regexp = "^(name|email|phone|tier)(,(name|email|phone|tier))*$")
        String fields,
        
        ToolContext context
    ) {
        // 컨텍스트에서 현재 사용자 정보 확인
        String currentUser = (String) context.getContext().get("username");
        String userRole = (String) context.getContext().get("role");
        
        // 추가 권한 검증
        if ("SUPPORT".equals(userRole)) {
            // 지원팀은 본인이 담당하는 고객만 조회 가능
            if (!customerService.isAssignedToSupport(customerId, currentUser)) {
                throw new SecurityException("Not authorized to view this customer");
            }
        }
        
        // SQL Injection 방지를 위한 추가 검증
        if (customerId.contains(";") || customerId.contains("--")) {
            throw new IllegalArgumentException("Invalid customer ID format");
        }
        
        return customerService.getCustomerInfo(customerId, fields);
    }
    
    @Tool(description = "Update customer tier - requires admin approval")
    @PreAuthorize("hasRole('ADMIN')")
    @Transactional
    public TierUpdateResult updateCustomerTier(
        @ToolParam(description = "Customer ID") 
        @NotBlank 
        String customerId,
        
        @ToolParam(description = "New tier (BRONZE, SILVER, GOLD, PLATINUM)") 
        @Pattern(regexp = "^(BRONZE|SILVER|GOLD|PLATINUM)$")
        String newTier,
        
        @ToolParam(description = "Reason for change")
        @Size(min = 10, max = 500)
        String reason,
        
        ToolContext context
    ) {
        // 감사 로그 기록
        auditService.log(AuditEntry.builder()
            .action("UPDATE_CUSTOMER_TIER")
            .userId((String) context.getContext().get("username"))
            .targetId(customerId)
            .details(Map.of("newTier", newTier, "reason", reason))
            .timestamp(LocalDateTime.now())
            .build());
        
        try {
            Customer customer = customerService.updateTier(customerId, newTier);
            return new TierUpdateResult(true, 
                "Tier updated successfully", 
                customer.getPreviousTier(), 
                newTier);
        } catch (Exception e) {
            return new TierUpdateResult(false, 
                "Failed to update tier: " + e.getMessage(), 
                null, null);
        }
    }
}
```

### Rate Limiting 구현

```java
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Refill;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.ai.tool.ToolContext;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class RateLimitedTools {
    
    private final ConcurrentHashMap<String, Bucket> buckets = new ConcurrentHashMap<>();
    
    @Tool(description = "Send notification with rate limiting")
    public NotificationResult sendNotification(
        @ToolParam(description = "Recipient email") String email,
        @ToolParam(description = "Message content") String message,
        ToolContext context
    ) {
        String userId = (String) context.getContext().get("userId");
        
        // 사용자별 Rate Limit 버킷 생성 (분당 10개 요청)
        Bucket bucket = buckets.computeIfAbsent(userId, k -> 
            Bucket.builder()
                .addLimit(Bandwidth.classic(10, Refill.intervally(10, Duration.ofMinutes(1))))
                .build()
        );
        
        if (!bucket.tryConsume(1)) {
            return new NotificationResult(false, 
                "Rate limit exceeded. Please try again later.");
        }
        
        // 알림 전송 로직
        notificationService.send(email, message);
        return new NotificationResult(true, "Notification sent successfully");
    }
    
    public record NotificationResult(boolean success, String message) {}
}
```

### 데이터 암호화 및 마스킹

```java
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;
import javax.crypto.Cipher;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;

@Component
public class SecureDataTools {
    
    private final SecretKeySpec secretKey;
    
    public SecureDataTools(@Value("${app.encryption.key}") String key) {
        this.secretKey = new SecretKeySpec(key.getBytes(), "AES");
    }
    
    @Tool(description = "Store sensitive data securely")
    public DataStorageResult storeSensitiveData(
        @ToolParam(description = "Data key") String key,
        @ToolParam(description = "Sensitive data") String data
    ) {
        try {
            // 데이터 암호화
            Cipher cipher = Cipher.getInstance("AES");
            cipher.init(Cipher.ENCRYPT_MODE, secretKey);
            byte[] encryptedData = cipher.doFinal(data.getBytes());
            String encodedData = Base64.getEncoder().encodeToString(encryptedData);
            
            // 암호화된 데이터 저장
            dataRepository.save(new SecureData(key, encodedData));
            
            return new DataStorageResult(true, "Data stored securely", key);
        } catch (Exception e) {
            return new DataStorageResult(false, "Failed to store data", null);
        }
    }
    
    @Tool(description = "Retrieve customer data with PII masking")
    public MaskedCustomerData getCustomerDataMasked(
        @ToolParam(description = "Customer ID") String customerId
    ) {
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new IllegalArgumentException("Customer not found"));
        
        // PII 마스킹
        return new MaskedCustomerData(
            customer.getId(),
            maskEmail(customer.getEmail()),
            maskPhone(customer.getPhone()),
            maskCreditCard(customer.getCreditCard()),
            customer.getTier()
        );
    }
    
    private String maskEmail(String email) {
        if (email == null) return null;
        int atIndex = email.indexOf('@');
        if (atIndex > 2) {
            return email.substring(0, 2) + "***" + email.substring(atIndex);
        }
        return "***" + email.substring(atIndex);
    }
    
    private String maskPhone(String phone) {
        if (phone == null || phone.length() < 4) return "***";
        return "***-***-" + phone.substring(phone.length() - 4);
    }
    
    private String maskCreditCard(String card) {
        if (card == null || card.length() < 4) return "****";
        return "**** **** **** " + card.substring(card.length() - 4);
    }
}

## 실제 사용 사례: 지능형 고객 지원 시스템

Tool Calling을 활용한 종합적인 고객 지원 시스템을 구현해보겠습니다.

### 고객 지원 봇 아키텍처

```java
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.InMemoryChatMemory;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.ai.tool.ToolContext;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.*;

@Service
public class CustomerSupportService {
    
    private final ChatClient chatClient;
    private final InMemoryChatMemory chatMemory = new InMemoryChatMemory();
    
    public CustomerSupportService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder
            .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
            .defaultTools(
                new OrderManagementTools(),
                new ProductInformationTools(),
                new CustomerServiceTools(),
                new TicketingTools()
            )
            .build();
    }
    
    public SupportResponse handleCustomerQuery(
        String customerId, 
        String sessionId,
        String query
    ) {
        // 고객 컨텍스트 준비
        Map<String, Object> context = Map.of(
            "customerId", customerId,
            "sessionId", sessionId,
            "timestamp", LocalDateTime.now()
        );
        
        String response = chatClient.prompt()
            .system("""
                You are Emma, a friendly and professional customer support specialist.
                
                Your responsibilities:
                1. Help customers with order inquiries, product information, and general support
                2. Process returns and exchanges following company policy
                3. Create support tickets for complex issues
                4. Provide personalized assistance based on customer history
                
                Guidelines:
                - Always verify customer identity before sharing order details
                - Be empathetic and solution-oriented
                - Proactively suggest alternatives when products are out of stock
                - Follow up on previous issues if relevant
                
                Current date: {current_date}
                Customer ID: {customerId}
                """)
            .user(query)
            .toolContext(context)
            .advisors(a -> a.param(MessageChatMemoryAdvisor.CONVERSATION_ID, sessionId))
            .call()
            .content();
        
        return new SupportResponse(response, sessionId);
    }
}

@Component
class OrderManagementTools {
    
    @Tool(description = "Check order status and details")
    public OrderDetails getOrderDetails(
        @ToolParam(description = "Order ID") String orderId,
        ToolContext context
    ) {
        String customerId = (String) context.getContext().get("customerId");
        
        // 주문이 해당 고객의 것인지 확인
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new IllegalArgumentException("Order not found"));
        
        if (!order.getCustomerId().equals(customerId)) {
            throw new SecurityException("Order does not belong to customer");
        }
        
        return OrderDetails.builder()
            .orderId(order.getId())
            .status(order.getStatus())
            .orderDate(order.getCreatedAt())
            .items(order.getItems().stream()
                .map(item -> new OrderItem(
                    item.getProductName(),
                    item.getQuantity(),
                    item.getPrice()
                ))
                .toList())
            .trackingNumber(order.getTrackingNumber())
            .estimatedDelivery(order.getEstimatedDelivery())
            .build();
    }
    
    @Tool(description = "Process return request for an order")
    @Transactional
    public ReturnResult processReturn(
        @ToolParam(description = "Order ID") String orderId,
        @ToolParam(description = "Reason for return") String reason,
        @ToolParam(description = "Item IDs to return") List<String> itemIds,
        ToolContext context
    ) {
        String customerId = (String) context.getContext().get("customerId");
        
        // 반품 정책 확인
        Order order = orderRepository.findByIdAndCustomerId(orderId, customerId)
            .orElseThrow(() -> new IllegalArgumentException("Order not found"));
        
        if (!returnPolicyService.isEligibleForReturn(order)) {
            return new ReturnResult(
                false,
                "Order is not eligible for return. Return period has expired.",
                null
            );
        }
        
        // 반품 처리
        Return returnRequest = returnService.createReturn(
            order,
            itemIds,
            reason,
            customerId
        );
        
        // 반품 라벨 생성
        String returnLabel = shippingService.generateReturnLabel(returnRequest);
        
        return new ReturnResult(
            true,
            "Return request created successfully",
            new ReturnInfo(
                returnRequest.getId(),
                returnLabel,
                "5-7 business days",
                returnRequest.getRefundAmount()
            )
        );
    }
}

@Component
class ProductInformationTools {
    
    @Tool(description = "Search for products by criteria")
    public List<ProductSearchResult> searchProducts(
        @ToolParam(description = "Search query") String query,
        @ToolParam(description = "Category filter", required = false) String category,
        @ToolParam(description = "Price range (min-max)", required = false) String priceRange,
        @ToolParam(description = "Sort by (relevance, price, rating)", required = false) String sortBy
    ) {
        SearchCriteria criteria = SearchCriteria.builder()
            .query(query)
            .category(category)
            .priceRange(parsePriceRange(priceRange))
            .sortBy(sortBy != null ? sortBy : "relevance")
            .build();
        
        return productSearchService.search(criteria).stream()
            .map(product -> new ProductSearchResult(
                product.getId(),
                product.getName(),
                product.getDescription(),
                product.getPrice(),
                product.getRating(),
                product.isInStock(),
                product.getImageUrl()
            ))
            .limit(10)
            .toList();
    }
    
    @Tool(description = "Get detailed product information including availability")
    public ProductDetails getProductDetails(
        @ToolParam(description = "Product ID") String productId,
        @ToolParam(description = "Include reviews", required = false) Boolean includeReviews
    ) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new IllegalArgumentException("Product not found"));
        
        List<StoreInventory> inventory = inventoryService.checkAvailability(productId);
        
        ProductDetails.ProductDetailsBuilder builder = ProductDetails.builder()
            .productId(product.getId())
            .name(product.getName())
            .description(product.getDescription())
            .price(product.getPrice())
            .specifications(product.getSpecifications())
            .inventory(inventory)
            .alternativeProducts(findAlternatives(product));
        
        if (Boolean.TRUE.equals(includeReviews)) {
            builder.reviews(reviewService.getTopReviews(productId, 5));
        }
        
        return builder.build();
    }
}

@Component
class TicketingTools {
    
    @Tool(description = "Create support ticket for complex issues")
    public SupportTicket createSupportTicket(
        @ToolParam(description = "Issue category") String category,
        @ToolParam(description = "Detailed description") String description,
        @ToolParam(description = "Priority (LOW, MEDIUM, HIGH)") String priority,
        @ToolParam(description = "Related order ID", required = false) String orderId,
        ToolContext context
    ) {
        String customerId = (String) context.getContext().get("customerId");
        String sessionId = (String) context.getContext().get("sessionId");
        
        // 티켓 생성
        Ticket ticket = Ticket.builder()
            .customerId(customerId)
            .category(TicketCategory.valueOf(category))
            .description(description)
            .priority(Priority.valueOf(priority))
            .relatedOrderId(orderId)
            .sessionId(sessionId)
            .status(TicketStatus.OPEN)
            .createdAt(LocalDateTime.now())
            .build();
        
        Ticket savedTicket = ticketRepository.save(ticket);
        
        // 고객에게 이메일 알림
        notificationService.sendTicketCreatedEmail(
            customerId,
            savedTicket.getId(),
            savedTicket.getEstimatedResponseTime()
        );
        
        return new SupportTicket(
            savedTicket.getId(),
            savedTicket.getStatus().name(),
            savedTicket.getEstimatedResponseTime(),
            "A support specialist will contact you within " + 
                savedTicket.getEstimatedResponseTime()
        );
    }
    
    @Tool(description = "Check status of existing support ticket")
    public TicketStatus checkTicketStatus(
        @ToolParam(description = "Ticket ID") String ticketId,
        ToolContext context
    ) {
        String customerId = (String) context.getContext().get("customerId");
        
        Ticket ticket = ticketRepository.findByIdAndCustomerId(ticketId, customerId)
            .orElseThrow(() -> new IllegalArgumentException("Ticket not found"));
        
        List<TicketUpdate> updates = ticketUpdateRepository.findByTicketId(ticketId);
        
        return new TicketStatus(
            ticket.getId(),
            ticket.getStatus().name(),
            ticket.getAssignedAgent(),
            updates,
            ticket.getEstimatedResolutionTime()
        );
    }
}

## 결론

이 장에서는 Spring AI의 Function Calling 기능과 `@Tool` 어노테이션을 활용하여 AI 모델에 Java 메서드를 노출하는 방법을 살펴보았습니다. 이 기능을 통해 AI가 실시간 데이터에 접근하고, 복잡한 계산을 수행하고, 외부 서비스와 상호작용할 수 있게 되어 더욱 강력하고 유용한 AI 애플리케이션을 구현할 수 있습니다.

주요 내용 요약:
- Function Calling은 LLM이 텍스트 응답 대신 미리 정의된 함수를 호출할 수 있게 하는 기능입니다.
- Spring AI의 `@Tool` 어노테이션은 Java 메서드를 AI 모델의 도구로 쉽게 등록할 수 있게 해줍니다.
- `@ToolParam` 어노테이션은 매개변수 설명을 제공하여 AI가 올바른 입력값을 추출하도록 돕습니다.
- 다양한 데이터 타입과 검증 어노테이션을 활용하여 강력한 도구를 구현할 수 있습니다.
- 복잡한 워크플로우를 구현하기 위해 여러 도구를 조합할 수 있습니다.
- 프롬프트 엔지니어링과 도구 설명 최적화를 통해 Function Calling의 성능을 향상시킬 수 있습니다.
- 도구 사용 시 보안, 유효성 검사, 권한 제한 등의 고려사항이 중요합니다.

다음 장에서는 이미지와 음성을 처리하는 멀티모달 AI 모델을 Spring AI와 통합하는 방법에 대해 알아보겠습니다.