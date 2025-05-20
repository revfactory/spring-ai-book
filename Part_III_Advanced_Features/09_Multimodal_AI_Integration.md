# 멀티모달 AI — 이미지·음성 모델 통합

## 개요

최신 AI 모델은 텍스트뿐만 아니라 이미지, 오디오, 비디오와 같은 다양한 데이터 타입을 처리할 수 있는 멀티모달 능력을 갖추게 되었습니다. Spring AI는 이러한 멀티모달 AI 모델을 자바 애플리케이션에 쉽게 통합할 수 있는 기능을 제공합니다.

이 장에서는 Spring AI를 사용하여 이미지 생성 및 이해, 음성 인식 및 합성과 같은 멀티모달 기능을 구현하는 방법을 알아보겠습니다. 텍스트를 이미지로 변환하는 DALL-E와 같은 모델을 통합하고, GPT-4 Vision이나 Claude 3와 같은 모델을 사용하여 이미지를 이해하는 기능을 구현하는 방법을 살펴볼 것입니다.

## 멀티모달 AI 개념 이해하기

### 멀티모달 AI란?

멀티모달 AI는 텍스트, 이미지, 오디오, 비디오 등 다양한 형태(모달리티)의 데이터를 이해하고 처리할 수 있는 인공지능 시스템을 말합니다. 이러한 시스템은 여러 감각 채널을 통합하여 더 풍부하고 맥락에 맞는 이해와 응답을 생성할 수 있습니다.

멀티모달 AI의 주요 기능은 다음과 같습니다:

1. **크로스모달 이해**: 여러 모달리티 간의 관계를 이해하고 연관짓는 능력
2. **다양한 입력 처리**: 다양한 형식의 데이터를 입력으로 받아들일 수 있는 능력
3. **통합된 표현 학습**: 서로 다른 형식의 데이터를 공통된 표현 공간에서 처리하는 능력
4. **다양한 출력 생성**: 텍스트, 이미지, 오디오 등 다양한 형식의 출력을 생성하는 능력

### 주요 멀티모달 AI 모델 유형

Spring AI에서 통합할 수 있는 주요 멀티모달 AI 모델 유형은 다음과 같습니다:

1. **텍스트와 이미지 통합 모델**
   - 이미지 생성 모델 (Text-to-Image): DALL-E, Stable Diffusion, Midjourney
   - 이미지 이해 모델 (Image Understanding): GPT-4 Vision, Claude 3, Gemini

2. **오디오 관련 모델**
   - 음성 인식 모델 (Speech-to-Text): Whisper
   - 텍스트 음성 변환 모델 (Text-to-Speech): ElevenLabs, Amazon Polly

3. **멀티모달 임베딩 모델**
   - CLIP: 텍스트와 이미지의 공통 임베딩 생성
   - 통합 임베딩 모델: 다양한 형식의 데이터에 대한 통합 벡터 표현 생성

## 이미지 생성 모델 통합하기

Spring AI는 텍스트 설명을 기반으로 이미지를 생성하는 다양한 모델을 지원합니다. 여기서는 OpenAI의 DALL-E를 통합하는 방법을 알아보겠습니다.

### OpenAI DALL-E 통합 구현

#### 의존성 설정

먼저 필요한 의존성을 추가합니다:

```kotlin
// Gradle - build.gradle.kts
dependencies {
    implementation("org.springframework.ai:spring-ai-openai-spring-boot-starter:1.0.0-M6")
    // 추가 의존성 생략
}
```

```xml
<!-- Maven - pom.xml -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    <version>1.0.0-M6</version>
</dependency>
```

#### 환경 설정

`application.yml` 파일에 OpenAI API 키와 이미지 모델 설정을 추가합니다:

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      image:
        options:
          model: dall-e-3
          quality: standard
          style: natural
          size: 1024x1024
```

#### 이미지 생성 서비스 구현

OpenAI를 사용하여 이미지를 생성하는 서비스를 구현합니다:

```java
package com.example.multimodal.service;

import org.springframework.ai.image.Image;
import org.springframework.ai.image.ImageClient;
import org.springframework.ai.image.ImagePrompt;
import org.springframework.ai.image.ImageResponse;
import org.springframework.ai.openai.OpenAiImageOptions;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.UUID;

@Service
public class ImageGenerationService {

    private final ImageClient imageClient;

    @Autowired
    public ImageGenerationService(@Qualifier("openAiImageClient") ImageClient imageClient) {
        this.imageClient = imageClient;
    }

    public Image generateImage(String prompt) {
        ImagePrompt imagePrompt = new ImagePrompt(prompt);
        ImageResponse response = imageClient.call(imagePrompt);
        return response.getResult();
    }

    public Image generateImageWithOptions(String prompt, String size, String style, String quality) {
        OpenAiImageOptions options = OpenAiImageOptions.builder()
                .withSize(size)
                .withStyle(style)
                .withQuality(quality)
                .build();
        
        ImagePrompt imagePrompt = new ImagePrompt(prompt, options);
        ImageResponse response = imageClient.call(imagePrompt);
        return response.getResult();
    }

    public String saveImageToFile(Image image, String directory) throws IOException {
        // 디렉토리가 존재하지 않으면 생성
        Path dirPath = Paths.get(directory);
        if (!Files.exists(dirPath)) {
            Files.createDirectories(dirPath);
        }

        // 이미지 데이터 추출
        Resource imageResource = image.getImageAsResource();
        String filename = UUID.randomUUID().toString() + ".png";
        Path filePath = dirPath.resolve(filename);

        // 이미지 파일 저장
        try (var inputStream = imageResource.getInputStream()) {
            Files.copy(inputStream, filePath);
        }

        return filePath.toString();
    }
}
```

#### 이미지 생성 컨트롤러

이미지 생성 기능을 노출하는 REST API를 구현합니다:

```java
package com.example.multimodal.controller;

import com.example.multimodal.service.ImageGenerationService;
import org.springframework.ai.image.Image;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.FileSystemResource;
import org.springframework.core.io.Resource;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;

@RestController
@RequestMapping("/api/images")
public class ImageGenerationController {

    private final ImageGenerationService imageGenerationService;
    private static final String IMAGE_DIRECTORY = "generated-images";

    @Autowired
    public ImageGenerationController(ImageGenerationService imageGenerationService) {
        this.imageGenerationService = imageGenerationService;
    }

    @PostMapping("/generate")
    public ResponseEntity<GenerationResponse> generateImage(@RequestBody GenerationRequest request) throws IOException {
        Image generatedImage = imageGenerationService.generateImage(request.getPrompt());
        String imagePath = imageGenerationService.saveImageToFile(generatedImage, IMAGE_DIRECTORY);
        
        return ResponseEntity.ok(new GenerationResponse(imagePath, "/api/images/view/" + imagePath.substring(imagePath.lastIndexOf('/') + 1)));
    }

    @PostMapping("/generate-custom")
    public ResponseEntity<GenerationResponse> generateCustomImage(@RequestBody CustomGenerationRequest request) throws IOException {
        Image generatedImage = imageGenerationService.generateImageWithOptions(
                request.getPrompt(),
                request.getSize(),
                request.getStyle(),
                request.getQuality()
        );
        
        String imagePath = imageGenerationService.saveImageToFile(generatedImage, IMAGE_DIRECTORY);
        
        return ResponseEntity.ok(new GenerationResponse(imagePath, "/api/images/view/" + imagePath.substring(imagePath.lastIndexOf('/') + 1)));
    }

    @GetMapping(value = "/view/{filename:.+}", produces = MediaType.IMAGE_PNG_VALUE)
    public ResponseEntity<Resource> viewImage(@PathVariable String filename) {
        Resource file = new FileSystemResource(IMAGE_DIRECTORY + "/" + filename);
        
        if (file.exists() && file.isReadable()) {
            return ResponseEntity.ok().body(file);
        } else {
            return ResponseEntity.notFound().build();
        }
    }

    // Request 및 Response 클래스
    public static class GenerationRequest {
        private String prompt;

        // getter, setter
        public String getPrompt() {
            return prompt;
        }

        public void setPrompt(String prompt) {
            this.prompt = prompt;
        }
    }

    public static class CustomGenerationRequest {
        private String prompt;
        private String size = "1024x1024";
        private String style = "natural";
        private String quality = "standard";

        // getter, setter
        public String getPrompt() {
            return prompt;
        }

        public void setPrompt(String prompt) {
            this.prompt = prompt;
        }

        public String getSize() {
            return size;
        }

        public void setSize(String size) {
            this.size = size;
        }

        public String getStyle() {
            return style;
        }

        public void setStyle(String style) {
            this.style = style;
        }

        public String getQuality() {
            return quality;
        }

        public void setQuality(String quality) {
            this.quality = quality;
        }
    }

    public static class GenerationResponse {
        private String imagePath;
        private String imageUrl;

        public GenerationResponse(String imagePath, String imageUrl) {
            this.imagePath = imagePath;
            this.imageUrl = imageUrl;
        }

        public String getImagePath() {
            return imagePath;
        }

        public String getImageUrl() {
            return imageUrl;
        }
    }
}
```

### 다른 이미지 생성 모델 활용하기

Spring AI는 Stability AI의 Stable Diffusion과 같은 다른 이미지 생성 모델도 지원합니다. 다음은 Stability AI 통합 예시입니다:

```java
package com.example.multimodal.service;

import org.springframework.ai.image.ImageClient;
import org.springframework.ai.image.ImagePrompt;
import org.springframework.ai.stability.StabilityAiImageOptions;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class StableDiffusionService {

    private final ImageClient stabilityImageClient;

    public StableDiffusionService(@Qualifier("stabilityAiImageClient") ImageClient stabilityImageClient) {
        this.stabilityImageClient = stabilityImageClient;
    }

    public byte[] generateImage(String prompt, StableDiffusionModel model) {
        StabilityAiImageOptions options = StabilityAiImageOptions.builder()
                .withModel(model.getModelId())
                .withWidth(1024)
                .withHeight(1024)
                .withCfgScale(7.0)
                .withSamples(1)
                .withSteps(30)
                .build();

        ImagePrompt imagePrompt = new ImagePrompt(prompt, options);
        return stabilityImageClient.call(imagePrompt).getResult().getImageAsBytes();
    }

    public enum StableDiffusionModel {
        SDXL_1_0("stable-diffusion-xl-1024-v1-0"),
        SD_2_1("stable-diffusion-v2-1");

        private final String modelId;

        StableDiffusionModel(String modelId) {
            this.modelId = modelId;
        }

        public String getModelId() {
            return modelId;
        }
    }
}
```

## 이미지 이해 모델 통합하기

최신 AI 모델은 이미지를 이해하고 분석하는 능력을 갖추고 있습니다. Spring AI를 사용하여 GPT-4 Vision이나 Claude 3와 같은 이미지 이해 모델을 통합해 보겠습니다.

### OpenAI GPT-4 Vision 통합하기

#### 의존성 및 구성

앞서 설정한 OpenAI 의존성과 설정을 사용하면서 추가 설정이 필요합니다:

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4-vision-preview
          max-tokens: 1000
```

#### 이미지 이해 서비스 구현

```java
package com.example.multimodal.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.AssistantMessage;
import org.springframework.ai.chat.messages.Media;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.List;

@Service
public class ImageUnderstandingService {

    private final ChatClient chatClient;

    public ImageUnderstandingService(@Qualifier("openAiChatClient") ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String analyzeImage(MultipartFile imageFile, String question) throws IOException {
        // 이미지 미디어 객체 생성
        Media imageMedia = Media.of(
            imageFile.getInputStream().readAllBytes(),
            Media.Type.PNG, // 또는 적절한 이미지 유형 (JPEG, GIF 등)
            imageFile.getOriginalFilename()
        );

        // 사용자 메시지 생성 (이미지 포함)
        UserMessage userMessage = UserMessage.builder()
            .content(question)
            .media(List.of(imageMedia))
            .build();

        // 프롬프트 생성 및 AI 호출
        Prompt prompt = new Prompt(userMessage);
        ChatResponse response = chatClient.call(prompt);

        // 응답 추출
        return response.getResult().getOutput().getContent();
    }

    public String analyzeImageWithSystemPrompt(
            Resource imageResource, 
            String question, 
            String systemPrompt) throws IOException {
        
        // 이미지 미디어 객체 생성
        Media imageMedia = Media.of(
            imageResource.getInputStream().readAllBytes(),
            Media.Type.PNG,
            imageResource.getFilename()
        );

        // 메시지 목록 생성
        List<Message> messages = List.of(
            new org.springframework.ai.chat.messages.SystemMessage(systemPrompt),
            UserMessage.builder()
                .content(question)
                .media(List.of(imageMedia))
                .build()
        );

        // 프롬프트 생성 및 AI 호출
        Prompt prompt = new Prompt(messages);
        ChatResponse response = chatClient.call(prompt);

        // 응답 추출
        return response.getResult().getOutput().getContent();
    }
    
    public String chatWithImageContext(
            Resource imageResource,
            String initialQuestion,
            List<String> conversationHistory) throws IOException {
        
        // 이미지 미디어 객체 생성
        Media imageMedia = Media.of(
            imageResource.getInputStream().readAllBytes(),
            Media.Type.PNG,
            imageResource.getFilename()
        );

        // 대화 메시지 목록 구성
        List<Message> messages = List.of(
            new org.springframework.ai.chat.messages.SystemMessage(
                "You are a helpful assistant that can understand images. " +
                "Provide detailed analysis and answers based on image content."
            ),
            UserMessage.builder()
                .content(initialQuestion)
                .media(List.of(imageMedia))
                .build()
        );

        // 이전 대화 기록 추가
        for (int i = 0; i < conversationHistory.size(); i++) {
            if (i % 2 == 0) {
                messages.add(new UserMessage(conversationHistory.get(i)));
            } else {
                messages.add(new AssistantMessage(conversationHistory.get(i)));
            }
        }

        // 프롬프트 생성 및 AI 호출
        Prompt prompt = new Prompt(messages);
        ChatResponse response = chatClient.call(prompt);

        // 응답 추출
        return response.getResult().getOutput().getContent();
    }
}
```

#### 이미지 이해 API 컨트롤러

```java
package com.example.multimodal.controller;

import com.example.multimodal.service.ImageUnderstandingService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.FileSystemResource;
import org.springframework.core.io.Resource;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.List;

@RestController
@RequestMapping("/api/vision")
public class ImageUnderstandingController {

    private final ImageUnderstandingService imageUnderstandingService;

    @Autowired
    public ImageUnderstandingController(ImageUnderstandingService imageUnderstandingService) {
        this.imageUnderstandingService = imageUnderstandingService;
    }

    @PostMapping(value = "/analyze", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<AnalysisResponse> analyzeImage(
            @RequestParam("image") MultipartFile imageFile,
            @RequestParam("question") String question) throws IOException {
        
        String analysis = imageUnderstandingService.analyzeImage(imageFile, question);
        return ResponseEntity.ok(new AnalysisResponse(analysis));
    }

    @PostMapping("/analyze-with-system-prompt")
    public ResponseEntity<AnalysisResponse> analyzeImageWithSystemPrompt(
            @RequestParam("imagePath") String imagePath,
            @RequestParam("question") String question,
            @RequestParam("systemPrompt") String systemPrompt) throws IOException {
        
        Resource imageResource = new FileSystemResource(imagePath);
        String analysis = imageUnderstandingService.analyzeImageWithSystemPrompt(
                imageResource, question, systemPrompt);
        
        return ResponseEntity.ok(new AnalysisResponse(analysis));
    }

    @PostMapping("/chat-with-image")
    public ResponseEntity<AnalysisResponse> chatWithImage(
            @RequestParam("imagePath") String imagePath,
            @RequestParam("initialQuestion") String initialQuestion,
            @RequestParam("history") List<String> conversationHistory) throws IOException {
        
        Resource imageResource = new FileSystemResource(imagePath);
        String response = imageUnderstandingService.chatWithImageContext(
                imageResource, initialQuestion, conversationHistory);
        
        return ResponseEntity.ok(new AnalysisResponse(response));
    }

    public static class AnalysisResponse {
        private String analysis;

        public AnalysisResponse(String analysis) {
            this.analysis = analysis;
        }

        public String getAnalysis() {
            return analysis;
        }
    }
}
```

### Anthropic Claude 3 통합하기

Claude 3 Opus와 Sonnet은 이미지 이해 능력이 뛰어난 멀티모달 모델입니다. Spring AI에서 Claude 3를 통합하는 방법을 알아보겠습니다.

#### 의존성 및 구성

```kotlin
// Gradle - build.gradle.kts
dependencies {
    implementation("org.springframework.ai:spring-ai-anthropic-spring-boot-starter:1.0.0-M6")
    // 추가 의존성 생략
}
```

```yaml
# application.yml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-3-opus-20240229
          max-tokens: 1000
```

#### Claude 이미지 이해 서비스

```java
package com.example.multimodal.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Media;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.List;

@Service
public class ClaudeVisionService {

    private final ChatClient claudeChatClient;

    public ClaudeVisionService(@Qualifier("anthropicChatClient") ChatClient claudeChatClient) {
        this.claudeChatClient = claudeChatClient;
    }

    public String analyzeImageWithClaude(Resource imageResource, String question) throws IOException {
        // 이미지 미디어 객체 생성
        Media imageMedia = Media.of(
            imageResource.getInputStream().readAllBytes(),
            Media.Type.PNG,
            imageResource.getFilename()
        );

        // 시스템 프롬프트 설정
        SystemMessage systemMessage = new SystemMessage(
            "You are a helpful AI vision assistant that can analyze images. " +
            "Provide detailed and accurate descriptions of what you see in the image."
        );

        // 사용자 메시지 생성 (이미지 포함)
        UserMessage userMessage = UserMessage.builder()
            .content(question)
            .media(List.of(imageMedia))
            .build();

        // 프롬프트 생성 및 AI 호출
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        return claudeChatClient.call(prompt).getResult().getOutput().getContent();
    }

    public String compareImages(Resource image1, Resource image2, String question) throws IOException {
        // 이미지 미디어 객체 생성
        Media imageMedia1 = Media.of(
            image1.getInputStream().readAllBytes(),
            Media.Type.PNG,
            image1.getFilename()
        );
        
        Media imageMedia2 = Media.of(
            image2.getInputStream().readAllBytes(),
            Media.Type.PNG,
            image2.getFilename()
        );

        // 시스템 프롬프트 설정
        SystemMessage systemMessage = new SystemMessage(
            "You are a helpful AI vision assistant that can compare images. " +
            "Analyze both images and provide detailed comparisons highlighting similarities and differences."
        );

        // 사용자 메시지 생성 (두 이미지 포함)
        UserMessage userMessage = UserMessage.builder()
            .content("Here are two images. " + question)
            .media(List.of(imageMedia1, imageMedia2))
            .build();

        // 프롬프트 생성 및 AI 호출
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        return claudeChatClient.call(prompt).getResult().getOutput().getContent();
    }
}
```

## 오디오 모델 통합하기

Spring AI는 음성 인식(Speech-to-Text)과 텍스트 음성 변환(Text-to-Speech) 기능도 지원합니다. OpenAI Whisper를 사용한 음성 인식과 ElevenLabs를 사용한 텍스트 음성 변환을 통합해 보겠습니다.

### 음성 인식(Speech-to-Text) 구현

#### 의존성 및 구성

```kotlin
// Gradle - build.gradle.kts
dependencies {
    implementation("org.springframework.ai:spring-ai-openai-spring-boot-starter:1.0.0-M6")
    // 추가 의존성 생략
}
```

```yaml
# application.yml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      audio:
        options:
          model: whisper-1
          temperature: 0
          response-format: text
```

#### 음성 인식 서비스 구현

```java
package com.example.multimodal.service;

import org.springframework.ai.openai.OpenAiAudioTranscriptionOptions;
import org.springframework.ai.openai.OpenAiAudioTranslationOptions;
import org.springframework.ai.openai.audio.OpenAiAudioTranscription;
import org.springframework.ai.openai.audio.OpenAiAudioTranslationClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;

@Service
public class SpeechToTextService {

    private final OpenAiAudioTranscription transcriptionClient;
    private final OpenAiAudioTranslationClient translationClient;

    @Autowired
    public SpeechToTextService(
            OpenAiAudioTranscription transcriptionClient,
            OpenAiAudioTranslationClient translationClient) {
        this.transcriptionClient = transcriptionClient;
        this.translationClient = translationClient;
    }

    public String transcribeAudio(MultipartFile audioFile) throws IOException {
        // 기본 옵션으로 오디오 파일 변환
        return transcriptionClient.transcribe(audioFile.getBytes());
    }

    public String transcribeAudioWithOptions(
            MultipartFile audioFile,
            String language,
            double temperature,
            String prompt) throws IOException {
        
        // 커스텀 옵션으로 오디오 파일 변환
        OpenAiAudioTranscriptionOptions options = OpenAiAudioTranscriptionOptions.builder()
                .withLanguage(language)
                .withTemperature(temperature)
                .withPrompt(prompt)
                .withResponseFormat("text")
                .build();
        
        return transcriptionClient.transcribe(audioFile.getBytes(), options);
    }

    public String translateAudio(MultipartFile audioFile) throws IOException {
        // 오디오 파일을 영어로 번역
        return translationClient.translate(audioFile.getBytes());
    }

    public String translateAudioWithOptions(
            MultipartFile audioFile,
            double temperature,
            String prompt) throws IOException {
        
        // 커스텀 옵션으로 오디오 파일 번역
        OpenAiAudioTranslationOptions options = OpenAiAudioTranslationOptions.builder()
                .withTemperature(temperature)
                .withPrompt(prompt)
                .withResponseFormat("text")
                .build();
        
        return translationClient.translate(audioFile.getBytes(), options);
    }
}
```

#### 음성 인식 컨트롤러

```java
package com.example.multimodal.controller;

import com.example.multimodal.service.SpeechToTextService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;

@RestController
@RequestMapping("/api/speech")
public class SpeechToTextController {

    private final SpeechToTextService speechToTextService;

    @Autowired
    public SpeechToTextController(SpeechToTextService speechToTextService) {
        this.speechToTextService = speechToTextService;
    }

    @PostMapping(value = "/transcribe", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<TranscriptionResponse> transcribeAudio(
            @RequestParam("audio") MultipartFile audioFile) throws IOException {
        
        String transcription = speechToTextService.transcribeAudio(audioFile);
        return ResponseEntity.ok(new TranscriptionResponse(transcription));
    }

    @PostMapping(value = "/transcribe-custom", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<TranscriptionResponse> transcribeAudioWithOptions(
            @RequestParam("audio") MultipartFile audioFile,
            @RequestParam(value = "language", required = false, defaultValue = "en") String language,
            @RequestParam(value = "temperature", required = false, defaultValue = "0.0") double temperature,
            @RequestParam(value = "prompt", required = false, defaultValue = "") String prompt) throws IOException {
        
        String transcription = speechToTextService.transcribeAudioWithOptions(
                audioFile, language, temperature, prompt);
        
        return ResponseEntity.ok(new TranscriptionResponse(transcription));
    }

    @PostMapping(value = "/translate", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<TranscriptionResponse> translateAudio(
            @RequestParam("audio") MultipartFile audioFile) throws IOException {
        
        String translation = speechToTextService.translateAudio(audioFile);
        return ResponseEntity.ok(new TranscriptionResponse(translation));
    }

    public static class TranscriptionResponse {
        private String text;

        public TranscriptionResponse(String text) {
            this.text = text;
        }

        public String getText() {
            return text;
        }
    }
}
```

### 텍스트 음성 변환(Text-to-Speech) 구현

텍스트를 음성으로 변환하는 기능을 구현해 보겠습니다. ElevenLabs API를 사용하는 예제입니다.

#### ElevenLabs 커스텀 클라이언트 구현

Spring AI는 현재 ElevenLabs를 직접 지원하지 않으므로 커스텀 클라이언트를 구현합니다:

```java
package com.example.multimodal.client;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.util.HashMap;
import java.util.Map;

@Component
public class ElevenLabsClient {

    private final RestTemplate restTemplate;
    private final String apiKey;
    private final String apiBaseUrl = "https://api.elevenlabs.io/v1";

    public ElevenLabsClient(
            RestTemplate restTemplate,
            @Value("${elevenlabs.api-key}") String apiKey) {
        this.restTemplate = restTemplate;
        this.apiKey = apiKey;
    }

    public byte[] generateSpeech(
            String text, 
            String voiceId, 
            double stability, 
            double similarityBoost) {
        
        String url = apiBaseUrl + "/text-to-speech/" + voiceId;
        
        // 요청 헤더 설정
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("xi-api-key", apiKey);
        
        // 요청 바디 설정
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("text", text);
        
        Map<String, Object> voiceSettings = new HashMap<>();
        voiceSettings.put("stability", stability);
        voiceSettings.put("similarity_boost", similarityBoost);
        requestBody.put("voice_settings", voiceSettings);
        
        HttpEntity<Map<String, Object>> requestEntity = new HttpEntity<>(requestBody, headers);
        
        // API 호출
        ResponseEntity<byte[]> response = restTemplate.exchange(
                url, 
                HttpMethod.POST, 
                requestEntity, 
                byte[].class
        );
        
        return response.getBody();
    }

    public record Voice(String voiceId, String name, String[] labels) {}

    public Voice[] getAvailableVoices() {
        String url = apiBaseUrl + "/voices";
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("xi-api-key", apiKey);
        
        HttpEntity<Void> requestEntity = new HttpEntity<>(headers);
        
        ResponseEntity<Map> response = restTemplate.exchange(
                url, 
                HttpMethod.GET, 
                requestEntity, 
                Map.class
        );
        
        // API 응답에서 목소리 목록 추출 (실제 구현은 API 응답 구조에 맞게 조정 필요)
        return new Voice[0]; // 더미 반환
    }
}
```

#### 텍스트 음성 변환 서비스

```java
package com.example.multimodal.service;

import com.example.multimodal.client.ElevenLabsClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.UUID;

@Service
public class TextToSpeechService {

    private final ElevenLabsClient elevenLabsClient;
    private static final String AUDIO_DIRECTORY = "generated-audio";

    @Autowired
    public TextToSpeechService(ElevenLabsClient elevenLabsClient) {
        this.elevenLabsClient = elevenLabsClient;
    }

    public String generateSpeech(String text, String voiceId) throws IOException {
        // 기본 설정으로 음성 생성
        return generateSpeechWithOptions(text, voiceId, 0.5, 0.75);
    }

    public String generateSpeechWithOptions(
            String text,
            String voiceId,
            double stability,
            double similarityBoost) throws IOException {
        
        // ElevenLabs API를 사용하여 음성 생성
        byte[] audioData = elevenLabsClient.generateSpeech(text, voiceId, stability, similarityBoost);
        
        // 생성된 오디오 파일 저장
        String filename = UUID.randomUUID() + ".mp3";
        Path filePath = Paths.get(AUDIO_DIRECTORY, filename);
        
        try (FileOutputStream fos = new FileOutputStream(filePath.toFile())) {
            fos.write(audioData);
        }
        
        return filePath.toString();
    }

    public ElevenLabsClient.Voice[] getAvailableVoices() {
        return elevenLabsClient.getAvailableVoices();
    }
}
```

#### 텍스트 음성 변환 컨트롤러

```java
package com.example.multimodal.controller;

import com.example.multimodal.client.ElevenLabsClient;
import com.example.multimodal.service.TextToSpeechService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.FileSystemResource;
import org.springframework.core.io.Resource;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;

@RestController
@RequestMapping("/api/tts")
public class TextToSpeechController {

    private final TextToSpeechService textToSpeechService;

    @Autowired
    public TextToSpeechController(TextToSpeechService textToSpeechService) {
        this.textToSpeechService = textToSpeechService;
    }

    @PostMapping("/generate")
    public ResponseEntity<TtsResponse> generateSpeech(
            @RequestBody TtsRequest request) throws IOException {
        
        String audioPath = textToSpeechService.generateSpeech(
                request.getText(), request.getVoiceId());
        
        return ResponseEntity.ok(new TtsResponse(audioPath, "/api/tts/audio/" + audioPath.substring(audioPath.lastIndexOf('/') + 1)));
    }

    @PostMapping("/generate-custom")
    public ResponseEntity<TtsResponse> generateSpeechWithOptions(
            @RequestBody CustomTtsRequest request) throws IOException {
        
        String audioPath = textToSpeechService.generateSpeechWithOptions(
                request.getText(),
                request.getVoiceId(),
                request.getStability(),
                request.getSimilarityBoost()
        );
        
        return ResponseEntity.ok(new TtsResponse(audioPath, "/api/tts/audio/" + audioPath.substring(audioPath.lastIndexOf('/') + 1)));
    }

    @GetMapping("/voices")
    public ResponseEntity<ElevenLabsClient.Voice[]> getAvailableVoices() {
        return ResponseEntity.ok(textToSpeechService.getAvailableVoices());
    }

    @GetMapping(value = "/audio/{filename:.+}", produces = MediaType.APPLICATION_OCTET_STREAM_VALUE)
    public ResponseEntity<Resource> getAudioFile(@PathVariable String filename) {
        Resource audioFile = new FileSystemResource("generated-audio/" + filename);
        
        if (audioFile.exists() && audioFile.isReadable()) {
            return ResponseEntity.ok()
                    .header("Content-Disposition", "attachment; filename=\"" + filename + "\"")
                    .body(audioFile);
        } else {
            return ResponseEntity.notFound().build();
        }
    }

    // Request 및 Response 클래스
    public static class TtsRequest {
        private String text;
        private String voiceId;

        // getter, setter
        public String getText() {
            return text;
        }

        public void setText(String text) {
            this.text = text;
        }

        public String getVoiceId() {
            return voiceId;
        }

        public void setVoiceId(String voiceId) {
            this.voiceId = voiceId;
        }
    }

    public static class CustomTtsRequest {
        private String text;
        private String voiceId;
        private double stability = 0.5;
        private double similarityBoost = 0.75;

        // getter, setter
        public String getText() {
            return text;
        }

        public void setText(String text) {
            this.text = text;
        }

        public String getVoiceId() {
            return voiceId;
        }

        public void setVoiceId(String voiceId) {
            this.voiceId = voiceId;
        }

        public double getStability() {
            return stability;
        }

        public void setStability(double stability) {
            this.stability = stability;
        }

        public double getSimilarityBoost() {
            return similarityBoost;
        }

        public void setSimilarityBoost(double similarityBoost) {
            this.similarityBoost = similarityBoost;
        }
    }

    public static class TtsResponse {
        private String audioPath;
        private String audioUrl;

        public TtsResponse(String audioPath, String audioUrl) {
            this.audioPath = audioPath;
            this.audioUrl = audioUrl;
        }

        public String getAudioPath() {
            return audioPath;
        }

        public String getAudioUrl() {
            return audioUrl;
        }
    }
}
```

## 멀티모달 애플리케이션 사례 연구

이제 지금까지 구현한 멀티모달 기능을 종합하여 완전한 멀티모달 애플리케이션을 구현해 보겠습니다. 이미지 설명 및 음성 변환 기능을 갖춘 접근성 향상 도구를 예로 들어보겠습니다.

### 접근성 도우미 서비스 구현

```java
package com.example.multimodal.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.FileSystemResource;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.UUID;

@Service
public class AccessibilityService {

    private final ImageUnderstandingService imageUnderstandingService;
    private final TextToSpeechService textToSpeechService;
    private final SpeechToTextService speechToTextService;
    
    private static final String TEMP_DIRECTORY = "temp-files";

    @Autowired
    public AccessibilityService(
            ImageUnderstandingService imageUnderstandingService,
            TextToSpeechService textToSpeechService,
            SpeechToTextService speechToTextService) {
        this.imageUnderstandingService = imageUnderstandingService;
        this.textToSpeechService = textToSpeechService;
        this.speechToTextService = speechToTextService;
        
        // 임시 디렉토리 생성
        try {
            Files.createDirectories(Paths.get(TEMP_DIRECTORY));
        } catch (IOException e) {
            throw new RuntimeException("Failed to create temp directory", e);
        }
    }

    public AccessibilityResult processImage(MultipartFile imageFile) throws IOException {
        // 1. 임시 파일로 이미지 저장
        String tempFilePath = saveToTempFile(imageFile);
        Resource imageResource = new FileSystemResource(tempFilePath);
        
        // 2. 이미지 설명 생성
        String imageDescription = imageUnderstandingService.analyzeImageWithSystemPrompt(
                imageResource,
                "Describe this image in detail for a visually impaired person. " +
                "Include all important visual elements, spatial relationships, and any text visible in the image.",
                "You are an accessibility assistant that provides detailed image descriptions for visually impaired users. " +
                "Your descriptions should be clear, concise, and focus on the most important aspects of the image."
        );
        
        // 3. 음성으로 설명 변환
        String audioPath = textToSpeechService.generateSpeech(
                imageDescription,
                "21m00Tcm4TlvDq8ikWAM" // Rachel voice ID
        );
        
        // 4. 결과 반환
        return new AccessibilityResult(
                imageDescription,
                audioPath,
                "/api/tts/audio/" + audioPath.substring(audioPath.lastIndexOf('/') + 1)
        );
    }

    public String processVoiceCommand(MultipartFile audioFile) throws IOException {
        // 1. 음성을 텍스트로 변환
        String command = speechToTextService.transcribeAudio(audioFile);
        
        // 2. 명령 처리 (이 예제에서는 간단한 응답만 반환)
        String response = "I heard you say: " + command;
        
        if (command.toLowerCase().contains("describe")) {
            response += ". Please upload an image for me to describe.";
        } else if (command.toLowerCase().contains("help")) {
            response += ". I can describe images for you. Try uploading an image or saying 'describe this image'.";
        }
        
        return response;
    }

    private String saveToTempFile(MultipartFile file) throws IOException {
        String filename = UUID.randomUUID() + "-" + file.getOriginalFilename();
        Path filePath = Paths.get(TEMP_DIRECTORY, filename);
        
        try (var outputStream = Files.newOutputStream(filePath)) {
            outputStream.write(file.getBytes());
        }
        
        return filePath.toString();
    }

    public record AccessibilityResult(
            String imageDescription,
            String audioPath,
            String audioUrl
    ) {}
}
```

### 접근성 도우미 컨트롤러

```java
package com.example.multimodal.controller;

import com.example.multimodal.service.AccessibilityService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;

@RestController
@RequestMapping("/api/accessibility")
public class AccessibilityController {

    private final AccessibilityService accessibilityService;

    @Autowired
    public AccessibilityController(AccessibilityService accessibilityService) {
        this.accessibilityService = accessibilityService;
    }

    @PostMapping(value = "/describe-image", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<AccessibilityService.AccessibilityResult> describeImage(
            @RequestParam("image") MultipartFile imageFile) throws IOException {
        
        AccessibilityService.AccessibilityResult result = accessibilityService.processImage(imageFile);
        return ResponseEntity.ok(result);
    }

    @PostMapping(value = "/voice-command", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<VoiceCommandResponse> processVoiceCommand(
            @RequestParam("audio") MultipartFile audioFile) throws IOException {
        
        String response = accessibilityService.processVoiceCommand(audioFile);
        return ResponseEntity.ok(new VoiceCommandResponse(response));
    }

    public static class VoiceCommandResponse {
        private String response;

        public VoiceCommandResponse(String response) {
            this.response = response;
        }

        public String getResponse() {
            return response;
        }
    }
}
```

### 웹 인터페이스 구현

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Accessibility Assistant</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            line-height: 1.6;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            color: #333;
        }
        
        h1 {
            color: #2c3e50;
            text-align: center;
            margin-bottom: 30px;
        }
        
        .container {
            display: flex;
            flex-direction: column;
            gap: 20px;
        }
        
        .card {
            background: white;
            border-radius: 8px;
            padding: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
        
        .card h2 {
            margin-top: 0;
            color: #3498db;
        }
        
        .button {
            background-color: #3498db;
            color: white;
            border: none;
            padding: 10px 15px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 1rem;
        }
        
        .button:hover {
            background-color: #2980b9;
        }
        
        .result {
            margin-top: 20px;
            padding: 15px;
            background: #f9f9f9;
            border-radius: 4px;
            border-left: 4px solid #3498db;
        }
        
        .audio-player {
            margin-top: 15px;
            width: 100%;
        }
        
        #image-preview {
            max-width: 100%;
            margin-top: 15px;
            border-radius: 4px;
        }
        
        .recording {
            color: #e74c3c;
            font-weight: bold;
        }
        
        .voice-controls {
            display: flex;
            gap: 10px;
        }
    </style>
</head>
<body>
    <h1>Accessibility Assistant</h1>
    
    <div class="container">
        <div class="card">
            <h2>Image Description</h2>
            <p>Upload an image to get a detailed description and audio narration.</p>
            
            <input type="file" id="image-input" accept="image/*">
            <button class="button" id="upload-image">Describe Image</button>
            
            <div id="image-result" class="result" style="display: none;">
                <img id="image-preview" src="#" alt="Preview">
                <h3>Description:</h3>
                <p id="image-description"></p>
                <h3>Audio Narration:</h3>
                <audio id="audio-player" controls class="audio-player"></audio>
            </div>
        </div>
        
        <div class="card">
            <h2>Voice Command</h2>
            <p>Use your voice to interact with the assistant.</p>
            
            <div class="voice-controls">
                <button class="button" id="start-recording">Start Recording</button>
                <button class="button" id="stop-recording" disabled>Stop Recording</button>
            </div>
            
            <div id="recording-status"></div>
            
            <div id="voice-result" class="result" style="display: none;">
                <h3>Response:</h3>
                <p id="voice-response"></p>
            </div>
        </div>
    </div>
    
    <script>
        // 이미지 설명 기능
        document.getElementById('upload-image').addEventListener('click', async () => {
            const imageInput = document.getElementById('image-input');
            
            if (!imageInput.files || imageInput.files.length === 0) {
                alert('Please select an image file');
                return;
            }
            
            const formData = new FormData();
            formData.append('image', imageInput.files[0]);
            
            try {
                const response = await fetch('/api/accessibility/describe-image', {
                    method: 'POST',
                    body: formData
                });
                
                if (!response.ok) {
                    throw new Error('Failed to process image');
                }
                
                const result = await response.json();
                
                // 결과 표시
                document.getElementById('image-description').textContent = result.imageDescription;
                document.getElementById('audio-player').src = result.audioUrl;
                
                // 이미지 미리보기
                const reader = new FileReader();
                reader.onload = function(e) {
                    document.getElementById('image-preview').src = e.target.result;
                }
                reader.readAsDataURL(imageInput.files[0]);
                
                document.getElementById('image-result').style.display = 'block';
            } catch (error) {
                console.error('Error:', error);
                alert('Error processing image: ' + error.message);
            }
        });
        
        // 음성 명령 기능
        let mediaRecorder;
        let audioChunks = [];
        
        document.getElementById('start-recording').addEventListener('click', async () => {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                mediaRecorder = new MediaRecorder(stream);
                
                mediaRecorder.addEventListener('dataavailable', event => {
                    audioChunks.push(event.data);
                });
                
                mediaRecorder.addEventListener('stop', async () => {
                    const audioBlob = new Blob(audioChunks, { type: 'audio/wav' });
                    
                    const formData = new FormData();
                    formData.append('audio', audioBlob);
                    
                    try {
                        const response = await fetch('/api/accessibility/voice-command', {
                            method: 'POST',
                            body: formData
                        });
                        
                        if (!response.ok) {
                            throw new Error('Failed to process voice command');
                        }
                        
                        const result = await response.json();
                        
                        // 결과 표시
                        document.getElementById('voice-response').textContent = result.response;
                        document.getElementById('voice-result').style.display = 'block';
                    } catch (error) {
                        console.error('Error:', error);
                        alert('Error processing voice command: ' + error.message);
                    }
                    
                    // 녹음 상태 업데이트
                    document.getElementById('recording-status').textContent = '';
                    audioChunks = [];
                });
                
                mediaRecorder.start();
                
                document.getElementById('start-recording').disabled = true;
                document.getElementById('stop-recording').disabled = false;
                document.getElementById('recording-status').innerHTML = '<span class="recording">Recording...</span>';
            } catch (error) {
                console.error('Error accessing microphone:', error);
                alert('Error accessing microphone: ' + error.message);
            }
        });
        
        document.getElementById('stop-recording').addEventListener('click', () => {
            if (mediaRecorder && mediaRecorder.state !== 'inactive') {
                mediaRecorder.stop();
                
                document.getElementById('start-recording').disabled = false;
                document.getElementById('stop-recording').disabled = true;
                document.getElementById('recording-status').textContent = 'Processing...';
            }
        });
    </script>
</body>
</html>
```

## 최적화 및 성능 고려사항

멀티모달 AI를 통합할 때 고려해야 할 중요한 최적화 및 성능 사항을 살펴보겠습니다.

### 이미지 및 오디오 처리 최적화

```java
package com.example.multimodal.util;

import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;

import javax.imageio.ImageIO;
import java.awt.*;
import java.awt.image.BufferedImage;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;

@Component
public class MediaProcessor {

    // 이미지 크기 최적화
    public byte[] optimizeImage(MultipartFile imageFile, int maxWidth, int maxHeight) throws IOException {
        // 원본 이미지 로드
        BufferedImage originalImage = ImageIO.read(imageFile.getInputStream());
        
        // 원본 크기
        int originalWidth = originalImage.getWidth();
        int originalHeight = originalImage.getHeight();
        
        // 리사이징이 필요한지 확인
        if (originalWidth <= maxWidth && originalHeight <= maxHeight) {
            return imageFile.getBytes(); // 리사이징 불필요
        }
        
        // 종횡비 유지하며 크기 계산
        double widthRatio = (double) maxWidth / originalWidth;
        double heightRatio = (double) maxHeight / originalHeight;
        double ratio = Math.min(widthRatio, heightRatio);
        
        int newWidth = (int) (originalWidth * ratio);
        int newHeight = (int) (originalHeight * ratio);
        
        // 이미지 리사이징
        BufferedImage resizedImage = new BufferedImage(newWidth, newHeight, BufferedImage.TYPE_INT_RGB);
        Graphics2D g = resizedImage.createGraphics();
        g.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BILINEAR);
        g.drawImage(originalImage, 0, 0, newWidth, newHeight, null);
        g.dispose();
        
        // 바이트 배열로 변환
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        ImageIO.write(resizedImage, "JPEG", outputStream);
        
        return outputStream.toByteArray();
    }
    
    // 오디오 정규화
    public byte[] normalizeAudio(byte[] audioData) {
        // 실제 구현에서는 오디오 처리 라이브러리(ex. JavaSound API, FFmpeg)를 사용하여
        // 볼륨 정규화, 노이즈 감소 등의 오디오 처리를 수행합니다.
        // 이 예제에서는 간단히 원본 오디오를 반환합니다.
        return audioData;
    }
}
```

### 캐싱 구현

```java
package com.example.multimodal.config;

import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
                .expireAfterWrite(1, TimeUnit.HOURS)
                .maximumSize(100)
                .recordStats());
        return cacheManager;
    }
}
```

캐싱을 활용한 서비스 예시:

```java
package com.example.multimodal.service;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Base64;

@Service
public class CachedImageAnalysisService {

    private final ImageUnderstandingService imageUnderstandingService;

    public CachedImageAnalysisService(ImageUnderstandingService imageUnderstandingService) {
        this.imageUnderstandingService = imageUnderstandingService;
    }

    @Cacheable(value = "imageAnalysis", key = "#imageHash + '-' + #question")
    public String analyzeImageWithCache(byte[] imageData, String question, String imageHash) throws Exception {
        // 이미지 데이터를 리소스로 변환하고 분석
        // 여기서는 원본 ImageUnderstandingService 메서드를 호출합니다.
        return "Image analysis result"; // 실제 구현에서는 실제 분석 결과 반환
    }

    // 이미지 데이터에 대한 해시 생성
    public String generateImageHash(byte[] imageData) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] hashBytes = md.digest(imageData);
            return Base64.getEncoder().encodeToString(hashBytes);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("Failed to generate image hash", e);
        }
    }
}
```

### 비동기 처리

```java
package com.example.multimodal.service;

import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.concurrent.CompletableFuture;

@Service
public class AsyncMultimodalService {

    private final ImageUnderstandingService imageUnderstandingService;
    private final TextToSpeechService textToSpeechService;

    public AsyncMultimodalService(
            ImageUnderstandingService imageUnderstandingService,
            TextToSpeechService textToSpeechService) {
        this.imageUnderstandingService = imageUnderstandingService;
        this.textToSpeechService = textToSpeechService;
    }

    @Async
    public CompletableFuture<String> analyzeImageAsync(MultipartFile imageFile, String question) throws IOException {
        String analysis = imageUnderstandingService.analyzeImage(imageFile, question);
        return CompletableFuture.completedFuture(analysis);
    }

    @Async
    public CompletableFuture<String> generateSpeechAsync(String text, String voiceId) throws IOException {
        String audioPath = textToSpeechService.generateSpeech(text, voiceId);
        return CompletableFuture.completedFuture(audioPath);
    }

    // 병렬 처리 예시
    public CompletableFuture<ImageTextAudioResult> processImageWithAudio(MultipartFile imageFile) throws IOException {
        CompletableFuture<String> imageAnalysisFuture = analyzeImageAsync(
                imageFile,
                "Describe this image in detail."
        );
        
        return imageAnalysisFuture.thenCompose(analysis -> {
            try {
                return generateSpeechAsync(analysis, "default-voice-id")
                        .thenApply(audioPath -> new ImageTextAudioResult(analysis, audioPath));
            } catch (IOException e) {
                throw new RuntimeException("Failed to generate speech", e);
            }
        });
    }

    public record ImageTextAudioResult(String textDescription, String audioPath) {}
}
```

## 보안 및 개인정보 보호 고려사항

멀티모달 AI 애플리케이션을 개발할 때 보안 및 개인정보 보호에 특별히 주의해야 합니다.

### 입력 검증 및 제한

```java
package com.example.multimodal.validator;

import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;

import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

@Component
public class MediaValidator {

    private static final Set<String> ALLOWED_IMAGE_TYPES = new HashSet<>(Arrays.asList(
            "image/jpeg", "image/png", "image/gif", "image/webp"
    ));
    
    private static final Set<String> ALLOWED_AUDIO_TYPES = new HashSet<>(Arrays.asList(
            "audio/wav", "audio/mpeg", "audio/mp3", "audio/mp4", "audio/webm"
    ));
    
    private static final long MAX_IMAGE_SIZE = 10 * 1024 * 1024; // 10MB
    private static final long MAX_AUDIO_SIZE = 25 * 1024 * 1024; // 25MB

    public void validateImage(MultipartFile file) {
        // 파일 존재 확인
        if (file == null || file.isEmpty()) {
            throw new IllegalArgumentException("Image file is required");
        }
        
        // 파일 유형 확인
        String contentType = file.getContentType();
        if (contentType == null || !ALLOWED_IMAGE_TYPES.contains(contentType.toLowerCase())) {
            throw new IllegalArgumentException("Invalid image format. Allowed formats: JPEG, PNG, GIF, WebP");
        }
        
        // 파일 크기 확인
        if (file.getSize() > MAX_IMAGE_SIZE) {
            throw new IllegalArgumentException("Image size exceeds the maximum allowed size (10MB)");
        }
    }

    public void validateAudio(MultipartFile file) {
        // 파일 존재 확인
        if (file == null || file.isEmpty()) {
            throw new IllegalArgumentException("Audio file is required");
        }
        
        // 파일 유형 확인
        String contentType = file.getContentType();
        if (contentType == null || !ALLOWED_AUDIO_TYPES.contains(contentType.toLowerCase())) {
            throw new IllegalArgumentException("Invalid audio format. Allowed formats: WAV, MP3, MP4, WebM");
        }
        
        // 파일 크기 확인
        if (file.getSize() > MAX_AUDIO_SIZE) {
            throw new IllegalArgumentException("Audio size exceeds the maximum allowed size (25MB)");
        }
    }

    public void validateTextPrompt(String prompt) {
        if (prompt == null || prompt.trim().isEmpty()) {
            throw new IllegalArgumentException("Text prompt is required");
        }
        
        if (prompt.length() > 4000) {
            throw new IllegalArgumentException("Text prompt exceeds maximum length (4000 characters)");
        }
        
        // 민감한 콘텐츠 필터링 로직 추가 가능
    }
}
```

### 개인정보 제거 유틸리티

```java
package com.example.multimodal.util;

import org.springframework.stereotype.Component;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

@Component
public class PrivacyUtils {

    // 이메일 주소 패턴
    private static final Pattern EMAIL_PATTERN = 
            Pattern.compile("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,6}");
    
    // 전화번호 패턴 (다양한 형식 지원)
    private static final Pattern PHONE_PATTERN = 
            Pattern.compile("(\\+\\d{1,3}[- ]?)?\\d{3}[- ]?\\d{3,4}[- ]?\\d{4}");
    
    // 주민등록번호 패턴 (한국)
    private static final Pattern KR_ID_PATTERN = 
            Pattern.compile("\\d{6}[- ]?[1-4]\\d{6}");
    
    // 신용카드 번호 패턴
    private static final Pattern CREDIT_CARD_PATTERN = 
            Pattern.compile("\\d{4}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}");

    // 텍스트에서 개인정보 제거
    public String sanitizeText(String text) {
        if (text == null || text.isEmpty()) {
            return text;
        }
        
        String sanitized = text;
        
        // 이메일 주소 마스킹
        Matcher emailMatcher = EMAIL_PATTERN.matcher(sanitized);
        while (emailMatcher.find()) {
            String email = emailMatcher.group();
            String maskedEmail = maskEmail(email);
            sanitized = sanitized.replace(email, maskedEmail);
        }
        
        // 전화번호 마스킹
        Matcher phoneMatcher = PHONE_PATTERN.matcher(sanitized);
        while (phoneMatcher.find()) {
            String phone = phoneMatcher.group();
            String maskedPhone = maskPhone(phone);
            sanitized = sanitized.replace(phone, maskedPhone);
        }
        
        // 주민등록번호 마스킹
        Matcher idMatcher = KR_ID_PATTERN.matcher(sanitized);
        while (idMatcher.find()) {
            String id = idMatcher.group();
            String maskedId = maskId(id);
            sanitized = sanitized.replace(id, maskedId);
        }
        
        // 신용카드 번호 마스킹
        Matcher ccMatcher = CREDIT_CARD_PATTERN.matcher(sanitized);
        while (ccMatcher.find()) {
            String cc = ccMatcher.group();
            String maskedCc = maskCreditCard(cc);
            sanitized = sanitized.replace(cc, maskedCc);
        }
        
        return sanitized;
    }
    
    // 이메일 마스킹 (username의 일부만 표시)
    private String maskEmail(String email) {
        int atIndex = email.indexOf('@');
        if (atIndex <= 1) {
            return "***@" + email.substring(atIndex + 1);
        }
        
        String username = email.substring(0, atIndex);
        String domain = email.substring(atIndex);
        
        int visibleChars = Math.min(3, username.length());
        return username.substring(0, visibleChars) + "***" + domain;
    }
    
    // 전화번호 마스킹 (마지막 4자리만 표시)
    private String maskPhone(String phone) {
        String digitsOnly = phone.replaceAll("[^0-9]", "");
        
        if (digitsOnly.length() <= 4) {
            return "******";
        }
        
        return "***-***-" + digitsOnly.substring(digitsOnly.length() - 4);
    }
    
    // 주민등록번호 마스킹
    private String maskId(String id) {
        String digitsOnly = id.replaceAll("[^0-9]", "");
        
        if (digitsOnly.length() < 7) {
            return "******-*******";
        }
        
        return digitsOnly.substring(0, 6) + "-*******";
    }
    
    // 신용카드 번호 마스킹 (첫 4자리와 마지막 4자리만 표시)
    private String maskCreditCard(String cc) {
        String digitsOnly = cc.replaceAll("[^0-9]", "");
        
        if (digitsOnly.length() < 8) {
            return "****-****-****-****";
        }
        
        return digitsOnly.substring(0, 4) + "-****-****-" + 
               digitsOnly.substring(digitsOnly.length() - 4);
    }
}
```

## 결론

이 장에서는 Spring AI를 사용하여 멀티모달 AI 기능을 자바 애플리케이션에 통합하는 방법을 살펴보았습니다. 이미지 생성 및 이해, 음성 인식 및 합성과 같은 주요 멀티모달 기능을 구현하는 방법과 이를 활용한 실제 애플리케이션 사례를 알아보았습니다.

Spring AI의 멀티모달 지원은 점점 확장되고 있으며, 다양한 AI 모델 및 프로바이더를 통합하여 풍부한 사용자 경험을 제공하는 애플리케이션을 구축할 수 있게 해줍니다. 최적화, 캐싱, 비동기 처리 및 보안과 같은 고려사항을 적용하여 안정적이고 효율적인 멀티모달 AI 애플리케이션을 구현할 수 있습니다.

다음 장에서는 Spring AI의 스트리밍 응답과 비동기 처리에 대해 자세히 알아보겠습니다. 이러한 기능은 대화형 AI와 멀티모달 AI 애플리케이션에서 특히 중요한 역할을 합니다.