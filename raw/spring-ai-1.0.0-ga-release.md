# Spring AI 1.0.0 GA Release Announcement

**Release Date**: May 20, 2025

## Overview

Spring AI 1.0.0 has reached General Availability (GA), marking a significant milestone for the Spring AI project. This release brings enterprise-ready AI capabilities to the Spring ecosystem.

## Key Features

### ChatClient - The Core API
- Primary interface for interacting with AI models
- Portable and easy-to-use API
- Supports 20 AI Models from various providers (Anthropic to ZhiPu)

### Augmented LLM Capabilities
The concept of the Augmented LLM adds to the base model interaction with:
- Data retrieval
- Conversational memory  
- Tool calling
These capabilities allow you to bring your own data and external APIs directly into the model's reasoning process.

## Maven Dependencies

All artifacts are available in Maven Central. Use the Spring AI BOM to manage dependencies:

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
```

## Getting Started

- Create new projects on Spring Initializr: https://start.spring.io
- Reference documentation: Getting Started section

## Migration Support

Automate the upgrade process to 1.0.0-GA using an OpenRewrite recipe:
- Find the recipe and usage instructions at Arconia Spring AI Migrations
- The recipe helps apply many of the necessary code changes for this version

## Community Contributions

### Partner Blogs and Videos:
- **Microsoft Azure**: Blog and Video (Special thanks to Asir Selvasingh who helped launch Spring AI at Spring One conference in Vegas 2023)
- **AWS**: Blog - "Spring AI 1.0 Brings AI to the Developer Masses"
- **Cloud Foundry**: Video - "Spring AI and CloudFoundry: Bootiful, Agentic, Production-Worthy, Cloud-Native Systems and Services"
- **Elastic**: Blog - "Spring AI and Elasticsearch as your vector database"
- **Redis**: Blog - "Build fast, production-worthy AI apps with Spring AI and Redis"
- **MongoDB**: Blog - "Spring AI and MongoDB: How to Build RAG Applications"

## Visual Updates

- New Spring AI logo created by designer Jorge Rigabert
- Special thanks to Sergi Almar, organizer of the Spring IO conference

## Source

Original announcement: https://spring.io/blog/2025/05/20/spring-ai-1-0-GA-released/