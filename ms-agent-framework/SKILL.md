---
name: ms-agent-framework
description: Microsoft Agent Framework(Microsoft.Agents.AI + Microsoft.Agents.AI.OpenAI)로 AI 에이전트를 구축할 때 사용하는 스킬. 에이전트 생성, Tool 정의, 세션 관리, 스트리밍 응답, 라이프사이클 관리 등 프로덕션 패턴을 안내한다. OpenRouter, Gemini 등 OpenAI 호환 엔드포인트를 사용하는 C# 프로젝트에 적용 가능.
---

# Microsoft Agent Framework 스킬

Microsoft.Agents.AI / Microsoft.Agents.AI.OpenAI 패키지를 사용하여 C# AI 에이전트를 구축하는 프로덕션 패턴 가이드.

## 이 스킬을 사용하는 상황

- 사용자가 Microsoft Agent Framework으로 AI 에이전트를 만들려고 할 때
- C#에서 OpenAI 호환 API(OpenRouter, Gemini 등)를 사용하는 에이전트를 구축할 때
- 에이전트에 Tool(Function Calling)을 등록하고 사용하려 할 때
- 멀티턴 대화, 세션 관리, 스트리밍 응답을 구현할 때
- 에이전트 라이프사이클(생성/타임아웃/정리)을 관리하려 할 때

## 1. 프로젝트 설정

### NuGet 패키지

```xml
<PackageReference Include="Microsoft.Agents.AI" Version="1.0.0" />
<PackageReference Include="Microsoft.Agents.AI.OpenAI" Version="1.0.0" />
```

### 필요한 using 문

```csharp
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.OpenAI;
using Microsoft.Extensions.AI;   // AIFunction, AIFunctionFactory, IChatClient 등
using OpenAI;                     // OpenAIClient, ApiKeyCredential
```

## 2. 에이전트 생성 패턴

### 2-1. 기본 에이전트 생성

```csharp
// 1) OpenAI 호환 클라이언트 생성 (OpenRouter, Gemini 등 가능)
var llmClient = new OpenAIClient(
    new ApiKeyCredential(apiKey),
    new OpenAIClientOptions { Endpoint = new Uri(endpoint) });

// 2) Tool 정의 (AIFunctionFactory로 메서드를 AIFunction으로 변환)
var toolInstance = new MyTools();
AIFunction[] tools =
[
    AIFunctionFactory.Create(toolInstance.SearchItems),
    AIFunctionFactory.Create(toolInstance.CreateItem),
    AIFunctionFactory.Create(toolInstance.DeleteItem),
];

// 3) AIAgent 생성 (ChatClient → AsAIAgent 확장 메서드)
AIAgent agent = llmClient
    .GetChatClient(modelName)       // 예: "google/gemini-2.5-flash-lite-preview"
    .AsAIAgent(
        instructions: systemPrompt, // 시스템 프롬프트 (string)
        tools: tools,               // AIFunction[] 배열
        name: "MyAgent"             // 에이전트 이름 (선택)
    );

// 4) 세션 생성 (대화 히스토리 관리)
var session = await AgentSession.CreateSessionAsync();
```

### 2-2. 비동기 팩토리 패턴 (권장)

`AgentSession.CreateSessionAsync()`가 async이므로 생성자에서 직접 호출할 수 없다.
static 팩토리 메서드 패턴을 사용한다.

```csharp
public sealed class MyAgent
{
    private readonly AIAgent _agent;
    private readonly AgentSession _session;

    private MyAgent(AIAgent agent, AgentSession session)
    {
        _agent = agent;
        _session = session;
    }

    public static async Task<MyAgent> CreateAsync(
        string systemPrompt, string apiKey, string endpoint, string model,
        AIFunction[] tools)
    {
        var client = new OpenAIClient(
            new ApiKeyCredential(apiKey),
            new OpenAIClientOptions { Endpoint = new Uri(endpoint) });

        var agent = client
            .GetChatClient(model)
            .AsAIAgent(instructions: systemPrompt, tools: tools);

        var session = await AgentSession.CreateSessionAsync();

        return new MyAgent(agent, session);
    }
}
```

## 3. 메시지 처리 패턴

### 3-1. 스트리밍 응답 (권장)

```csharp
public async Task<string> ProcessMessageAsync(string userMessage, CancellationToken ct = default)
{
    var sb = new StringBuilder();

    await foreach (var update in _agent.RunStreamingAsync(userMessage, _session, cancellationToken: ct))
    {
        sb.Append(update.Text);
    }

    return sb.ToString();
}
```

- `RunStreamingAsync`는 `IAsyncEnumerable<AgentResponseUpdate>`를 반환
- 프레임워크가 Tool 호출 → 결과 수집 → 재호출을 자동으로 처리
- `_session`이 대화 히스토리를 자동 유지 (멀티턴)

### 3-2. 동시성 제어 (멀티유저 환경)

같은 에이전트에 동시에 메시지가 들어오면 세션 상태가 꼬일 수 있다.
사용자별 SemaphoreSlim으로 직렬화한다.

```csharp
private readonly SemaphoreSlim _lock = new(1, 1);

public async Task<string> ProcessMessageAsync(string userMessage, CancellationToken ct = default)
{
    await _lock.WaitAsync(ct);
    try
    {
        var sb = new StringBuilder();
        await foreach (var update in _agent.RunStreamingAsync(userMessage, _session, cancellationToken: ct))
            sb.Append(update.Text);
        return sb.ToString();
    }
    finally
    {
        _lock.Release();
    }
}
```

## 4. Tool 정의 패턴

### 4-1. [Description] 어트리뷰트 기반 Tool 정의

`[Description]`이 LLM에게 전달되는 Tool 스키마가 된다. 모든 파라미터에 명확한 설명을 쓴다.

```csharp
public class MyTools
{
    [Description(
        "지정한 조건으로 아이템을 검색합니다. " +
        "결과는 최대 maxResults개까지 반환됩니다.")]
    public async Task<List<SearchResult>> SearchItems(
        [Description("검색 키워드")] string query,
        [Description("결과 최대 개수 (기본: 10)")] int maxResults = 10,
        [Description("카테고리 필터 (선택)")] string? category = null)
    {
        // 실제 검색 로직
        return await _apiClient.SearchAsync(query, maxResults, category);
    }

    [Description("새 아이템을 생성합니다. 성공 시 아이템 ID를 반환합니다.")]
    public async Task<CreateResult> CreateItem(
        [Description("아이템 이름")] string name,
        [Description("아이템 설명 (선택)")] string? description = null)
    {
        return await _apiClient.CreateAsync(name, description);
    }
}
```

### 4-2. Tool 등록 시 주의사항

- **인터페이스가 아닌 구체 클래스**의 메서드를 `AIFunctionFactory.Create()`에 전달해야 `[Description]`이 보존된다
- 반환 타입은 JSON 직렬화 가능한 클래스/레코드 사용
- `Task<T>` 또는 `ValueTask<T>` 반환 가능
- 파라미터의 기본값(`= null`, `= 10` 등)은 optional 파라미터로 LLM에 전달됨

### 4-3. Tool 결과 클래스 설계

```csharp
// Tool이 반환하는 결과 — LLM이 JSON으로 해석
public sealed class ToolResult
{
    public bool Success { get; init; }
    public string Message { get; init; } = "";
    public string? ItemId { get; init; }
}
```

`success` 필드를 포함하면 Tool 성공/실패를 코드에서 감지할 수 있다 (아래 로깅 래퍼 참조).

## 5. Tool 로깅 래퍼 패턴

Tool 호출을 감싸서 로깅 + 실패 추적을 한다.
`DelegatingAIFunction`을 상속하여 구현한다.

```csharp
using Microsoft.Extensions.AI;

internal sealed class LoggingAIFunction(
    AIFunction inner, ILogger logger, int[]? consecutiveFailures = null)
    : DelegatingAIFunction(inner)
{
    protected override async ValueTask<object?> InvokeCoreAsync(
        AIFunctionArguments arguments, CancellationToken cancellationToken)
    {
        logger.LogDebug("[Tool:{Name}] 호출 - args={Args}", Name, SerializeArgs(arguments));

        try
        {
            var result = await base.InvokeCoreAsync(arguments, cancellationToken);
            logger.LogDebug("[Tool:{Name}] 완료", Name);

            // success 필드로 연속 실패 카운터 관리
            if (consecutiveFailures is not null)
                UpdateFailureCounter(result, consecutiveFailures);

            return result;
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "[Tool:{Name}] 예외", Name);
            if (consecutiveFailures is not null)
                Interlocked.Increment(ref consecutiveFailures[0]);
            throw;
        }
    }

    private static void UpdateFailureCounter(object? result, int[] counter)
    {
        // result를 JSON으로 직렬화하여 success 필드 확인
        if (result is null) return;
        try
        {
            var json = JsonSerializer.Serialize(result);
            using var doc = JsonDocument.Parse(json);
            if (doc.RootElement.TryGetProperty("success", out var s) &&
                s.ValueKind == JsonValueKind.False)
            {
                Interlocked.Increment(ref counter[0]);
                return;
            }
        }
        catch { /* ignore */ }
        Interlocked.Exchange(ref counter[0], 0); // 성공이면 리셋
    }

    private static string SerializeArgs(AIFunctionArguments args) =>
        string.Join(", ", args.Select(kv => $"{kv.Key}={kv.Value}"));

    /// <summary>AIFunction 배열 전체를 래핑한다.</summary>
    public static AIFunction[] WrapAll(AIFunction[] functions, ILogger logger, int[]? failures = null) =>
        functions.Select(f => (AIFunction)new LoggingAIFunction(f, logger, failures)).ToArray();
}
```

사용:
```csharp
AIFunction[] rawTools = [ AIFunctionFactory.Create(tools.Search), ... ];
var consecutiveFailures = new int[1]; // 공유 카운터
AIFunction[] tools = LoggingAIFunction.WrapAll(rawTools, logger, consecutiveFailures);
// tools를 AsAIAgent()에 전달
```

## 6. 에이전트 라이프사이클 관리

### 6-1. 멀티유저 에이전트 매니저

```csharp
public sealed class AgentManager
{
    private readonly ConcurrentDictionary<string, MyAgent> _agents = new();
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _userLocks = new();
    private readonly ConcurrentDictionary<string, DateTime> _createdTimes = new();

    public async Task<string> HandleMessageAsync(string userId, string message)
    {
        // 사용자별 직렬화 (동시 메시지 방지)
        var userLock = _userLocks.GetOrAdd(userId, _ => new SemaphoreSlim(1, 1));
        await userLock.WaitAsync();
        try
        {
            if (!_agents.TryGetValue(userId, out var agent))
            {
                agent = await MyAgent.CreateAsync(systemPrompt, apiKey, endpoint, model, tools);
                _agents[userId] = agent;
                _createdTimes[userId] = DateTime.UtcNow;
            }
            return await agent.ProcessMessageAsync(message);
        }
        finally
        {
            userLock.Release();
        }
    }

    /// <summary>에이전트 종료 및 리소스 정리.</summary>
    public async Task RemoveAgentAsync(string userId)
    {
        if (_agents.TryRemove(userId, out var agent))
        {
            await agent.CleanupAsync();
            _createdTimes.TryRemove(userId, out _);
        }
    }

    /// <summary>타임아웃된 에이전트 ID 목록 반환.</summary>
    public List<string> GetExpiredUserIds(TimeSpan maxLifetime)
    {
        var now = DateTime.UtcNow;
        return _createdTimes
            .Where(kv => now - kv.Value > maxLifetime)
            .Select(kv => kv.Key)
            .ToList();
    }
}
```

### 6-2. 리소스 정리

AgentSession은 IAsyncDisposable을 구현할 수 있으므로 반드시 정리한다.

```csharp
public async Task CleanupAsync()
{
    if (_session is IAsyncDisposable disposable)
        await disposable.DisposeAsync();
}
```

### 6-3. 타임아웃 백그라운드 서비스

```csharp
public class AgentTimeoutService(AgentManager manager) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await Task.Delay(TimeSpan.FromMinutes(1), ct);

            var expired = manager.GetExpiredUserIds(TimeSpan.FromHours(8));
            foreach (var userId in expired)
                await manager.RemoveAgentAsync(userId);
        }
    }
}
```

## 7. 시스템 프롬프트 패턴

### 7-1. 런타임 변수 주입

프롬프트 파일에 플레이스홀더를 두고 런타임에 치환한다.

```markdown
# prompt.md
오늘 날짜: {{TODAY}}, 현재 시각: {{NOW}}
사용자 이름: {{USER_NAME}}
```

```csharp
private static string BuildInstructions(string basePrompt)
{
    var kst = TimeZoneInfo.FindSystemTimeZoneById("Korea Standard Time");
    var now = TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, kst);
    return basePrompt
        .Replace("{{TODAY}}", now.ToString("yyyy-MM-dd"))
        .Replace("{{NOW}}", now.ToString("HH:mm"));
}
```

### 7-2. 프롬프트 파일 로딩

마크다운의 코드 블록에서 시스템 프롬프트를 파싱하는 패턴:

```csharp
public static class PromptLoader
{
    /// <summary>
    /// prompt.md의 각 # 섹션 아래 코드 블록을 파싱하여 딕셔너리로 반환.
    /// 예: # SystemPrompt → 아래 ```markdown ... ``` 블록이 "SystemPrompt" 키의 값
    /// </summary>
    public static async Task<IReadOnlyDictionary<string, string>> LoadAsync(string path)
    {
        var content = await File.ReadAllTextAsync(path);
        var result = new Dictionary<string, string>();
        // 섹션 제목(# xxx)과 코드 블록(``` ... ```)을 파싱
        // ...
        return result;
    }
}
```

## 8. 대화 히스토리 관리 (IChatReducer)

장기 대화에서 토큰 제한, 비용, 응답 지연 문제를 해결하기 위해 `IChatReducer`를 사용하여 히스토리를 압축한다.
`IChatReducer`는 `Microsoft.Extensions.AI.Abstractions`에 정의된 인터페이스로, 메시지 목록을 받아 축소된 목록을 반환한다.

### 8-1. Truncation 전략 (오래된 메시지 삭제)

비용 제로, 하지만 삭제된 정보는 영구 소실.

```csharp
public class TruncationChatReducer(int maxMessages) : IChatReducer
{
    public Task<IEnumerable<ChatMessage>> ReduceAsync(
        IEnumerable<ChatMessage> messages,
        CancellationToken cancellationToken = default)
    {
        var list = messages.ToList();
        if (list.Count <= maxMessages)
            return Task.FromResult<IEnumerable<ChatMessage>>(list);

        // System 메시지는 항상 유지
        var systemMessages = list.Where(m => m.Role == ChatRole.System).ToList();
        var nonSystemMessages = list.Where(m => m.Role != ChatRole.System).ToList();

        // 최근 메시지만 유지
        var kept = nonSystemMessages
            .TakeLast(maxMessages - systemMessages.Count)
            .ToList();

        IEnumerable<ChatMessage> result = [.. systemMessages, .. kept];
        return Task.FromResult(result);
    }
}
```

### 8-2. Summarization 전략 (LLM으로 요약)

LLM 호출 비용이 들지만, 문맥을 유지하면서 메시지 수를 줄인다.

```csharp
public class SummarizationChatReducer(IChatClient chatClient, int keepLastMessages) : IChatReducer
{
    public async Task<IEnumerable<ChatMessage>> ReduceAsync(
        IEnumerable<ChatMessage> messages,
        CancellationToken cancellationToken = default)
    {
        var list = messages.ToList();
        if (list.Count <= keepLastMessages)
            return list;

        var systemMessages = list.Where(m => m.Role == ChatRole.System).ToList();
        var nonSystemMessages = list.Where(m => m.Role != ChatRole.System).ToList();

        var keepCount = keepLastMessages - systemMessages.Count;
        var toSummarize = nonSystemMessages.SkipLast(keepCount).ToList();
        var toKeep = nonSystemMessages.TakeLast(keepCount).ToList();

        if (toSummarize.Count == 0)
            return list;

        // LLM에 요약 요청
        var conversationText = string.Join("\n",
            toSummarize.Select(m => $"{m.Role}: {m.Text}"));
        List<ChatMessage> summaryRequest =
        [
            new(ChatRole.User, $"다음 대화를 간결하게 요약해 주세요:\n\n{conversationText}"),
        ];

        var response = await chatClient.GetResponseAsync(
            summaryRequest, cancellationToken: cancellationToken);
        var summaryMessage = new ChatMessage(
            ChatRole.Assistant, $"[요약] {response.Text}");

        return [.. systemMessages, summaryMessage, .. toKeep];
    }
}
```

### 8-3. 에이전트에 적용

`InMemoryChatHistoryProvider`에 reducer를 설정하여 에이전트에 연결한다.

```csharp
var reducer = new SummarizationChatReducer(chatClient, keepLastMessages: 5);

var historyProvider = new InMemoryChatHistoryProvider(
    new InMemoryChatHistoryProviderOptions
    {
        ChatReducer = reducer,
    });

AIAgent agent = llmClient
    .GetChatClient(modelName)
    .AsAIAgent(
        new ChatClientAgentOptions
        {
            Name = "MyAgent",
            ChatOptions = new ChatOptions
            {
                Instructions = systemPrompt,
            },
            ChatHistoryProvider = historyProvider,
        });
```

`ChatReducerTriggerEvent` 속성으로 실행 타이밍을 제어할 수 있다:
- `BeforeMessagesRetrieval` (기본값) — 메시지 조회 전에 축소
- `AfterMessageAdded` — 메시지 추가 후 축소

### 8-4. 전략 선택 가이드

| 전략 | 비용 | 문맥 유지 | 적합한 상황 |
|------|------|-----------|-------------|
| Truncation | 없음 | 낮음 (과거 정보 소실) | 짧은 대화, 비용 최소화 |
| Summarization | LLM 호출 발생 | 높음 (요약으로 보존) | 장기 대화, 문맥이 중요한 업무 |

## 9. 태그 기반 상태 전환

LLM 응답에서 특정 태그를 감지하여 서버 측 동작을 트리거하는 패턴.
State Machine 없이 AI가 대화 흐름을 판단하는 방식.

```csharp
// 프롬프트에서 AI에게 태그 출력을 지시
// 예: "작업이 완료되면 [TASK_DONE] 태그를 출력하세요"

// 서버에서 태그 감지
var response = await agent.ProcessMessageAsync(message);

if (response.Contains("[SHOW_BUTTONS]"))
{
    response = response.Replace("[SHOW_BUTTONS]", "");
    await SendButtonsAsync(userId);  // UI 버튼 표시
}
if (response.Contains("[TASK_DONE]"))
{
    response = response.Replace("[TASK_DONE]", "");
    await CompleteTaskAsync(userId);  // 작업 완료 처리
}
if (response.Contains("[SESSION_END]"))
{
    response = response.Replace("[SESSION_END]", "");
    await CleanupAgentAsync(userId);  // 세션 종료
}
```

장점: 코드에 State Machine을 만들지 않고, 프롬프트만으로 흐름 제어 가능.

## 10. DI 등록 패턴 (ASP.NET Core)

```csharp
// Program.cs

// LLM 설정
var apiKey = Environment.GetEnvironmentVariable("LLM_API_KEY")
    ?? throw new InvalidOperationException("LLM_API_KEY 환경변수 필요");
var endpoint = builder.Configuration["LLM:Endpoint"]!;
var model = builder.Configuration["LLM:Model"]!;

// 프롬프트 로딩
var prompts = await PromptLoader.LoadAsync("Prompts/prompt.md");
builder.Services.AddSingleton<IReadOnlyDictionary<string, string>>(prompts);

// 에이전트 매니저 등록
builder.Services.AddSingleton(sp => new AgentManager(
    new LlmOptions { ApiKey = apiKey, Endpoint = endpoint, Model = model },
    sp.GetRequiredService<IReadOnlyDictionary<string, string>>(),
    sp.GetRequiredService<ILogger<AgentManager>>()
));

// 타임아웃 서비스
builder.Services.AddHostedService<AgentTimeoutService>();
```

## 11. 에러 핸들링 체크리스트

| 상황 | 대처 |
|------|------|
| LLM API 호출 실패 | try-catch로 사용자 친화적 메시지 반환 |
| Tool 실행 예외 | DelegatingAIFunction에서 catch → 로깅 후 re-throw (프레임워크가 LLM에 에러 전달) |
| Tool 연속 실패 (5회 이상) | consecutiveFailures 카운터 → 세션 재시작 권고 |
| 동시 메시지 충돌 | 사용자별 SemaphoreSlim으로 직렬화 |
| 세션 타임아웃 | BackgroundService가 주기적으로 만료 검사 → 정리 |
| 서버 종료 | IHostApplicationLifetime.ApplicationStopping에서 활성 에이전트 정리 |
| AgentSession 릭 | CleanupAsync에서 IAsyncDisposable 확인 후 DisposeAsync |

## 12. appsettings.json 설정 예시

```jsonc
{
  "LLM": {
    "Endpoint": "https://openrouter.ai/api/v1",
    "Model": "google/gemini-2.5-flash-lite-preview"
  },
  "Agent": {
    "MaxLifetimeHours": 8,
    "MaxConcurrentAgents": 10,
    "IdleTimeoutMinutes": 10
  }
}
```

## 13. 핵심 클래스/인터페이스 요약

| 클래스 | 패키지 | 역할 |
|--------|--------|------|
| `OpenAIClient` | OpenAI | OpenAI 호환 API 클라이언트 |
| `IChatClient` | Microsoft.Extensions.AI | 채팅 클라이언트 인터페이스 |
| `AIAgent` | Microsoft.Agents.AI | 에이전트 (Tool + LLM 통합) |
| `AgentSession` | Microsoft.Agents.AI | 대화 세션 (히스토리 관리) |
| `AgentResponseUpdate` | Microsoft.Agents.AI | 스트리밍 응답 단위 |
| `AIFunction` | Microsoft.Extensions.AI | Tool 정의 |
| `AIFunctionFactory` | Microsoft.Extensions.AI | 메서드 → AIFunction 변환 |
| `AIFunctionArguments` | Microsoft.Extensions.AI | Tool 호출 인자 |
| `DelegatingAIFunction` | Microsoft.Extensions.AI | AIFunction 래핑 베이스 클래스 |
| `IChatReducer` | Microsoft.Extensions.AI.Abstractions | 대화 히스토리 축소 인터페이스 |
| `InMemoryChatHistoryProvider` | Microsoft.Agents.AI | 인메모리 대화 히스토리 관리 (ChatReducer 속성 지원) |
| `ChatClientAgentOptions` | Microsoft.Agents.AI | AsAIAgent 확장 설정 (ChatHistoryProvider 포함) |
| `AsAIAgent()` | Microsoft.Agents.AI.OpenAI | ChatClient → AIAgent 확장 메서드 |

## 참고

이 스킬의 패턴은 프로덕션 환경에서 검증된 것으로, 다음 구성에서 안정적으로 동작한다:
- OpenRouter API 경유 Gemini 모델
- 멀티유저 동시 접속 (ConcurrentDictionary + SemaphoreSlim)
- 에이전트 타임아웃 관리 (BackgroundService)
- 태그 기반 상태 전환 (State Machine 없이 프롬프트로 제어)
- Tool 로깅 래퍼 (DelegatingAIFunction)
- Graceful shutdown (서버 종료 시 활성 에이전트 정리)
