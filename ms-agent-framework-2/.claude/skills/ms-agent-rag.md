# Microsoft Agent Framework RAG (C#) - 검색 증강 생성 스킬

에이전트에 RAG(Retrieval-Augmented Generation) 기능을 추가할 때 사용하는 스킬. TextSearchProvider, File Search, Web Search, Context Providers, ChatHistoryMemoryProvider, Neo4j GraphRAG 패턴을 안내한다.

## 트리거 조건

다음 상황에서 이 스킬을 활성화한다:
- 코드에서 `TextSearchProvider`, `AIContextProvider`, `ChatHistoryMemoryProvider`, `FileSearchToolDefinition`, `WebSearchToolDefinition` 등을 사용할 때
- 사용자가 "RAG", "검색 증강", "문서 검색", "벡터 검색", "컨텍스트 프로바이더", "메모리" 등을 언급할 때
- 에이전트에 외부 지식/문서/웹 검색 기능을 추가하려 할 때

---

## 1. RAG 아키텍처 개요

Agent Framework는 **AI Context Providers**를 통해 RAG를 구현한다.

```
사용자 질문
    │
    ▼
┌──────────────────────────────────────┐
│            AIAgent                   │
│  ┌────────────────────────────────┐  │
│  │   AI Context Providers        │  │
│  │  ┌──────────────────────────┐ │  │
│  │  │ TextSearchProvider       │ │  │  ← 텍스트 검색 RAG
│  │  │ ChatHistoryMemoryProvider│ │  │  ← 대화 히스토리 메모리
│  │  │ Neo4jContextProvider     │ │  │  ← 그래프 RAG
│  │  │ CustomContextProvider    │ │  │  ← 커스텀 구현
│  │  └──────────────────────────┘ │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │   Hosted Tools (서비스 제공)    │  │
│  │  FileSearchToolDefinition     │  │  ← 파일 검색
│  │  WebSearchToolDefinition      │  │  ← 웹 검색 (Bing)
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
    │
    ▼
LLM 추론 → 응답
```

### RAG 접근 방식 선택 가이드

| 접근 방식 | 사용 시 | NuGet 패키지 |
|---|---|---|
| **TextSearchProvider** | 커스텀 검색 어댑터로 자체 데이터 소스 연동 | `Microsoft.Agents.AI` |
| **File Search (Hosted)** | Azure AI Foundry 벡터 스토어 문서 검색 | `Microsoft.Agents.AI.Foundry` |
| **Web Search (Hosted)** | Bing 기반 실시간 웹 검색 | `Microsoft.Agents.AI.Foundry` |
| **ChatHistoryMemoryProvider** | 대화 히스토리를 벡터 스토어에 저장/검색 | `Microsoft.Agents.AI` + VectorData |
| **Neo4j GraphRAG** | 지식 그래프 기반 관계형 검색 | `Neo4j.AgentFramework.GraphRAG` |
| **Custom AIContextProvider** | 완전한 커스텀 RAG 로직 | `Microsoft.Agents.AI` |

---

## 2. 필수 NuGet 패키지

```bash
# 핵심 (TextSearchProvider, AIContextProvider, ChatHistoryMemoryProvider)
dotnet add package Microsoft.Agents.AI --prerelease

# Hosted Tools (File Search, Web Search)
dotnet add package Microsoft.Agents.AI.Foundry --prerelease
dotnet add package Azure.AI.Projects --prerelease
dotnet add package Azure.Identity

# 벡터 스토어 (ChatHistoryMemoryProvider용)
dotnet add package Microsoft.Extensions.VectorData.Abstractions
dotnet add package Microsoft.SemanticKernel.Connectors.InMemory  # 인메모리 벡터 스토어

# Neo4j GraphRAG
dotnet add package Neo4j.AgentFramework.GraphRAG
dotnet add package Neo4j.Driver
```

---

## 3. TextSearchProvider (텍스트 검색 RAG)

### 3.1 기본 사용법

```csharp
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

// 검색 어댑터 함수 정의
static Task<IEnumerable<TextSearchProvider.TextSearchResult>> SearchAdapter(
    string query, CancellationToken cancellationToken)
{
    List<TextSearchProvider.TextSearchResult> results = new();

    if (query.Contains("return", StringComparison.OrdinalIgnoreCase))
    {
        results.Add(new()
        {
            SourceName = "Return Policy",
            SourceLink = "https://contoso.com/policies/returns",
            Text = "Customers may return any item within 30 days of delivery."
        });
    }

    return Task.FromResult<IEnumerable<TextSearchProvider.TextSearchResult>>(results);
}

// TextSearchProvider 설정
TextSearchProviderOptions textSearchOptions = new()
{
    SearchTime = TextSearchProviderOptions.TextSearchBehavior.BeforeAIInvoke,
    RecentMessageMemoryLimit = 6,
};

// 에이전트에 연결
AIAgent agent = azureOpenAIClient
    .GetChatClient(deploymentName)
    .AsAIAgent(new ChatClientAgentOptions
    {
        ChatOptions = new()
        {
            Instructions = "You are a helpful support specialist. Answer using provided context and cite sources."
        },
        AIContextProviders = [new TextSearchProvider(SearchAdapter, textSearchOptions)]
    });
```

### 3.2 TextSearchProviderOptions 전체 설정

| 옵션 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `SearchTime` | `TextSearchBehavior` | `BeforeAIInvoke` | 검색 시점 (BeforeAIInvoke / OnDemandFunctionCalling) |
| `FunctionToolName` | `string` | `"Search"` | 온디맨드 모드 시 도구 이름 |
| `FunctionToolDescription` | `string` | 기본 설명 | 온디맨드 모드 시 도구 설명 |
| `ContextPrompt` | `string` | `"## Additional Context..."` | 검색 결과 앞에 붙는 프롬프트 |
| `CitationsPrompt` | `string` | `"Include citations..."` | 인용 지시 |
| `ContextFormatter` | `Func<IList<TextSearchResult>, string>` | `null` | 커스텀 포맷터 (설정 시 ContextPrompt/CitationsPrompt 무시) |
| `RecentMessageMemoryLimit` | `int` | `0` | 검색 입력에 포함할 최근 메시지 수 |
| `RecentMessageRolesIncluded` | `List<ChatRole>` | `[ChatRole.User]` | 포함할 메시지 역할 |

### 3.3 TextSearchResult 속성

| 속성 | 타입 | 설명 |
|---|---|---|
| `Text` | `string` | 검색 결과 텍스트 |
| `SourceName` | `string` | 출처 문서명 |
| `SourceLink` | `string` | 출처 링크 |

---

## 4. File Search (Hosted Tool)

Azure AI Foundry 벡터 스토어에 업로드된 문서를 검색한다.

```csharp
using Azure.AI.Projects;
using Azure.Identity;
using Microsoft.Agents.AI;

AIAgent agent = new AIProjectClient(new Uri(endpoint), new DefaultAzureCredential())
    .AsAIAgent(
        model: deploymentName,
        instructions: "You are a helpful assistant that searches through files to find information.",
        tools: [new FileSearchToolDefinition(vectorStoreIds: ["<your-vector-store-id>"])]);

Console.WriteLine(await agent.RunAsync("What does the document say about today's weather?"));
```

> **참고**: File Search는 Responses API 또는 Foundry Agent에서만 지원된다 (Chat Completion API에서는 미지원).

---

## 5. Web Search (Hosted Tool)

Bing grounding 기반 실시간 웹 검색을 수행한다.

```csharp
using Azure.AI.Projects;
using Azure.Identity;
using Microsoft.Agents.AI;

AIAgent agent = new AIProjectClient(new Uri(endpoint), new DefaultAzureCredential())
    .AsAIAgent(
        model: deploymentName,
        instructions: "You are a helpful assistant that can search the web for current information.",
        tools: [new WebSearchToolDefinition()]);

Console.WriteLine(await agent.RunAsync("What is the current weather in Seattle?"));
```

> **참고**: Web Search는 Chat Completion API와 Responses API 모두에서 지원된다.

---

## 6. AIContextProvider (커스텀 Context Provider)

### 6.1 아키텍처

`AIContextProvider`는 에이전트의 **각 LLM 호출 전후에 실행**된다:
- **호출 전 (Invoking)**: 외부 소스에서 컨텍스트를 가져와 LLM 입력에 추가
- **호출 후 (Invoked)**: 새 메시지에서 데이터를 추출하여 저장

### 6.2 Simple 구현 (권장 시작점)

```csharp
internal sealed class SimpleServiceMemoryProvider : AIContextProvider
{
    private readonly ProviderSessionState<State> _sessionState;
    private readonly ServiceClient _client;

    public SimpleServiceMemoryProvider(ServiceClient client)
        : base(null, null)
    {
        this._sessionState = new ProviderSessionState<State>(
            _ => new State(), this.GetType().Name);
        this._client = client;
    }

    public override string StateKey => this._sessionState.StateKey;

    // 호출 전: 관련 컨텍스트 로드
    protected override ValueTask<AIContext> ProvideAIContextAsync(
        InvokingContext context, CancellationToken cancellationToken = default)
    {
        var state = this._sessionState.GetOrInitializeState(context.Session);

        if (state.MemoriesId == null)
            return new(new AIContext());

        var memories = this._client.LoadMemories(
            state.MemoriesId,
            string.Join("\n", context.AIContext.Messages?.Select(x => x.Text) ?? []));

        return new(new AIContext
        {
            Messages = [new ChatMessage(ChatRole.User,
                "Here are some memories: " + string.Join("\n", memories.Select(x => x.Text)))]
        });
    }

    // 호출 후: 새 데이터 저장
    protected override async ValueTask StoreAIContextAsync(
        InvokedContext context, CancellationToken cancellationToken = default)
    {
        var state = this._sessionState.GetOrInitializeState(context.Session);
        state.MemoriesId ??= this._client.CreateMemoryContainer();
        this._sessionState.SaveState(context.Session, state);

        await this._client.StoreMemoriesAsync(
            state.MemoriesId,
            context.RequestMessages.Concat(context.ResponseMessages ?? []),
            cancellationToken);
    }

    public class State
    {
        public string? MemoriesId { get; set; }
    }
}
```

### 6.3 Advanced 구현 (메시지 필터링 포함)

```csharp
internal sealed class AdvancedServiceMemoryProvider : AIContextProvider
{
    private readonly ProviderSessionState<State> _sessionState;
    private readonly ServiceClient _client;

    public AdvancedServiceMemoryProvider(ServiceClient client)
        : base(null, null)
    {
        this._sessionState = new ProviderSessionState<State>(
            _ => new State(), this.GetType().Name);
        this._client = client;
    }

    public override string StateKey => this._sessionState.StateKey;

    // Advanced: 전체 입력 메시지 제어 가능
    protected override async ValueTask<AIContext> InvokingCoreAsync(
        InvokingContext context, CancellationToken cancellationToken = default)
    {
        var state = this._sessionState.GetOrInitializeState(context.Session);

        if (state.MemoriesId == null)
            return new AIContext();

        // 외부 입력 메시지만 필터링하여 검색
        var filteredInputMessages = context.AIContext.Messages?
            .Where(m => m.GetAgentRequestMessageSourceType() == AgentRequestMessageSourceType.External);

        var memories = this._client.LoadMemories(
            state.MemoriesId,
            string.Join("\n", filteredInputMessages?.Select(x => x.Text) ?? []));

        // 메모리 메시지에 소스 스탬프
        var memoryMessages =
            new[] { new ChatMessage(ChatRole.User,
                "Here are some memories: " + string.Join("\n", memories.Select(x => x.Text))) }
            .Select(m => m.WithAgentRequestMessageSource(
                AgentRequestMessageSourceType.AIContextProvider, this.GetType().FullName!));

        return new AIContext
        {
            Instructions = context.AIContext.Instructions,
            Messages = context.AIContext.Messages.Concat(memoryMessages),
            Tools = context.AIContext.Tools
        };
    }

    // Advanced: 저장 시 입력 메시지 필터링
    protected override async ValueTask InvokedCoreAsync(
        InvokedContext context, CancellationToken cancellationToken = default)
    {
        if (context.InvokeException is not null)
            return;

        var state = this._sessionState.GetOrInitializeState(context.Session);
        state.MemoriesId ??= this._client.CreateMemoryContainer();
        this._sessionState.SaveState(context.Session, state);

        // 외부 입력만 필터링하여 저장 (중복 방지)
        var filteredRequestMessages = context.RequestMessages
            .Where(m => m.GetAgentRequestMessageSourceType() == AgentRequestMessageSourceType.External);

        await this._client.StoreMemoriesAsync(
            state.MemoriesId,
            filteredRequestMessages.Concat(context.ResponseMessages ?? []),
            cancellationToken);
    }

    public class State
    {
        public string? MemoriesId { get; set; }
    }
}
```

### 6.4 에이전트에 Context Provider 연결

```csharp
AIAgent agent = new OpenAIClient("<your_api_key>")
    .GetChatClient(modelName)
    .AsAIAgent(new ChatClientAgentOptions()
    {
        ChatOptions = new() { Instructions = "You are a helpful assistant." },
        AIContextProviders = [
            new MyCustomMemoryProvider(),
            new TextSearchProvider(SearchAdapter, textSearchOptions),
            // 여러 Provider를 동시에 사용 가능
        ],
    });
```

### 6.5 AIContextProvider Override 메서드 요약

**Simple (기본 흐름):**
| 메서드 | 시점 | 역할 |
|---|---|---|
| `ProvideAIContextAsync` | LLM 호출 전 | 관련 데이터 로드, AIContext 반환 |
| `StoreAIContextAsync` | LLM 호출 후 | 새 메시지에서 데이터 추출/저장 |

**Advanced (전체 제어):**
| 메서드 | 시점 | 역할 |
|---|---|---|
| `InvokingCoreAsync` | LLM 호출 전 | 전체 입력(Messages, Instructions, Tools) 수정 |
| `InvokedCoreAsync` | LLM 호출 후 | 전체 요청/응답 메시지 접근, 필터링 저장 |

> **중요**: `AIContextProvider` 인스턴스는 모든 세션에서 공유된다. 세션별 상태는 반드시 `ProviderSessionState<T>`를 통해 `AgentSession`에 저장해야 한다.

---

## 7. ChatHistoryMemoryProvider (대화 히스토리 메모리)

대화 메시지를 벡터 스토어에 임베딩하여 저장하고, 이후 대화에서 의미적 유사도 검색으로 관련 기억을 회수한다.

### 7.1 기본 사용법

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.VectorData;
using Microsoft.SemanticKernel.Connectors.InMemory;

// 벡터 스토어 생성 (임베딩 생성기 포함)
VectorStore vectorStore = new InMemoryVectorStore(new InMemoryVectorStoreOptions()
{
    EmbeddingGenerator = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
        .GetEmbeddingClient(embeddingDeploymentName)
        .AsIEmbeddingGenerator()
});

// ChatHistoryMemoryProvider로 에이전트 생성
AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(new ChatClientAgentOptions
    {
        ChatOptions = new() { Instructions = "You are a helpful assistant." },
        Name = "MemoryAgent",
        AIContextProviders = [new ChatHistoryMemoryProvider(
            vectorStore,
            collectionName: "chathistory",
            vectorDimensions: 3072,
            session => new ChatHistoryMemoryProvider.State(
                storageScope: new() { UserId = "user-123", SessionId = Guid.NewGuid().ToString() },
                searchScope: new() { UserId = "user-123" }))]
    });

// 세션 1: 정보 저장
AgentSession session = await agent.CreateSessionAsync();
Console.WriteLine(await agent.RunAsync("I prefer window seats on flights.", session));

// 세션 2: 이전 세션의 정보를 기억
AgentSession session2 = await agent.CreateSessionAsync();
Console.WriteLine(await agent.RunAsync("Book me a flight to Seattle.", session2));
// → "window seat" 선호를 기억하여 응답
```

### 7.2 Scope (범위) 설정

```csharp
// Storage Scope: 이 메시지가 저장될 범위
storageScope: new() { UserId = "user-123", SessionId = "session-456" }

// Search Scope: 이 범위에서 검색 (더 넓게 설정 가능)
searchScope: new() { UserId = "user-123" }  // 해당 사용자의 모든 세션에서 검색
```

| Scope 속성 | 설명 |
|---|---|
| `ApplicationId` | 특정 애플리케이션 범위 |
| `AgentId` | 특정 에이전트 범위 |
| `UserId` | 특정 사용자 범위 |
| `SessionId` | 특정 세션 범위 |

### 7.3 ChatHistoryMemoryProviderOptions

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `SearchTime` | `BeforeAIInvoke` | 메모리 검색 시점 (BeforeAIInvoke / OnDemandFunctionCalling) |
| `MaxResults` | `3` | 검색당 최대 결과 수 |
| `ContextPrompt` | `"## Memories\n..."` | 검색 결과 앞 프롬프트 |
| `FunctionToolName` | `"Search"` | 온디맨드 모드 도구 이름 |
| `FunctionToolDescription` | 기본 설명 | 온디맨드 모드 도구 설명 |
| `SearchInputMessageFilter` | External만 | 검색 쿼리 구성 시 메시지 필터 |
| `StorageInputRequestMessageFilter` | External만 | 저장 전 요청 메시지 필터 |
| `StorageInputResponseMessageFilter` | 필터 없음 | 저장 전 응답 메시지 필터 |
| `EnableSensitiveTelemetryData` | `false` | 민감 데이터 로그 출력 |

### 7.4 보안 주의사항

- **프롬프트 인젝션**: 벡터 스토어에서 가져온 메시지가 LLM 컨텍스트에 주입됨
- **PII/민감 데이터**: 대화 메시지가 벡터로 저장되므로 접근 제어 및 암호화 필요
- **온디맨드 검색 도구**: AI 모델이 검색 시점/내용을 제어하므로 신뢰 경계 주의
- **프로덕션에서 `Redactor` 사용**: `EnableSensitiveTelemetryData` false 유지

---

## 8. Neo4j GraphRAG

지식 그래프 기반 관계형 검색으로 엔티티 간 관계를 활용한 RAG를 제공한다.

### 8.1 기본 사용법

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using Neo4j.AgentFramework.GraphRAG;
using Neo4j.Driver;

// Neo4j 연결
await using var driver = GraphDatabase.Driver(
    neo4jUri, AuthTokens.Basic(username, password));

// 임베딩 생성기
var credential = new DefaultAzureCredential();
var azureClient = new AzureOpenAIClient(new Uri(azureEndpoint), credential);
IEmbeddingGenerator<string, Embedding<float>> embedder = azureClient
    .GetEmbeddingClient("text-embedding-3-small")
    .AsIEmbeddingGenerator();

// Neo4j Context Provider 생성
await using var provider = new Neo4jContextProvider(driver, new Neo4jContextProviderOptions
{
    IndexName = "chunkEmbeddings",
    IndexType = IndexType.Vector,       // Vector, Fulltext, Hybrid
    EmbeddingGenerator = embedder,
    TopK = 5,
    RetrievalQuery = """
        MATCH (node)-[:FROM_DOCUMENT]->(doc:Document)
        OPTIONAL MATCH (doc)<-[:FILED]-(company:Company)
        RETURN node.text AS text, score, doc.title AS title, company.name AS company
        ORDER BY score DESC
        """,
});

// 에이전트 생성
AIAgent agent = azureClient
    .GetChatClient("gpt-4o")
    .AsIChatClient()
    .AsBuilder()
    .UseAIContextProviders(provider)
    .BuildAIAgent(new ChatClientAgentOptions
    {
        ChatOptions = new ChatOptions
        {
            Instructions = "You are a financial analyst assistant.",
        },
    });

var session = await agent.CreateSessionAsync();
Console.WriteLine(await agent.RunAsync("What risks does Acme Corp face?", session));
```

### 8.2 Neo4jContextProviderOptions

| 옵션 | 설명 |
|---|---|
| `IndexName` | Neo4j 벡터/풀텍스트 인덱스 이름 |
| `IndexType` | `Vector`, `Fulltext`, `Hybrid` |
| `EmbeddingGenerator` | 임베딩 생성기 (Vector/Hybrid 모드에서 필수) |
| `TopK` | 반환할 최대 결과 수 |
| `RetrievalQuery` | 커스텀 Cypher 쿼리 (관계 탐색 가능) |

---

## 9. 지원 벡터 스토어 (ChatHistoryMemoryProvider용)

| 구현체 | NuGet |
|---|---|
| Azure AI Search | 공식 SDK |
| Cosmos DB (MongoDB vCore / NoSQL) | 공식 SDK |
| In-Memory | `Microsoft.SemanticKernel.Connectors.InMemory` |
| PostgreSQL / Neon | 공식 SDK |
| Qdrant | 공식 SDK |
| Redis | 공식 SDK |
| SQL Server / SQLite | 공식 SDK |
| MongoDB | 공식 SDK |
| Pinecone | Microsoft SDK |
| Elasticsearch | 공식 SDK |
| Weaviate | 공식 SDK |

---

## 10. 핵심 클래스/인터페이스 레퍼런스

| 클래스/인터페이스 | 설명 |
|---|---|
| `AIContextProvider` | Context Provider 베이스 클래스 |
| `AIContext` | Messages, Instructions, Tools 속성 |
| `InvokingContext` | LLM 호출 전 컨텍스트 |
| `InvokedContext` | LLM 호출 후 컨텍스트 |
| `ProviderSessionState<T>` | 세션 상태 관리 유틸리티 |
| `TextSearchProvider` | 텍스트 검색 RAG Provider |
| `TextSearchProvider.TextSearchResult` | 검색 결과 (Text, SourceName, SourceLink) |
| `TextSearchProviderOptions` | TextSearchProvider 설정 |
| `ChatHistoryMemoryProvider` | 대화 히스토리 메모리 Provider |
| `ChatHistoryMemoryProvider.State` | 메모리 상태 (storageScope, searchScope) |
| `ChatHistoryMemoryProviderScope` | Scope 정의 (ApplicationId, AgentId, UserId, SessionId) |
| `ChatHistoryMemoryProviderOptions` | 메모리 Provider 설정 |
| `FileSearchToolDefinition` | 파일 검색 Hosted Tool |
| `WebSearchToolDefinition` | 웹 검색 Hosted Tool |
| `Neo4jContextProvider` | Neo4j GraphRAG Provider |
| `Neo4jContextProviderOptions` | Neo4j 설정 |
| `IndexType` | enum (Vector, Fulltext, Hybrid) |
| `AgentRequestMessageSourceType` | enum (External, AIContextProvider) |
| `ChatClientAgentOptions` | 에이전트 옵션 (AIContextProviders 포함) |

---

## 참고 문서

- [RAG Overview](https://learn.microsoft.com/en-us/agent-framework/agents/rag?pivots=programming-language-csharp)
- [Context Providers](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/context-providers?pivots=programming-language-csharp)
- [File Search](https://learn.microsoft.com/en-us/agent-framework/agents/tools/file-search?pivots=programming-language-csharp)
- [Web Search](https://learn.microsoft.com/en-us/agent-framework/agents/tools/web-search?pivots=programming-language-csharp)
- [Neo4j GraphRAG](https://learn.microsoft.com/en-us/agent-framework/integrations/neo4j-graphrag?pivots=programming-language-csharp)
- [Chat History Memory](https://learn.microsoft.com/en-us/agent-framework/integrations/chat-history-memory-provider?pivots=programming-language-csharp)
- [GitHub 샘플](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/02-agents/AgentWithRAG)
