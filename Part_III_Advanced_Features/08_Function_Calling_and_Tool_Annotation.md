# Function Calling & `@Tool` 어노테이션

## 개요

함수 호출(Function Calling) 기능은 최신 LLM이 제공하는 가장 강력한 기능 중 하나로, AI가 텍스트 응답을 생성하는 것을 넘어 정의된 함수를 호출할 수 있게 해줍니다. Spring AI 1.0.0-M6 버전에서는 이 기능을 효과적으로 활용할 수 있는 `@Tool` 및 `@ToolParam` 어노테이션을 도입했습니다.

이 장에서는 Spring AI의 Function Calling 기능을 살펴보고, `@Tool` 어노테이션을 사용하여 Java 메서드를 AI 모델의 도구로 노출하는 방법을 배우게 됩니다. 이를 통해 AI가 외부 데이터를 조회하거나 애플리케이션의 특정 기능을 실행하는 복잡한 시스템을 구현할 수 있습니다.

## Function Calling 이해하기

### Function Calling의 개념

Function Calling은 LLM이 텍스트 응답을 생성하는 대신, 미리 정의된 함수를 호출하여 필요한 정보를 얻거나 작업을 수행할 수 있게 하는 기능입니다. 이 기능을 통해 LLM은:

1. 사용자의 의도를 파악하여 적절한 함수를 선택할 수 있습니다.
2. 함수 호출에 필요한 매개변수를 사용자 입력에서 추출할 수 있습니다.
3. 함수의 실행 결과를 기반으로 추가적인 응답을 생성할 수 있습니다.

이러한 기능은 LLM의 추론 능력을 실제 시스템 기능과 연결하여 더욱 유용한 AI 애플리케이션을 구현할 수 있게 해줍니다.

### Function Calling이 필요한 경우

Function Calling은 다음과 같은 상황에서 특히 유용합니다:

1. **실시간 데이터 접근**: 날씨, 주식 시세, 제품 재고 등 실시간 정보에 접근해야 할 때
2. **복잡한 계산**: 환율 변환, 세금 계산, 최적 경로 계산 등 복잡한 계산이 필요할 때
3. **외부 서비스 통합**: 이메일 발송, 일정 등록, 티켓 예약 등 외부 서비스와 연동해야 할 때
4. **데이터베이스 조작**: 사용자 정보 조회, 주문 내역 검색, 상품 정보 업데이트 등 데이터베이스 작업이 필요할 때
5. **트랜잭션 처리**: 결제 처리, 계정 이체, 포인트 적립 등 트랜잭션이 수반되는 작업을 수행할 때

### 주요 AI 모델 프로바이더의 Function Calling 지원

Spring AI는 다양한 AI 모델 프로바이더의 Function Calling 기능을 추상화하여 제공합니다:

| 프로바이더 | 지원 모델 | 특징 |
|------------|----------|------|
| OpenAI | GPT-3.5, GPT-4 | 최초로 Function Calling 개념 도입, 다중 함수 호출 지원 |
| Anthropic | Claude 3 | Anthropic에서는 Tool Use로 불리며, M6 버전부터 지원 |
| Amazon Bedrock | Claude, Titan | AWS를 통한 통합 지원 |
| Azure OpenAI | GPT-3.5, GPT-4 | Microsoft Azure 환경에서의 Function Calling |
| Google | Gemini | 멀티모달 Function Calling 지원 |

## Spring AI의 `@Tool` 어노테이션

Spring AI는 Java 메서드를 AI 모델에서 호출 가능한 함수로 변환하는 매우 직관적인 방법을 제공합니다. `@Tool` 어노테이션을 사용하면 메서드를 도구로 노출할 수 있으며, `@ToolParam` 어노테이션으로 매개변수에 대한 설명을 제공할 수 있습니다.

### `@Tool` 어노테이션 기본 사용법

```java
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.stereotype.Component;

@Component
public class WeatherService {
    
    @Tool("Get current weather information for a city")
    public WeatherInfo getCurrentWeather(
        @ToolParam("The name of the city") String city,
        @ToolParam("The country code (ISO 3166-1 alpha-2)") String countryCode
    ) {
        // 날씨 API 호출 및 정보 반환
        return weatherApi.getWeatherInfo(city, countryCode);
    }
}
```

위 예제에서:
- `@Tool` 어노테이션은 메서드를 AI 모델이 호출할 수 있는 도구로 정의합니다.
- 어노테이션 값은 도구의 설명으로, AI 모델이 도구의 용도를 이해하는 데 사용됩니다.
- `@ToolParam` 어노테이션은 각 매개변수의 용도를 설명합니다.

### 도구 매개변수 지원 타입

Spring AI의 `@Tool`은 다양한 데이터 타입을 지원합니다:

- 기본 타입: `int`, `long`, `double`, `boolean`, `String` 등
- 배열 및 컬렉션: `List<T>`, `Set<T>`, `T[]` 등
- 객체: POJO, Record 타입 (직렬화/역직렬화 가능한 객체)
- 열거형: `enum` 타입
- 옵셔널 타입: `Optional<T>`

### 타입 힌트와 제약 조건

Function Calling의 효과를 높이기 위해 매개변수에 제약 조건을 추가할 수 있습니다:

```java
import jakarta.validation.constraints.*;
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.stereotype.Component;

@Component
public class BookService {
    
    @Tool("Search for books in our catalog")
    public List<Book> searchBooks(
        @ToolParam("Search query") String query,
        @ToolParam("Maximum number of results") @Min(1) @Max(50) int limit,
        @ToolParam("Filter by genre") 
        @Pattern(regexp = "^(fiction|non-fiction|sci-fi|mystery|biography)$") 
        String genre
    ) {
        return bookRepository.findBooks(query, genre, limit);
    }
}
```

위 예제에서 Jakarta Validation 어노테이션을 사용하여:
- `limit` 매개변수는 1~50 사이의 값만 허용합니다.
- `genre` 매개변수는 지정된 장르 중 하나만 허용합니다.

### Record 및 복잡한 객체 사용하기

복잡한 매개변수를 지원하기 위해 Record나 POJO 클래스를 사용할 수 있습니다:

```java
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.stereotype.Component;

record FlightSearchCriteria(
    @ToolParam("Departure airport code") String from,
    @ToolParam("Destination airport code") String to,
    @ToolParam("Departure date (YYYY-MM-DD)") LocalDate departureDate,
    @ToolParam("Return date (YYYY-MM-DD)") LocalDate returnDate,
    @ToolParam("Number of passengers") int passengers
) {}

@Component
public class FlightService {
    
    @Tool("Search for available flights")
    public List<Flight> searchFlights(
        @ToolParam("Flight search criteria") FlightSearchCriteria criteria
    ) {
        return flightRepository.findFlights(
            criteria.from(), 
            criteria.to(), 
            criteria.departureDate(), 
            criteria.returnDate(), 
            criteria.passengers()
        );
    }
}
```

이 예제에서는 항공편 검색에 필요한 여러 매개변수를 `FlightSearchCriteria` 레코드로 그룹화하여 사용하고 있습니다.

## Function Calling 통합 구현

이제 Spring AI에서 Function Calling을 사용하는 방법을 단계별로 살펴보겠습니다.

### 1. 도구 등록 및 자동 구성

Spring Boot를 사용할 경우, `@Tool` 어노테이션이 적용된 빈은 자동으로 AI 서비스에 등록됩니다. 이를 위한 설정은 다음과 같습니다:

```java
import org.springframework.ai.autoconfigure.tool.ToolsAutoConfiguration;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Import;

@SpringBootApplication
@Import(ToolsAutoConfiguration.class)
public class FunctionCallingDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(FunctionCallingDemoApplication.class, args);
    }
}
```

Spring Boot starter를 사용하면 `ToolsAutoConfiguration`이 자동으로 가져와지므로, 별도의 `@Import` 설정 없이도 도구가 자동으로 등록됩니다.

### 2. 도구 스캔 및 구성

특정 패키지의 도구만 스캔하도록 구성할 수도 있습니다:

```java
import org.springframework.ai.tool.ToolRegisterService;
import org.springframework.ai.tool.ToolRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ToolConfiguration {
    
    @Bean
    public ToolRegistry toolRegistry(ToolRegisterService toolRegisterService) {
        return toolRegisterService.scan("com.example.myapp.tools");
    }
}
```

### 3. ChatClient를 통한 Function Calling 사용

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.messages.Message;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ChatController {
    
    private final ChatClient chatClient;
    
    @Autowired
    public ChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    @PostMapping("/chat")
    public String chat(@RequestBody String userInput) {
        UserMessage userMessage = new UserMessage(userInput);
        Prompt prompt = new Prompt(userMessage);
        
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
}
```

이제 사용자가 "서울의 현재 날씨는 어때요?"와 같은 질문을 하면, LLM은 이를 분석하여 `getCurrentWeather` 메서드를 호출하고 그 결과를 바탕으로 응답을 생성합니다.

### 4. 도구 호출 결과 처리

도구 호출 결과를 활용하여 더 풍부한 응답을 생성하는 방법을 알아보겠습니다:

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.SystemPromptTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
public class EnhancedChatController {
    
    private final ChatClient chatClient;
    
    @Autowired
    public EnhancedChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    @PostMapping("/enhanced-chat")
    public String enhancedChat(@RequestBody String userInput) {
        // 시스템 프롬프트 정의
        String systemPromptText = """
            You are a helpful assistant with access to various tools.
            When asked about weather, stock prices, or flight information, use the appropriate tool to get real-time data.
            For weather information, provide temperature in both Celsius and Fahrenheit, and mention if there's any precipitation.
            For flight information, suggest the best options based on price and duration.
            Respond in a friendly, conversational tone and offer additional helpful information when appropriate.
            """;
        
        SystemPromptTemplate systemPrompt = new SystemPromptTemplate(systemPromptText);
        UserMessage userMessage = new UserMessage(userInput);
        
        Prompt prompt = new Prompt(systemPrompt.create(), userMessage);
        
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
}
```

## 실용적인 도구 구현 예제

### 날씨 정보 도구

```java
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

@Component
public class WeatherTool {
    
    private final RestTemplate restTemplate = new RestTemplate();
    private final String apiKey = "your-weather-api-key";
    
    @Tool("Get current weather information for a location")
    public WeatherInfo getCurrentWeather(
        @ToolParam("City name") String city,
        @ToolParam("Country code (optional)") String countryCode
    ) {
        String location = city;
        if (countryCode != null && !countryCode.isEmpty()) {
            location += "," + countryCode;
        }
        
        String url = String.format(
            "https://api.weatherapi.com/v1/current.json?key=%s&q=%s&aqi=no",
            apiKey, location
        );
        
        WeatherApiResponse response = restTemplate.getForObject(url, WeatherApiResponse.class);
        
        return new WeatherInfo(
            response.getCurrent().getTempC(),
            response.getCurrent().getTempF(),
            response.getCurrent().getCondition().getText(),
            response.getCurrent().getHumidity(),
            response.getCurrent().getWindKph(),
            response.getLocation().getName(),
            response.getLocation().getCountry()
        );
    }
    
    public record WeatherInfo(
        double tempC,
        double tempF,
        String condition,
        int humidity,
        double windKph,
        String city,
        String country
    ) {}
    
    // WeatherApiResponse 클래스는 API 응답을 파싱하기 위한 클래스입니다.
    // 실제 구현에서는 API에 맞게 적절히 정의해야 합니다.
}
```

### 주식 시세 조회 도구

```java
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.List;

@Component
public class StockTool {
    
    private final RestTemplate restTemplate = new RestTemplate();
    private final String apiKey = "your-stock-api-key";
    
    @Tool("Get current stock price information")
    public StockInfo getStockPrice(
        @ToolParam("Stock symbol or ticker") String symbol
    ) {
        String url = String.format(
            "https://finnhub.io/api/v1/quote?symbol=%s&token=%s",
            symbol.toUpperCase(), apiKey
        );
        
        StockApiResponse response = restTemplate.getForObject(url, StockApiResponse.class);
        
        return new StockInfo(
            symbol.toUpperCase(),
            response.getC(), // 현재가
            response.getH(), // 최고가
            response.getL(), // 최저가
            response.getO(), // 시가
            response.getPc(), // 전일 종가
            LocalDate.now().format(DateTimeFormatter.ISO_DATE)
        );
    }
    
    @Tool("Get historical stock prices")
    public List<HistoricalPrice> getHistoricalPrices(
        @ToolParam("Stock symbol or ticker") String symbol,
        @ToolParam("Start date (YYYY-MM-DD)") String startDate,
        @ToolParam("End date (YYYY-MM-DD)") String endDate
    ) {
        // 실제 API 호출 및 데이터 처리 로직
        // ...
        
        return List.of(/* 과거 주가 데이터 */);
    }
    
    public record StockInfo(
        String symbol,
        double currentPrice,
        double highPrice,
        double lowPrice,
        double openPrice,
        double previousClosePrice,
        String date
    ) {}
    
    public record HistoricalPrice(
        String date,
        double openPrice,
        double highPrice,
        double lowPrice,
        double closePrice,
        long volume
    ) {}
    
    // StockApiResponse 클래스는 API 응답을 파싱하기 위한 클래스입니다.
}
```

### 환율 변환 도구

```java
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.Map;

@Component
public class CurrencyTool {
    
    private final RestTemplate restTemplate = new RestTemplate();
    private final String apiKey = "your-currency-api-key";
    
    @Tool("Convert amount from one currency to another")
    public CurrencyConversion convertCurrency(
        @ToolParam("Amount to convert") double amount,
        @ToolParam("Source currency code (e.g. USD)") String fromCurrency,
        @ToolParam("Target currency code (e.g. EUR)") String toCurrency
    ) {
        String url = String.format(
            "https://api.exchangerate.host/convert?from=%s&to=%s&amount=%f&access_key=%s",
            fromCurrency.toUpperCase(), toCurrency.toUpperCase(), amount, apiKey
        );
        
        ConversionResponse response = restTemplate.getForObject(url, ConversionResponse.class);
        
        return new CurrencyConversion(
            amount,
            fromCurrency.toUpperCase(),
            toCurrency.toUpperCase(),
            response.getResult(),
            response.getInfo().getRate(),
            LocalDate.now().format(DateTimeFormatter.ISO_DATE)
        );
    }
    
    @Tool("Get list of available currencies")
    public Map<String, String> getAvailableCurrencies() {
        String url = "https://api.exchangerate.host/symbols?access_key=" + apiKey;
        
        SymbolsResponse response = restTemplate.getForObject(url, SymbolsResponse.class);
        
        return response.getSymbols();
    }
    
    public record CurrencyConversion(
        double originalAmount,
        String fromCurrency,
        String toCurrency,
        double convertedAmount,
        double exchangeRate,
        String date
    ) {}
    
    // ConversionResponse 및 SymbolsResponse 클래스는 API 응답을 파싱하기 위한 클래스입니다.
}
```

## 복잡한 워크플로우 구현

이제 여러 도구를 조합하여 복잡한 워크플로우를 구현해 보겠습니다. 여행 계획 도우미 시스템을 예로 들어 설명하겠습니다.

### 여행 계획 도우미 시스템

```java
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.stereotype.Component;

import java.time.LocalDate;
import java.util.List;

@Component
public class TravelPlannerTool {
    
    private final WeatherTool weatherTool;
    private final CurrencyTool currencyTool;
    private final FlightTool flightTool;
    private final HotelTool hotelTool;
    
    public TravelPlannerTool(
            WeatherTool weatherTool,
            CurrencyTool currencyTool,
            FlightTool flightTool,
            HotelTool hotelTool) {
        this.weatherTool = weatherTool;
        this.currencyTool = currencyTool;
        this.flightTool = flightTool;
        this.hotelTool = hotelTool;
    }
    
    @Tool("Create a comprehensive travel plan for a destination")
    public TravelPlan createTravelPlan(
        @ToolParam("Destination city") String city,
        @ToolParam("Destination country") String country,
        @ToolParam("Departure date (YYYY-MM-DD)") String departureDateStr,
        @ToolParam("Return date (YYYY-MM-DD)") String returnDateStr,
        @ToolParam("Number of travelers") int travelers,
        @ToolParam("Budget per person in USD") double budgetPerPerson
    ) {
        LocalDate departureDate = LocalDate.parse(departureDateStr);
        LocalDate returnDate = LocalDate.parse(returnDateStr);
        
        // 1. 날씨 정보 조회
        WeatherTool.WeatherInfo weatherInfo = weatherTool.getCurrentWeather(city, country);
        
        // 2. 통화 정보 조회
        String localCurrency = getCurrencyForCountry(country);
        CurrencyTool.CurrencyConversion currencyInfo = 
            currencyTool.convertCurrency(budgetPerPerson, "USD", localCurrency);
        
        // 3. 항공편 검색
        List<FlightTool.Flight> flights = flightTool.searchFlights(
            "NYC", // 출발지 (실제로는 사용자 정보에서 가져올 수 있습니다)
            getNearestAirport(city, country),
            departureDateStr,
            returnDateStr,
            travelers
        );
        
        // 4. 호텔 검색
        List<HotelTool.Hotel> hotels = hotelTool.searchHotels(
            city,
            country,
            departureDateStr,
            returnDateStr,
            travelers,
            budgetPerPerson * 0.4 // 예산의 40%를 숙박에 할당
        );
        
        // 5. 여행 계획 생성
        return new TravelPlan(
            city,
            country,
            departureDate,
            returnDate,
            travelers,
            budgetPerPerson,
            weatherInfo,
            flights.isEmpty() ? null : flights.get(0),
            hotels.isEmpty() ? null : hotels.get(0),
            currencyInfo.exchangeRate(),
            generateItineraryRecommendations(city, country, departureDate, returnDate)
        );
    }
    
    // 국가별 통화 코드 매핑 (실제로는 더 완전한 매핑이 필요합니다)
    private String getCurrencyForCountry(String country) {
        return switch (country.toLowerCase()) {
            case "usa", "united states" -> "USD";
            case "japan" -> "JPY";
            case "south korea" -> "KRW";
            case "france", "germany", "italy", "spain" -> "EUR";
            case "uk", "united kingdom" -> "GBP";
            default -> "USD";
        };
    }
    
    // 도시에서 가장 가까운 공항 코드 찾기 (실제로는 공항 데이터베이스 조회가 필요합니다)
    private String getNearestAirport(String city, String country) {
        // 실제 구현에서는 공항 데이터베이스 API를 사용해 가장 가까운 공항을 찾습니다.
        return switch (city.toLowerCase()) {
            case "new york" -> "JFK";
            case "tokyo" -> "HND";
            case "seoul" -> "ICN";
            case "paris" -> "CDG";
            case "london" -> "LHR";
            default -> city.substring(0, 3).toUpperCase(); // 단순화된 예시
        };
    }
    
    // 추천 일정 생성 (실제로는 현지 관광지, 레스토랑 등의 데이터가 필요합니다)
    private List<ItineraryItem> generateItineraryRecommendations(
            String city, String country, LocalDate startDate, LocalDate endDate) {
        // 실제 구현에서는 목적지의 관광지, 음식점, 활동 등을 API에서 가져옵니다.
        // 이 예제에서는 간단히 더미 데이터를 반환합니다.
        return List.of(
            new ItineraryItem("Day 1", "Airport arrival and hotel check-in"),
            new ItineraryItem("Day 2", "Visit local attractions"),
            new ItineraryItem("Day 3", "Try local cuisine"),
            new ItineraryItem("Last Day", "Souvenir shopping and departure")
        );
    }
    
    // 모델 클래스들
    public record TravelPlan(
        String city,
        String country,
        LocalDate departureDate,
        LocalDate returnDate,
        int travelers,
        double budgetPerPerson,
        WeatherTool.WeatherInfo weatherInfo,
        FlightTool.Flight recommendedFlight,
        HotelTool.Hotel recommendedHotel,
        double exchangeRate,
        List<ItineraryItem> recommendedItinerary
    ) {}
    
    public record ItineraryItem(
        String day,
        String activity
    ) {}
}
```

이 예제에서는 날씨 정보, 통화 환율, 항공편, 호텔 정보를 조합하여 종합적인 여행 계획을 생성하는 도구를 구현했습니다. 이러한 복잡한 워크플로우는 여러 API를 조합하여 사용자에게 가치 있는 정보를 제공합니다.

## 프롬프트 엔지니어링과 도구 사용 지침

Function Calling의 성능을 극대화하기 위한 프롬프트 엔지니어링 기법을 알아보겠습니다.

### 시스템 프롬프트에 도구 사용 지침 포함하기

```java
import org.springframework.ai.chat.messages.SystemMessage;

String systemPromptText = """
    You are a travel assistant with access to several tools.
    
    TOOLS USAGE GUIDELINES:
    1. For weather information, ALWAYS use the getCurrentWeather tool.
    2. For currency conversion, ALWAYS use the convertCurrency tool.
    3. When asked about flights, ALWAYS use the searchFlights tool.
    4. For comprehensive travel planning, use the createTravelPlan tool.
    5. Do NOT make up information about weather, flights or exchange rates - ALWAYS use the tools.
    
    When providing information:
    - Give weather in both Celsius and Fahrenheit
    - Present flight prices in USD and local currency
    - Provide specific hotel addresses and amenities
    - Suggest activities based on current weather
    """;

SystemMessage systemMessage = new SystemMessage(systemPromptText);
```

이러한 시스템 프롬프트는 AI에게 도구 사용에 대한 명확한 지침을 제공하여 더욱 일관되고 정확한 응답을 생성하도록 돕습니다.

### 도구 설명 최적화

도구와 매개변수 설명은 AI가 적절한 도구를 선택하고 올바른 매개변수를 추출하는 데 중요한 역할을 합니다:

```java
@Tool("""
    Search for available flights between airports.
    Use this tool when the user asks about flight availability, prices, or schedules.
    The tool requires departure and arrival airport codes (e.g. JFK, ICN, LHR),
    travel dates, and passenger count.
    """)
public List<Flight> searchFlights(
    @ToolParam("Departure airport 3-letter IATA code (e.g. JFK for New York)") String from,
    @ToolParam("Arrival airport 3-letter IATA code (e.g. ICN for Seoul)") String to,
    @ToolParam("Departure date in YYYY-MM-DD format") String departureDate,
    @ToolParam("Return date in YYYY-MM-DD format (leave empty for one-way)") String returnDate,
    @ToolParam("Number of passengers (1-9)") int passengers
) {
    // 구현 생략
}
```

구체적이고 자세한 설명을 제공하면 AI가 더 정확히 도구를 사용할 수 있습니다.

## 도구 테스트 및 디버깅

Function Calling 기능을 테스트하고 디버깅하는 방법을 알아보겠습니다.

### 단위 테스트 작성

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.tool.ToolManager;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
public class WeatherToolTest {
    
    @Autowired
    private ToolManager toolManager;
    
    @Autowired
    private WeatherTool weatherTool;
    
    @Test
    void testWeatherToolRegistration() {
        var tools = toolManager.getTools();
        
        assertTrue(tools.stream()
            .anyMatch(tool -> tool.getName().equals("getCurrentWeather")),
            "Weather tool should be registered");
    }
    
    @Test
    void testWeatherToolExecution() {
        WeatherTool.WeatherInfo result = weatherTool.getCurrentWeather("Seoul", "KR");
        
        assertNotNull(result);
        assertEquals("Seoul", result.city());
        assertEquals("South Korea", result.country());
        assertTrue(result.tempC() >= -50 && result.tempC() <= 50, "Temperature should be in reasonable range");
    }
    
    @Test
    void testToolSchemaGeneration() {
        var toolSchema = toolManager.generateToolSpecification();
        
        assertNotNull(toolSchema);
        assertTrue(toolSchema.contains("getCurrentWeather"), "Schema should include weather tool");
        assertTrue(toolSchema.contains("city"), "Schema should include city parameter");
        assertTrue(toolSchema.contains("countryCode"), "Schema should include country parameter");
    }
}
```

### 통합 테스트 작성

```java
import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
public class FunctionCallingIntegrationTest {
    
    @Autowired
    private ChatClient chatClient;
    
    @Test
    void testWeatherQueryShouldUseFunctionCalling() {
        UserMessage userMessage = new UserMessage("What's the weather like in Tokyo today?");
        Prompt prompt = new Prompt(userMessage);
        
        var response = chatClient.call(prompt);
        var content = response.getResult().getOutput().getContent();
        
        assertNotNull(content);
        assertTrue(
            content.contains("Tokyo") && (content.contains("°C") || content.contains("degrees")),
            "Response should include weather information for Tokyo"
        );
        
        // Function calling 사용 여부 확인 (OpenAI의 경우)
        var metadata = response.getMetadata();
        assertTrue(
            metadata.containsKey("tool_calls") || 
            metadata.containsKey("function_calls") || 
            metadata.containsKey("tools"),
            "Response should include tool usage metadata"
        );
    }
}
```

### 도구 호출 로깅 구현

도구 호출을 로깅하여 디버깅을 용이하게 할 수 있습니다:

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Aspect
@Component
public class ToolLoggingAspect {
    
    private static final Logger logger = LoggerFactory.getLogger(ToolLoggingAspect.class);
    
    @Around("@annotation(org.springframework.ai.tool.Tool)")
    public Object logToolExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        String toolName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        
        logger.info("Tool invoked: {} with parameters: {}", toolName, Arrays.toString(args));
        
        long startTime = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();
            long executionTime = System.currentTimeMillis() - startTime;
            
            logger.info("Tool execution completed: {} in {}ms with result: {}", 
                toolName, executionTime, result);
            
            return result;
        } catch (Exception e) {
            logger.error("Tool execution failed: {} with error: {}", toolName, e.getMessage(), e);
            throw e;
        }
    }
}
```

## 보안 고려사항

Function Calling을 사용할 때 반드시 고려해야 할 보안 사항을 알아보겠습니다.

### 입력 유효성 검사

사용자 입력이 AI 모델을 통해 도구로 전달되므로, 강력한 입력 유효성 검사가 필수적입니다:

```java
import jakarta.validation.constraints.*;
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.stereotype.Component;

@Component
public class UserAccountTool {
    
    @Tool("Get user profile information")
    public UserProfile getUserProfile(
        @ToolParam("User ID") @Pattern(regexp = "^[a-zA-Z0-9]{6,20}$") String userId,
        @ToolParam("Information fields to return (comma-separated)") 
        @Pattern(regexp = "^(name|email|phone|address)(,(name|email|phone|address))*$") 
        String fields
    ) {
        // 추가 보안 검증
        validateUserAccess(userId);
        validateFieldAccess(fields.split(","));
        
        // 프로필 정보 조회 및 반환
        return userService.getUserProfile(userId, fields.split(","));
    }
    
    private void validateUserAccess(String userId) {
        // 현재 인증된 사용자가 해당 사용자 정보에 접근 권한이 있는지 확인
        if (!authorizationService.canAccessUserData(userId)) {
            throw new SecurityException("Unauthorized access to user data");
        }
    }
    
    private void validateFieldAccess(String[] fields) {
        // 요청된 각 필드에 대한 접근 권한 확인
        for (String field : fields) {
            if (!authorizationService.canAccessUserField(field)) {
                throw new SecurityException("Unauthorized access to field: " + field);
            }
        }
    }
}
```

### 도구 권한 제한

도구가 수행할 수 있는 작업의 범위를 제한하는 것이 중요합니다:

```java
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Component;

@Component
public class AdminTool {
    
    @Tool("Get system status information")
    @PreAuthorize("hasRole('ADMIN')")
    public SystemStatus getSystemStatus() {
        return systemMonitorService.getCurrentStatus();
    }
    
    @Tool("Delete user account")
    @PreAuthorize("hasRole('ADMIN')")
    public OperationResult deleteUserAccount(
        @ToolParam("User ID to delete") String userId,
        @ToolParam("Reason for deletion") String reason
    ) {
        // 계정 삭제 로직
        boolean success = userService.deleteUser(userId, reason);
        return new OperationResult(success, "User deletion " + (success ? "successful" : "failed"));
    }
    
    record SystemStatus(String status, int activeUsers, double cpuUsage, double memoryUsage) {}
    record OperationResult(boolean success, String message) {}
}
```

### 샌드박스 및 멱등성

특히 변형(mutation) 작업을 수행하는 도구의 경우, 샌드박스 환경에서 테스트하고 멱등성을 고려해야 합니다:

```java
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
public class OrderTool {
    
    @Tool("Place a new order")
    @Transactional
    public OrderResult placeOrder(
        @ToolParam("Product ID") String productId,
        @ToolParam("Quantity") int quantity,
        @ToolParam("Shipping address") String address,
        @ToolParam("Idempotency key") String idempotencyKey
    ) {
        // 멱등성 키를 사용하여 중복 주문 방지
        if (orderRepository.existsByIdempotencyKey(idempotencyKey)) {
            Order existingOrder = orderRepository.findByIdempotencyKey(idempotencyKey);
            return new OrderResult(existingOrder.getId(), "Order already exists", false);
        }
        
        // 주문 처리 로직
        try {
            Order order = orderService.createOrder(productId, quantity, address, idempotencyKey);
            return new OrderResult(order.getId(), "Order placed successfully", true);
        } catch (Exception e) {
            return new OrderResult(null, "Order failed: " + e.getMessage(), false);
        }
    }
    
    record OrderResult(String orderId, String message, boolean success) {}
}
```

## 실제 사용 사례: 고객 지원 봇

마지막으로, Function Calling과 `@Tool` 어노테이션을 활용한 실제 사용 사례를 살펴보겠습니다. 이 예제에서는 다양한 도구를 활용하는 고객 지원 봇을 구현합니다.

### 고객 지원 봇 구현

```java
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.tool.Tool;
import org.springframework.ai.tool.ToolParam;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@Service
public class CustomerSupportService {
    
    private final ChatClient chatClient;
    
    @Autowired
    public CustomerSupportService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    public String handleCustomerQuery(String customerId, String query) {
        // 사용자 정보 조회 및 대화 컨텍스트 설정
        CustomerInfo customerInfo = getCustomerInfo(customerId);
        
        // 시스템 프롬프트 설정
        String systemPromptText = """
            You are a helpful customer support assistant for our e-commerce store.
            You have access to tools that can check order status, process returns, search product information,
            check inventory, and create support tickets.
            
            Always be polite, friendly, and helpful. If you don't know the answer, use the appropriate tool to find it.
            
            Customer information:
            - Name: %s
            - Email: %s
            - Membership Level: %s
            - Account since: %s
            
            Current date and time: %s
            
            For order inquiries, always verify the order belongs to the customer before providing details.
            For returns, explain our return policy and process.
            For product questions, provide detailed information and check stock availability.
            For issues you cannot resolve, create a support ticket and provide the ticket number.
            """.formatted(
                customerInfo.name(),
                customerInfo.email(),
                customerInfo.membershipLevel(),
                customerInfo.accountCreationDate(),
                LocalDateTime.now()
            );
        
        SystemMessage systemMessage = new SystemMessage(systemPromptText);
        UserMessage userMessage = new UserMessage(query);
        
        Prompt prompt = new Prompt(systemMessage, userMessage);
        
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
    
    private CustomerInfo getCustomerInfo(String customerId) {
        // 실제 구현에서는 데이터베이스에서 고객 정보를 조회합니다.
        return new CustomerInfo(
            "John Doe",
            "john.doe@example.com",
            "Gold",
            "2019-05-15"
        );
    }
    
    // 고객 정보 레코드
    record CustomerInfo(String name, String email, String membershipLevel, String accountCreationDate) {}
    
    // 도구 클래스들
    
    @Tool("Check the status of a customer order")
    public OrderStatus checkOrderStatus(
        @ToolParam("Order ID") String orderId,
        @ToolParam("Customer email for verification") String customerEmail
    ) {
        // 주문 상태 조회 로직
        return orderService.getOrderStatus(orderId, customerEmail);
    }
    
    @Tool("Get detailed information about a product")
    public ProductInfo getProductInfo(
        @ToolParam("Product ID or name") String productIdentifier
    ) {
        // 제품 정보 조회 로직
        return productService.getProductInfo(productIdentifier);
    }
    
    @Tool("Check inventory availability for a product")
    public InventoryStatus checkInventory(
        @ToolParam("Product ID") String productId,
        @ToolParam("Store location (optional)") String location
    ) {
        // 재고 확인 로직
        return inventoryService.checkAvailability(productId, location);
    }
    
    @Tool("Initiate a return request for an order")
    public ReturnRequest initiateReturn(
        @ToolParam("Order ID") String orderId,
        @ToolParam("Product ID") String productId,
        @ToolParam("Return reason") String reason,
        @ToolParam("Customer email for verification") String customerEmail
    ) {
        // 반품 요청 처리 로직
        return returnService.createReturnRequest(orderId, productId, reason, customerEmail);
    }
    
    @Tool("Create a support ticket for issues that cannot be resolved immediately")
    public SupportTicket createSupportTicket(
        @ToolParam("Customer email") String customerEmail,
        @ToolParam("Issue category") String category,
        @ToolParam("Issue description") String description,
        @ToolParam("Priority (low, medium, high)") String priority
    ) {
        // 티켓 생성 로직
        String ticketId = "TKT-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        return new SupportTicket(
            ticketId,
            customerEmail,
            category,
            description,
            priority,
            "open",
            LocalDateTime.now().toString(),
            "24-48 hours"
        );
    }
    
    // 모델 클래스들
    
    record OrderStatus(
        String orderId,
        String status,
        String orderDate,
        String estimatedDelivery,
        String lastUpdate,
        List<OrderItem> items
    ) {}
    
    record OrderItem(
        String productId,
        String productName,
        int quantity,
        double price
    ) {}
    
    record ProductInfo(
        String productId,
        String name,
        String description,
        double price,
        String category,
        Map<String, String> specifications,
        List<String> images,
        double averageRating,
        int reviewCount
    ) {}
    
    record InventoryStatus(
        String productId,
        int availableQuantity,
        List<StoreAvailability> storeAvailability,
        boolean inStock,
        String restockDate
    ) {}
    
    record StoreAvailability(
        String storeName,
        String location,
        int quantity,
        boolean available
    ) {}
    
    record ReturnRequest(
        String returnId,
        String orderId,
        String productId,
        String status,
        String returnLabel,
        String refundStatus,
        String estimatedRefundDate
    ) {}
    
    record SupportTicket(
        String ticketId,
        String customerEmail,
        String category,
        String description,
        String priority,
        String status,
        String createdAt,
        String estimatedResponseTime
    ) {}
}
```

이 고객 지원 봇은 Function Calling을 활용하여 주문 상태 확인, 제품 정보 조회, 재고 확인, 반품 요청, 지원 티켓 생성 등 다양한 기능을 수행할 수 있습니다.

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