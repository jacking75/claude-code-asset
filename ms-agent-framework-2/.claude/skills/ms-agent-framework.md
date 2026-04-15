# Microsoft Agent Framework (C#) - AI 에이전트 구축 스킬

Microsoft Agent Framework(Microsoft.Agents.AI)로 AI 에이전트를 구축할 때 사용하는 스킬. 에이전트 생성, Tool 정의, 세션 관리, 스트리밍 응답, 미들웨어, Agent-as-Tool, 커스텀 에이전트, 워크플로 등 프로덕션 패턴을 안내한다. OpenAI, Azure OpenAI, Anthropic, Foundry, OpenRouter 등 다양한 Provider를 지원하는 C# 프로젝트에 적용 가능.

## 트리거 조건

다음 상황에서 이 스킬을 활성화한다:
- 코드에서 `Microsoft.Agents.AI`, `Microsoft.Agents.AI.Foundry`, `ChatClientAgent`, `AIAgent` 등을 import하거나 사용할 때
- 사용자가 "에이전트 만들어줘", "Agent Framework", "MS Agent", "Microsoft Agent" 등을 언급할 때
- C# 프로젝트에서 AI 에이전트, 함수 도구, 세션, 미들웨어를 구현할 때

다음 상황에서는 트리거하지 않는다:
- Semantic Kernel, AutoGen 등 다른 프레임워크를 명시적으로 사용할 때
- Python 프로젝트일 때

---

## 1. 아키텍처 개요

```
┌─────────────────────────────────────────────────┐
│           Microsoft Agent Framework             │
├─────────────────┬───────────────────────────────┤
│     Agents      │         Workflows             │
│  (단일 에이전트)  │  (그래프 기반 멀티에이전트)     │
├─────────────────┴───────────────────────────────┤
│  Building Blocks:                               │
│  - Model Clients (Chat Completions/Responses)   │
│  - Agent Session (상태 관리)                     │
│  - Context Providers (에이전트 메모리)            │
│  - Middleware (동작 인터셉트)                     │
│  - MCP Clients (도구 통합)                       │
├─────────────────────────────────────────────────┤
│  Providers:                                     │
│  Foundry | Azure OpenAI | OpenAI | Anthropic    │
│  Ollama | 기타 IChatClient 구현체                │
├─────────────────────────────────────────────────┤
│  Microsoft.Extensions.AI 추상화 기반             │
└─────────────────────────────────────────────────┘
```

### Agent vs Workflow 선택 기준

| Agent를 사용할 때 | Workflow를 사용할 때 |
|---|---|
| 작업이 개방형이거나 대화형일 때 | 프로세스가 잘 정의된 단계를 가질 때 |
| 자율적 도구 사용과 계획이 필요할 때 | 실행 순서에 대한 명시적 제어가 필요할 때 |
| 단일 LLM 호출(도구 포함 가능)로 충분할 때 | 여러 에이전트 또는 함수가 조율되어야 할 때 |

> 함수로 작업을 처리할 수 있다면, AI 에이전트 대신 함수를 사용하세요.

---

## 2. 필수 NuGet 패키지

### 핵심 패키지
```xml
<!-- 핵심 에이전트 프레임워크 -->
<PackageReference Include="Microsoft.Agents.AI" Version="*-*" />

<!-- Foundry 통합 (Azure AI Foundry 사용 시) -->
<PackageReference Include="Microsoft.Agents.AI.Foundry" Version="*-*" />

<!-- Azure AI Projects SDK -->
<PackageReference Include="Azure.AI.Projects" Version="*-*" />

<!-- Azure 인증 -->
<PackageReference Include="Azure.Identity" Version="*" />
```

### CLI 설치
```bash
dotnet add package Microsoft.Agents.AI --prerelease
dotnet add package Microsoft.Agents.AI.Foundry --prerelease
dotnet add package Azure.AI.Projects --prerelease
dotnet add package Azure.Identity
```

### Provider별 추가 패키지

| Provider | NuGet 패키지 |
|---|---|
| Azure OpenAI | `Azure.AI.OpenAI` |
| OpenAI | `OpenAI` |
| Azure AI Inference | `Azure.AI.Inference` |
| Anthropic | `Anthropic` |
| Anthropic via Foundry | `Anthropic.Foundry` |

---

## 3. 에이전트 생성 패턴

### 3.1 Azure AI Foundry (AIProjectClient)

```csharp
using System;
using Azure.AI.Projects;
using Azure.Identity;
using Microsoft.Agents.AI;

AIAgent agent = new AIProjectClient(
        new Uri("https://your-foundry-service.services.ai.azure.com/api/projects/your-foundry-project"),
        new DefaultAzureCredential())
    .AsAIAgent(
        model: "gpt-4o-mini",
        instructions: "You are a friendly assistant. Keep your answers brief.",
        name: "MyAgent");
```

### 3.2 OpenAI SDK 사용

```csharp
using OpenAI;
using Microsoft.Agents.AI;

// API Key 인증
OpenAIClient client = new OpenAIClient(new ApiKeyCredential(apiKey));

// Azure OpenAI 사용 시 (엔드포인트 지정)
var clientOptions = new OpenAIClientOptions() { Endpoint = new Uri(serviceUrl) };
OpenAIClient client = new OpenAIClient(new ApiKeyCredential(apiKey), clientOptions);

// 에이전트 생성
AIAgent agent = client.AsAIAgent(
    model: "gpt-4o-mini",
    instructions: "You are good at telling jokes.",
    name: "Joker");
```

### 3.3 Azure OpenAI (Azure 자격증명)

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;

var clientOptions = new OpenAIClientOptions() { Endpoint = new Uri(serviceUrl) };
OpenAIClient client = new OpenAIClient(
    new BearerTokenPolicy(new DefaultAzureCredential(), "https://ai.azure.com/.default"),
    clientOptions);

AIAgent agent = client.AsAIAgent(
    model: "gpt-4o-mini",
    instructions: "You are a helpful assistant.",
    name: "Assistant");
```

### 3.4 Anthropic SDK 사용

```csharp
using Anthropic;
using Microsoft.Agents.AI;

var client = new AnthropicClient() { ApiKey = apiKey };
AIAgent agent = client.AsAIAgent(
    model: "claude-sonnet-4-20250514",
    instructions: "You are a helpful assistant.",
    name: "ClaudeAgent");
```

### 3.5 Anthropic via Foundry

```csharp
using Anthropic.Foundry;
using Microsoft.Agents.AI;

var client = new AnthropicFoundryClient(
    new AnthropicFoundryApiKeyCredentials(apiKey, resource));
AIAgent agent = client.AsAIAgent(
    model: deploymentName,
    instructions: "You are a helpful assistant.",
    name: "FoundryClaudeAgent");
```

### 3.6 ChatClientAgent (IChatClient 직접 사용)

```csharp
using Microsoft.Agents.AI;

var agent = new ChatClientAgent(chatClient,
    instructions: "You are a helpful assistant");
```

---

## 4. 에이전트 실행

### 4.1 Non-Streaming (RunAsync)

```csharp
// 간단한 텍스트 출력
Console.WriteLine(await agent.RunAsync("What is the largest city in France?"));

// AgentResponse 객체로 상세 접근
var response = await agent.RunAsync("What is the weather like in Amsterdam?");
Console.WriteLine(response.Text);           // 모든 TextContent 집계
Console.WriteLine(response.Messages.Count); // ChatMessage 수
```

### 4.2 Streaming (RunStreamingAsync)

```csharp
await foreach (var update in agent.RunStreamingAsync("Tell me a one-sentence fun fact."))
{
    Console.Write(update);  // 실시간 출력
}

// AgentResponseUpdate 상세 접근
await foreach (var update in agent.RunStreamingAsync("Tell me about AI."))
{
    Console.WriteLine(update.Text);
    Console.WriteLine(update.Contents.Count);
}
```

### 4.3 실행 옵션 (AgentRunOptions)

```csharp
using Microsoft.Extensions.AI;

var chatOptions = new ChatOptions()
{
    Tools = [AIFunctionFactory.Create(GetWeather)]
};

Console.WriteLine(await agent.RunAsync(
    "What is the weather like in Amsterdam?",
    options: new ChatClientAgentRunOptions(chatOptions)));
```

---

## 5. Function Tools (함수 도구)

### 5.1 함수 도구 정의

`System.ComponentModel.DescriptionAttribute`로 함수와 파라미터에 설명을 추가한다.

```csharp
using System.ComponentModel;

[Description("Get the weather for a given location.")]
static string GetWeather(
    [Description("The location to get the weather for.")] string location)
    => $"The weather in {location} is cloudy with a high of 15°C.";
```

### 5.2 에이전트에 함수 도구 제공

```csharp
using Microsoft.Extensions.AI;

AIAgent agent = new AIProjectClient(
    new Uri("<your-foundry-project-endpoint>"),
    new DefaultAzureCredential())
     .AsAIAgent(
        model: "gpt-4o-mini",
        instructions: "You are a helpful assistant",
        tools: [AIFunctionFactory.Create(GetWeather)]);

Console.WriteLine(await agent.RunAsync("What is the weather like in Amsterdam?"));
```

### 5.3 지원 도구 유형 매트릭스

| 도구 유형 | Chat Completion | Responses | Foundry | Anthropic | Ollama |
|---|:---:|:---:|:---:|:---:|:---:|
| Function Tools | O | O | O | O | O |
| Tool Approval | X | O | O | X | X |
| Code Interpreter | X | O | O | X | X |
| File Search | X | O | O | X | X |
| Web Search | O | O | X | X | X |
| Hosted MCP Tools | X | O | O | O | X |
| Local MCP Tools | O | O | O | O | O |

---

## 6. Agent-as-Tool (에이전트를 도구로 사용)

에이전트를 다른 에이전트의 함수 도구로 변환하여 에이전트 합성(composition)을 구현한다.

```csharp
// 1. 내부 에이전트 생성 (자체 도구 보유)
AIAgent weatherAgent = new AIProjectClient(
    new Uri("<your-foundry-project-endpoint>"),
    new DefaultAzureCredential())
     .AsAIAgent(
        model: "gpt-4o-mini",
        instructions: "You answer questions about the weather.",
        name: "WeatherAgent",
        description: "An agent that answers questions about the weather.",
        tools: [AIFunctionFactory.Create(GetWeather)]);

// 2. 메인 에이전트 생성 - 내부 에이전트를 도구로 제공
AIAgent agent = new AIProjectClient(
    new Uri("<your-foundry-project-endpoint>"),
    new DefaultAzureCredential())
     .AsAIAgent(
        model: "gpt-4o-mini",
        instructions: "You are a helpful assistant.",
        tools: [weatherAgent.AsAIFunction()]);  // <-- 핵심: .AsAIFunction()

// 3. 메인 에이전트가 필요시 내부 에이전트를 자동 호출
Console.WriteLine(await agent.RunAsync("What is the weather like in Amsterdam?"));
```

**아키텍처:**
```
Main Agent (외부)
  └── tools: [weatherAgent.AsAIFunction()]
        └── WeatherAgent (내부, 함수 도구로 변환)
              └── tools: [GetWeather 함수]
```

---

## 7. 세션 관리 (AgentSession)

### 7.1 기본 멀티턴 대화

```csharp
AgentSession session = await agent.CreateSessionAsync();

var first = await agent.RunAsync("My name is Alice.", session);
var second = await agent.RunAsync("What is my name?", session);
// second는 "Alice"를 기억함
```

### 7.2 세션 직렬화/복원

```csharp
// 직렬화 (영속 스토리지에 저장)
JsonElement serialized = agent.SerializeSession(session);

// 복원
AgentSession resumed = await agent.DeserializeSessionAsync(serialized);
```

### 7.3 기존 대화 ID에서 세션 생성

```csharp
// ChatClientAgent
AgentSession session = await chatClientAgent.CreateSessionAsync(conversationId);

// Service-managed 스토리지 사용 시 ConversationId 접근
ChatClientAgentSession typedSession = (ChatClientAgentSession)session;
Console.WriteLine(typedSession.ConversationId);
```

### 7.4 In-Memory 채팅 히스토리

```csharp
AIAgent agent = new OpenAIClient("<your_api_key>")
    .GetChatClient(modelName)
    .AsAIAgent(instructions: "You are a helpful assistant.", name: "Assistant");

AgentSession session = await agent.CreateSessionAsync();
Console.WriteLine(await agent.RunAsync("Tell me a joke about a pirate.", session));

// 인메모리 채팅 히스토리 접근
var provider = agent.GetService<InMemoryChatHistoryProvider>();
List<ChatMessage>? messages = provider?.GetMessages(session);
```

### 7.5 히스토리 크기 제한 (ChatReducer)

```csharp
AIAgent agent = new OpenAIClient("<your_api_key>")
    .GetChatClient(modelName)
    .AsAIAgent(new ChatClientAgentOptions
    {
        Name = "Assistant",
        ChatOptions = new() { Instructions = "You are a helpful assistant." },
        ChatHistoryProvider = new InMemoryChatHistoryProvider(
            new InMemoryChatHistoryProviderOptions
            {
                ChatReducer = new MessageCountingChatReducer(20)  // 최근 20개 메시지 유지
            })
    });
```

### 7.6 커스텀 ChatHistoryProvider 구현

```csharp
public sealed class SimpleInMemoryChatHistoryProvider : ChatHistoryProvider
{
    private readonly ProviderSessionState<State> _sessionState;

    public SimpleInMemoryChatHistoryProvider(
        Func<AgentSession?, State>? stateInitializer = null,
        string? stateKey = null)
    {
        this._sessionState = new ProviderSessionState<State>(
            stateInitializer ?? (_ => new State()),
            stateKey ?? this.GetType().Name);
    }

    public override string StateKey => this._sessionState.StateKey;

    protected override ValueTask<IEnumerable<ChatMessage>> ProvideChatHistoryAsync(
        InvokingContext context, CancellationToken cancellationToken = default) =>
        new(this._sessionState.GetOrInitializeState(context.Session).Messages);

    protected override ValueTask StoreChatHistoryAsync(
        InvokedContext context, CancellationToken cancellationToken = default)
    {
        var state = this._sessionState.GetOrInitializeState(context.Session);
        var allNewMessages = context.RequestMessages.Concat(context.ResponseMessages ?? []);
        state.Messages.AddRange(allNewMessages);
        this._sessionState.SaveState(context.Session, state);
        return default;
    }

    public sealed class State
    {
        [JsonPropertyName("messages")]
        public List<ChatMessage> Messages { get; set; } = [];
    }
}
```

> **중요**: `ChatHistoryProvider` 인스턴스는 모든 세션에서 공유되므로, 세션별 상태를 프로바이더 인스턴스 필드에 저장하면 안 된다. `ProviderSessionState<T>` 유틸리티를 사용하여 `AgentSession` 자체에 저장한다.

---

## 8. 미들웨어 (Middleware)

### 8.1 미들웨어 유형

| 유형 | 설명 |
|---|---|
| **Agent Run Middleware** | 에이전트 실행 전/후 가로채기 (입출력 검사/수정) |
| **Function Calling Middleware** | 함수 도구 호출 전/후 가로채기 |
| **IChatClient Middleware** | IChatClient 호출 전/후 가로채기 (원시 메시지 접근) |

### 8.2 Agent Run Middleware

```csharp
// Non-streaming 미들웨어
async Task<AgentResponse> CustomAgentRunMiddleware(
    IEnumerable<ChatMessage> messages,
    AgentSession? session,
    AgentRunOptions? options,
    AIAgent innerAgent,
    CancellationToken cancellationToken)
{
    Console.WriteLine($"Input: {messages.Count()}");
    var response = await innerAgent.RunAsync(messages, session, options, cancellationToken);
    Console.WriteLine($"Output: {response.Messages.Count}");
    return response;
}

// Streaming 미들웨어
async IAsyncEnumerable<AgentResponseUpdate> CustomAgentRunStreamingMiddleware(
    IEnumerable<ChatMessage> messages,
    AgentSession? session,
    AgentRunOptions? options,
    AIAgent innerAgent,
    [EnumeratorCancellation] CancellationToken cancellationToken)
{
    await foreach (var update in innerAgent.RunStreamingAsync(messages, session, options, cancellationToken))
    {
        yield return update;
    }
}

// 등록
var middlewareEnabledAgent = baseAgent
    .AsBuilder()
        .Use(runFunc: CustomAgentRunMiddleware, runStreamingFunc: CustomAgentRunStreamingMiddleware)
    .Build();
```

### 8.3 Function Calling Middleware

```csharp
async ValueTask<object?> CustomFunctionCallingMiddleware(
    AIAgent agent,
    FunctionInvocationContext context,
    Func<FunctionInvocationContext, CancellationToken, ValueTask<object?>> next,
    CancellationToken cancellationToken)
{
    Console.WriteLine($"Function Name: {context!.Function.Name}");
    var result = await next(context, cancellationToken);
    Console.WriteLine($"Function Call Result: {result}");
    return result;
}

// 등록
var middlewareEnabledAgent = baseAgent
    .AsBuilder()
        .Use(CustomFunctionCallingMiddleware)
    .Build();
```

### 8.4 IChatClient Middleware

```csharp
async Task<ChatResponse> CustomChatClientMiddleware(
    IEnumerable<ChatMessage> messages,
    ChatOptions? options,
    IChatClient innerChatClient,
    CancellationToken cancellationToken)
{
    var response = await innerChatClient.GetResponseAsync(messages, options, cancellationToken);
    return response;
}

// 등록 방법 1: Builder 패턴
var middlewareEnabledChatClient = chatClient
    .AsBuilder()
        .Use(getResponseFunc: CustomChatClientMiddleware, getStreamingResponseFunc: null)
    .Build();
var agent = new ChatClientAgent(middlewareEnabledChatClient, instructions: "...");

// 등록 방법 2: clientFactory 사용
var agent = new AIProjectClient(new Uri(endpoint), new DefaultAzureCredential())
    .AsAIAgent(
        model: "gpt-4o-mini",
        instructions: "You are a helpful assistant.",
        clientFactory: (chatClient) => chatClient
            .AsBuilder()
                .Use(getResponseFunc: CustomChatClientMiddleware, getStreamingResponseFunc: null)
            .Build());
```

### 8.5 가드레일 미들웨어 (Termination & Guardrails)

```csharp
async Task<AgentResponse> GuardrailMiddleware(
    IEnumerable<ChatMessage> messages,
    AgentSession? session,
    AgentRunOptions? options,
    AIAgent innerAgent,
    CancellationToken cancellationToken)
{
    // 입력 검증 - 민감한 단어 차단
    var lastMessage = messages.LastOrDefault()?.Text?.ToLower() ?? "";
    string[] blockedWords = ["password", "secret", "credentials"];

    foreach (var word in blockedWords)
    {
        if (lastMessage.Contains(word))
        {
            return new AgentResponse([new ChatMessage(ChatRole.Assistant,
                $"Sorry, I cannot process requests containing '{word}'.")]);
        }
    }

    // 에이전트 실행
    var response = await innerAgent.RunAsync(messages, session, options, cancellationToken);

    // 출력 검증 - 응답 길이 제한
    var responseText = response.Messages.LastOrDefault()?.Text ?? "";
    if (responseText.Length > 5000)
    {
        return new AgentResponse([new ChatMessage(ChatRole.Assistant,
            responseText.Substring(0, 5000) + "... [truncated]")]);
    }

    return response;
}

var guardedAgent = agent.AsBuilder()
    .Use(runFunc: GuardrailMiddleware, runStreamingFunc: null)
    .Build();
```

> **주의**: `runFunc`와 `runStreamingFunc`를 모두 제공하는 것이 이상적이다. Non-streaming만 제공하면, streaming 호출에서도 non-streaming 모드로 실행된다.

---

## 9. 핵심 클래스/인터페이스 레퍼런스

| 클래스/인터페이스 | 역할 |
|---|---|
| `AIAgent` | 모든 에이전트의 기본 추상 클래스 |
| `ChatClientAgent` | IChatClient 기반 에이전트 구현 |
| `IChatClient` | Microsoft.Extensions.AI 채팅 클라이언트 인터페이스 |
| `AIFunction` | 함수 도구 추상화 |
| `AIFunctionFactory` | C# 메서드 → AIFunction 변환 팩토리 |
| `AgentResponse` | Non-streaming 응답 (.Text, .Messages) |
| `AgentResponseUpdate` | Streaming 응답 청크 (.Text, .Contents) |
| `AgentSession` | 대화 상태 컨테이너 (추상 클래스) |
| `ChatClientAgentSession` | AgentSession 상속, .ConversationId 접근 |
| `AgentRunOptions` | 실행 옵션 기본 타입 |
| `ChatClientAgentRunOptions` | ChatClientAgent 전용 실행 옵션 |
| `ChatOptions` | Microsoft.Extensions.AI 채팅 옵션 |
| `ChatMessage` | 메시지 표현 (Microsoft.Extensions.AI) |
| `ChatHistoryProvider` | 히스토리 프로바이더 기본 클래스 |
| `InMemoryChatHistoryProvider` | 인메모리 채팅 히스토리 내장 구현 |
| `MessageCountingChatReducer` | 메시지 수 기반 히스토리 축소기 |
| `ProviderSessionState<T>` | 세션 상태 유틸리티 |
| `FunctionInvocationContext` | 함수 호출 미들웨어 컨텍스트 |

---

## 10. 핵심 확장 메서드

| 메서드 | 설명 |
|---|---|
| `.AsAIAgent(model, instructions, ...)` | 클라이언트 → AIAgent 변환 |
| `.AsAIFunction()` | AIAgent → 함수 도구 변환 (Agent-as-Tool) |
| `.RunAsync(message, session?, options?)` | Non-streaming 실행 |
| `.RunStreamingAsync(message, session?, options?)` | Streaming 실행 |
| `.CreateSessionAsync()` | 새 세션 생성 |
| `.SerializeSession(session)` | 세션 직렬화 |
| `.DeserializeSessionAsync(json)` | 세션 복원 |
| `.AsBuilder()` | 미들웨어 추가용 빌더 반환 |
| `.Build()` | 미들웨어 적용된 새 에이전트 생성 |
| `.GetService<T>()` | 에이전트 서비스 접근 |

---

## 11. 메시지 타입 (Microsoft.Extensions.AI)

| 타입 | 설명 |
|---|---|
| `TextContent` | 텍스트 콘텐츠 (입출력) |
| `DataContent` | 바이너리 콘텐츠 (이미지, 오디오, 비디오) |
| `UriContent` | URL 기반 콘텐츠 |
| `FunctionCallContent` | 함수 도구 호출 요청 |
| `FunctionResultContent` | 함수 도구 호출 결과 |

---

## 12. 프로덕션 체크리스트

- [ ] `DefaultAzureCredential` 대신 `ManagedIdentityCredential` 등 특정 자격증명 사용
- [ ] 세션은 동일한 에이전트/프로바이더 구성으로만 복원
- [ ] ChatHistoryProvider에 세션별 상태를 인스턴스 필드에 저장하지 않기
- [ ] 미들웨어에서 runFunc과 runStreamingFunc 모두 제공
- [ ] FunctionInvocationContext.Terminate 사용 시 채팅 기록 불일치 주의
- [ ] 히스토리 크기 관리를 위해 ChatReducer 적용 고려

---

## 참고 문서

- [공식 Overview](https://learn.microsoft.com/en-us/agent-framework/overview/?pivots=programming-language-csharp)
- [Agent Types](https://learn.microsoft.com/en-us/agent-framework/agents/)
- [Tools](https://learn.microsoft.com/en-us/agent-framework/agents/tools/)
- [Function Tools](https://learn.microsoft.com/en-us/agent-framework/agents/tools/function-tools)
- [Running Agents](https://learn.microsoft.com/en-us/agent-framework/agents/running-agents)
- [Sessions](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/session)
- [Storage](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/storage)
- [Middleware](https://learn.microsoft.com/en-us/agent-framework/agents/middleware/)
- [GitHub 샘플](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples)
