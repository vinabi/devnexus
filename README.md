# Java AI Agents with Spring AI
## DevNexus Workshop Documentation

This document explains the concepts and architecture used in the DevNexus Spring AI Workshop for building production-ready AI agents using Java, Spring AI, and AWS services.

The focus is understanding architecture and concepts rather than copying full implementations.

Reference workshop material: https://s12d.com/spring-ai-devnexus

---

# 1 Introduction to Spring AI

Spring AI is a framework that integrates **Large Language Models into Java applications** using the familiar Spring programming model.

It allows AI capabilities to be implemented using standard enterprise architecture patterns.

Spring AI provides abstractions for:

- LLM interaction
- vector databases
- AI tools
- conversation memory
- external services
- observability

Spring AI allows developers to build **AI agents that behave like normal microservices**.

---

## Key Features

### Model Abstraction

Spring AI introduces `ChatClient`, a provider-agnostic API.

This allows switching model providers without changing business logic.

Supported providers include:

| Provider |
|---|
Amazon Bedrock |
OpenAI |
Anthropic |
Google |
Local models |

Example concept:

```java
chatClient.prompt()
    .user(prompt)
    .stream()
    .content();
````

---

### Streaming Responses

Responses can stream tokens instead of returning a full response.

Benefits:

* faster UX
* real-time chat
* lower latency perception

---

### Structured Output

Responses can be converted into Java objects.

Example use cases:

* returning JSON
* tables
* structured responses
* workflows

---

### Tool Calling

AI can call Java methods to access external systems.

Examples:

* weather APIs
* database queries
* sending notifications
* triggering workflows

Tools use the `@Tool` annotation.

---

### Advisors

Spring AI provides advisors that automatically modify prompts.

Examples:

| Advisor                  | Purpose             |
| ------------------------ | ------------------- |
| MessageChatMemoryAdvisor | conversation memory |
| QuestionAnswerAdvisor    | RAG integration     |

---

### Vector Store Integration

Spring AI supports semantic search using vector databases.

Examples:

| Vector Store |
| ------------ |
| PgVector     |
| Redis        |
| OpenSearch   |

---

### Observability

Spring Boot observability tools are supported.

Examples:

* logs
* tracing
* metrics

---

# 2 Bootstrapping the AI Agent

A Spring Boot project is created using Spring Initializr.

Typical dependencies:

| Dependency                               | Purpose                    |
| ---------------------------------------- | -------------------------- |
| spring-ai-starter-model-bedrock-converse | Amazon Bedrock integration |
| spring-boot-starter-web                  | REST API                   |
| spring-boot-starter-webflux              | streaming                  |
| spring-boot-starter-actuator             | monitoring                 |

---

## Configuration

Example `application.properties`

```properties
logging.level.org.springframework.ai=DEBUG

spring.ai.bedrock.converse.chat.options.model=global.anthropic.claude-sonnet
spring.ai.bedrock.converse.chat.options.max-tokens=4096
```

---

## AI Service

Interaction with the LLM is handled inside a service.

Example structure:

```java
@Service
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public Flux<String> chat(String prompt) {
        return chatClient.prompt()
            .user(prompt)
            .stream()
            .content();
    }
}
```

---

## REST Endpoint

The AI agent exposes an endpoint:

```
POST /invocations
```

Example request:

```json
{
  "prompt": "Write a short story about unicorns"
}
```

---

## Running the Agent

```bash
./mvnw spring-boot:run
```

Example test:

```bash
curl -X POST localhost:8080/invocations
```

---

# 3 Agent Persona

AI agents require a defined identity.

This is implemented using a **system prompt**.

The system prompt is prepended to every user request.

Example:

```java
.defaultSystem("""
You are an AI assistant for Unicorn Rentals.
Be helpful and concise.
If information is unknown say I don't know.
""")
```

Changing the system prompt changes the agent's behavior.

---

# 4 Conversation Memory

LLMs are stateless.

Each request is independent.

Conversation memory stores previous interactions.

Spring AI provides **ChatMemory** implementations.

---

## Memory Repository Options

| Repository                    | Persistence         |
| ----------------------------- | ------------------- |
| InMemoryChatMemoryRepository  | temporary           |
| JdbcChatMemoryRepository      | relational database |
| CassandraChatMemoryRepository | distributed systems |
| Neo4jChatMemoryRepository     | graph systems       |

The workshop uses **PostgreSQL JDBC memory**.

---

## Configuration

```properties
spring.ai.chat.memory.repository.jdbc.initialize-schema=always
```

---

## Memory Setup Concept

```java
var chatMemory = MessageWindowChatMemory.builder()
    .maxMessages(20)
    .build();

MessageChatMemoryAdvisor.builder(chatMemory).build();
```

---

## Conversation IDs

Each conversation needs an identifier.

Example:

```
userId : sessionId
```

This separates conversations between users.

---

# 5 Retrieval Augmented Generation (RAG)

LLMs cannot access private company knowledge.

RAG allows grounding responses using external data.

---

## RAG Workflow

1 Documents split into chunks
2 Embeddings generated
3 Stored in vector database
4 Query converted to embedding
5 Similar chunks retrieved
6 Context added to prompt
7 Model generates grounded response

---

## Vector Store

The workshop uses **PgVector with PostgreSQL**.

Configuration example:

```properties
spring.ai.vectorstore.pgvector.initialize-schema=true
spring.ai.vectorstore.pgvector.dimensions=1024
```

---

## Embedding Model

Amazon Titan embedding model generates vectors.

Example:

```properties
spring.ai.model.embedding=bedrock-titan
spring.ai.bedrock.titan.embedding.model=amazon.titan-embed-text-v2
```

---

## RAG Advisor

```java
QuestionAnswerAdvisor.builder(vectorStore).build();
```

---

## Loading Knowledge

Documents can be added to the vector store.

Example:

```java
vectorStore.add(List.of(new Document(content)));
```

Endpoint example:

```
POST /load
```

---

# 6 Tool Calling

Tool calling allows AI to execute Java functions.

Tools use `@Tool`.

---

## Example Tool

```java
class DateTimeTools {

    @Tool(description="Get current time")
    public String getCurrentTime(String timezone) {
        return ZonedDateTime.now().toString();
    }
}
```

---

## Weather Tool Concept

```java
class WeatherTools {

    @Tool(description="Get weather forecast")
    public String getWeather(String city, String date) {
        // call weather API
    }
}
```

---

## Registering Tools

```java
.defaultTools(new DateTimeTools(), new WeatherTools())
```

---

## Tool Calling Flow

1 User asks question
2 Model detects tool requirement
3 Model generates parameters
4 Tool executed
5 Result returned to model
6 Model produces final answer

---

# 7 Model Context Protocol (MCP)

MCP is an open standard for connecting AI models with external tools.

Architecture:

| Component  | Role           |
| ---------- | -------------- |
| MCP Server | exposes tools  |
| MCP Client | consumes tools |
| Protocol   | JSON-RPC       |

---

# 7.1 MCP Server

Add dependency:

```
spring-ai-starter-mcp-server-webmvc
```

Configuration:

```yaml
spring:
  ai:
    mcp:
      server:
        name: unicorn-store
        version: 1.0
        protocol: STREAMABLE
```

---

## Tool Registration

```java
@Bean
public ToolCallbackProvider provider() {
    return MethodToolCallbackProvider.builder()
        .toolObjects(new UnicornTools())
        .build();
}
```

---

# 7.2 MCP Client

Dependency:

```
spring-ai-starter-mcp-client
```

Configuration:

```properties
spring.ai.mcp.client.toolcallback.enabled=true
```

---

## MCP Connection

Environment variable:

```
SPRING_AI_MCP_CLIENT_STREAMABLEHTTP_CONNECTIONS_SERVER1_URL
```

Example:

```
http://localhost:8081
```

---

# 8 Security with Amazon Cognito

AI agents should be secured before production.

Amazon Cognito provides authentication.

---

## Authentication Flow

1 User logs into Cognito
2 Cognito returns JWT token
3 Client sends token with request
4 Spring Security validates token
5 API processes request

---

## Configuration

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=${COGNITO_ISSUER_URI}
```

---

## Security Configuration

```java
http.oauth2ResourceServer(oauth -> oauth.jwt());
```

---

## Accessing User Identity

```java
@AuthenticationPrincipal Jwt jwt
```

Used to generate conversation IDs.

---

# 9 Deploying MCP Server on Amazon EKS

The MCP server runs as a container on Kubernetes.

---

## Core Technologies

| Technology   | Purpose                |
| ------------ | ---------------------- |
| Amazon EKS   | Kubernetes cluster     |
| Amazon ECR   | container registry     |
| Jib          | build container images |
| Pod Identity | IAM permissions        |
| CSI Driver   | secret mounting        |

---

## Container Build

```bash
mvn compile jib:build
```

---

## Kubernetes Components

| Resource   | Purpose                 |
| ---------- | ----------------------- |
| Deployment | manages pods            |
| Service    | exposes pods internally |
| Ingress    | load balancer access    |

---

## Secret Management

Secrets are mounted using CSI driver.

Spring reads them via:

```
SPRING_CONFIG_IMPORT=configtree
```

---

# 10 Deploying AI Agent on EKS

Deployment steps:

1 Build container image
2 Push image to ECR
3 Create namespace
4 Configure service account
5 Deploy pods
6 Create service
7 Create ingress

---

## Environment Variables

```
SPRING_AI_MCP_CLIENT_STREAMABLEHTTP_CONNECTIONS_SERVER1_URL
SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI
```

---

# 11 Deploying on Amazon ECS

ECS provides container orchestration without Kubernetes.

The workshop uses **ECS Express Mode with Fargate**.

---

## ECS Components

| Component       | Purpose                 |
| --------------- | ----------------------- |
| Cluster         | group of services       |
| Task Definition | container configuration |
| Service         | running containers      |
| Fargate         | serverless compute      |

---

## Deployment Workflow

1 Build image with Jib
2 Push image to ECR
3 Update ECS service
4 Wait for deployment

---

## Monitoring

Logs are available in:

```
CloudWatch Logs
```

---

# Conclusion

This workshop demonstrates building a production ready AI agent using Java.

The architecture includes:

* LLM integration via Spring AI
* conversation memory
* retrieval augmented generation
* tool calling
* MCP tool integration
* authentication using Cognito
* deployment to Kubernetes and ECS

These patterns enable scalable enterprise AI applications.

```
