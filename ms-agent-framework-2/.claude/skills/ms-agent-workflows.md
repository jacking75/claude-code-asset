# Microsoft Agent Framework Workflows (C#) - 워크플로 구축 스킬

그래프 기반 워크플로를 구축할 때 사용하는 스킬. Executor, Edge, Events, State, Human-in-the-Loop, Checkpointing, Agent Orchestration(Sequential, Concurrent, Handoff, Group Chat) 패턴을 안내한다.

## 트리거 조건

다음 상황에서 이 스킬을 활성화한다:
- 코드에서 `Microsoft.Agents.AI.Workflows`, `WorkflowBuilder`, `Executor`, `AgentWorkflowBuilder` 등을 사용할 때
- 사용자가 "워크플로", "Workflow", "오케스트레이션", "멀티에이전트", "파이프라인" 등을 언급할 때
- 여러 에이전트를 조합하여 순차/병렬/핸드오프/그룹 채팅 패턴을 구현할 때

---

## 1. 아키텍처 개요

### Agent vs Workflow 선택

| Agent | Workflow |
|---|---|
| LLM이 동적으로 단계 결정 | 사전 정의된 작업 시퀀스 |
| 단일 LLM 호출(+도구)로 충분할 때 | 여러 에이전트/함수가 조율될 때 |
| 개방형, 대화형 작업 | 명시적 실행 흐름 제어 필요 |

### 핵심 기능

- **Type Safety** - 컴포넌트 간 메시지 타입 호환성 빌드 시 검증
- **Graph-Based Control Flow** - 조건부 라우팅, 병렬 처리, 동적 실행 경로
- **External Integration** - Request/Response 패턴, Human-in-the-Loop
- **Checkpointing** - 상태 저장/복구, 장기 실행 프로세스 재개
- **Multi-Agent Orchestration** - Sequential, Concurrent, Handoff, Group Chat 패턴 내장

### 실행 모델: Superstep (BSP - Bulk Synchronous Parallel)

```
Superstep N:
  1. 이전 superstep의 모든 대기 메시지 수집
  2. Edge 정의에 따라 타겟 Executor로 메시지 라우팅
  3. 모든 타겟 Executor 동시 실행
  4. 동기화 배리어 (모든 Executor 완료 대기)
  5. 새 메시지를 다음 superstep 큐에 적재
```

**보장사항**: 결정적 실행 순서, 안정적 체크포인팅, 레이스 컨디션 없음

---

## 2. 필수 NuGet 패키지

```bash
dotnet add package Microsoft.Agents.AI.Workflows --prerelease
dotnet add package Microsoft.Agents.AI.Foundry --prerelease  # Foundry 사용 시
dotnet add package Azure.AI.Projects --prerelease             # Azure AI Projects 사용 시
dotnet add package Azure.Identity                             # Azure 인증
```

---

## 3. Executor (처리 단위)

Executor는 워크플로의 기본 빌딩 블록으로, 메시지를 받아 처리하고 결과를 다음으로 전달한다.

### 3.1 기본 Executor (`[MessageHandler]` + `partial` + 소스 생성기)

```csharp
using Microsoft.Agents.AI.Workflows;

internal sealed partial class UppercaseExecutor() : Executor("UppercaseExecutor")
{
    [MessageHandler]
    private ValueTask<string> HandleAsync(string message, IWorkflowContext context)
    {
        string result = message.ToUpperInvariant();
        return ValueTask.FromResult(result); // 반환값이 자동으로 연결된 Executor에 전달
    }
}
```

### 3.2 수동 메시지 전송

```csharp
internal sealed partial class UppercaseExecutor() : Executor("UppercaseExecutor")
{
    [MessageHandler]
    private async ValueTask HandleAsync(string message, IWorkflowContext context)
    {
        string result = message.ToUpperInvariant();
        await context.SendMessageAsync(result); // 수동 전송
    }
}
```

### 3.3 다중 입력 타입 처리

```csharp
internal sealed partial class SampleExecutor() : Executor("SampleExecutor")
{
    [MessageHandler]
    private ValueTask<string> HandleStringAsync(string message, IWorkflowContext context)
        => ValueTask.FromResult(message.ToUpperInvariant());

    [MessageHandler]
    private ValueTask<int> HandleIntAsync(int message, IWorkflowContext context)
        => ValueTask.FromResult(message * 2);
}
```

### 3.4 함수 기반 Executor

```csharp
Func<string, string> uppercaseFunc = s => s.ToUpperInvariant();
var uppercase = uppercaseFunc.BindExecutor("UppercaseExecutor");
```

### 3.5 워크플로 출력 Executor

```csharp
internal sealed partial class OutputExecutor() : Executor("OutputExecutor")
{
    [MessageHandler]
    private async ValueTask HandleAsync(string message, IWorkflowContext context)
    {
        await context.YieldOutputAsync("Hello, World!"); // 워크플로 호출자에게 결과 반환
    }
}
```

### 3.6 부수 효과만 있는 Executor (void, 동기)

```csharp
internal sealed partial class LogExecutor() : Executor("LogExecutor")
{
    [MessageHandler]
    private void Handle(string message, IWorkflowContext context)
    {
        Console.WriteLine("Doing some work...");
    }
}
```

### IWorkflowContext 주요 메서드

| 메서드 | 설명 |
|---|---|
| `SendMessageAsync(msg)` | 연결된 Executor로 메시지 전송 |
| `YieldOutputAsync(data)` | 워크플로 출력 생성 (caller에게 반환) |
| `QueueStateUpdateAsync(key, value, scopeName)` | 공유 상태 저장 |
| `ReadStateAsync<T>(key, scopeName)` | 공�� 상태 읽기 |
| `AddEventAsync(event)` | 커스텀 이벤트 발행 |

---

## 4. Edge (연결)

### 4.1 Direct Edge (단순 연결)

```csharp
WorkflowBuilder builder = new(sourceExecutor);
builder.AddEdge(sourceExecutor, targetExecutor);
```

### 4.2 Conditional Edge (조건부 라우팅)

```csharp
private static Func<object?, bool> GetCondition(bool expectedResult) =>
    detectionResult => detectionResult is DetectionResult result && result.IsSpam == expectedResult;

var workflow = new WorkflowBuilder(spamDetectionExecutor)
    .AddEdge(spamDetectionExecutor, emailAssistantExecutor, condition: GetCondition(expectedResult: false))
    .AddEdge(emailAssistantExecutor, sendEmailExecutor)
    .AddEdge(spamDetectionExecutor, handleSpamExecutor, condition: GetCondition(expectedResult: true))
    .WithOutputFrom(handleSpamExecutor, sendEmailExecutor)
    .Build();
```

### 4.3 타입별 조건부 Edge

```csharp
.AddEdge<AnalysisResult>(
    emailAnalysisExecutor,
    databaseAccessExecutor,
    condition: analysisResult => analysisResult?.EmailLength <= 500)
```

### 4.4 Switch-Case Edge (다중 분��)

```csharp
public enum SpamDecision { NotSpam, Spam, Uncertain }

builder.AddSwitch(spamDetectionExecutor, switchBuilder =>
    switchBuilder
        .AddCase(GetCondition(SpamDecision.NotSpam), emailAssistantExecutor)
        .AddCase(GetCondition(SpamDecision.Spam), handleSpamExecutor)
        .WithDefault(handleUncertainExecutor)
);
```

### 4.5 Fan-Out Edge (병렬 분배)

```csharp
builder.AddFanOutEdge(
    emailAnalysisExecutor,
    targets: [handleSpamExecutor, emailAssistantExecutor, emailSummaryExecutor, handleUncertainExecutor],
    targetSelector: (analysisResult, targetCount) =>
    {
        if (analysisResult is AnalysisResult r)
        {
            if (r.SpamDecision == SpamDecision.Spam) return [0];
            if (r.SpamDecision == SpamDecision.NotSpam)
            {
                List<int> targets = [1];
                if (r.EmailLength > 1000) targets.Add(2);
                return targets;
            }
            return [3];
        }
        throw new ArgumentException("Invalid result.");
    });
```

### 4.6 Fan-In Edge (집계)

```csharp
builder.AddFanInBarrierEdge(
    sources: [worker1, worker2, worker3],
    target: aggregatorExecutor);
```

### Edge 타입 요약

| 타입 | 메서드 | 용도 |
|---|---|---|
| Direct | `AddEdge(src, tgt)` | 단순 1:1 연결 |
| Conditional | `AddEdge(src, tgt, condition:)` | if/else 라우팅 |
| Switch-Case | `AddSwitch(src, switchBuilder)` | 다중 분기 |
| Fan-Out | `AddFanOutEdge(src, targets, selector)` | 병렬 분배 |
| Fan-In | `AddFanInBarrierEdge(sources, target)` | 결과 집계 |

---

## 5. Events (이벤트)

### 5.1 내장 이벤트 타입

| 이벤트 | 설명 |
|---|---|
| `WorkflowStartedEvent` | 워크플로 시작 |
| `WorkflowOutputEvent` | 워크플로 출력 |
| `WorkflowErrorEvent` | 워크플로 오류 |
| `WorkflowWarningEvent` | 워크플로 경고 |
| `ExecutorInvokedEvent` | Executor 처리 시작 |
| `ExecutorCompletedEvent` | Executor 처리 완료 |
| `ExecutorFailedEvent` | Executor 오류 |
| `AgentResponseEvent` | 에이전트 응답 (비스트리밍) |
| `AgentResponseUpdateEvent` | 에이전트 스트리밍 업데이트 |
| `SuperStepStartedEvent` | Superstep 시작 |
| `SuperStepCompletedEvent` | Superstep ���료 |
| `RequestInfoEvent` | 요청 발생 (HITL) |

### 5.2 이벤트 소비

```csharp
await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    switch (evt)
    {
        case ExecutorInvokedEvent invoke:
            Console.WriteLine($"Starting {invoke.ExecutorId}");
            break;
        case ExecutorCompletedEvent complete:
            Console.WriteLine($"Completed {complete.ExecutorId}: {complete.Data}");
            break;
        case WorkflowOutputEvent output:
            Console.WriteLine($"Workflow output: {output.Data}");
            return;
        case WorkflowErrorEvent error:
            Console.WriteLine($"Workflow error: {error.Exception}");
            return;
    }
}
```

### 5.3 커스텀 이벤트

```csharp
// 정의
internal sealed class ProgressEvent(string step) : WorkflowEvent(step) { }

// Executor 내에서 발행
await context.AddEventAsync(new ProgressEvent("Validating input"));

// 소비
case ProgressEvent progress:
    Console.WriteLine($"Progress: {progress.Data}");
    break;
```

---

## 6. Workflow Builder & Execution

### 6.1 워크플로 빌드

```csharp
using Microsoft.Agents.AI.Workflows;

var processor = new DataProcessor();
var validator = new Validator();
var formatter = new Formatter();

WorkflowBuilder builder = new(processor); // 시작 Executor 지정
builder.AddEdge(processor, validator);
builder.AddEdge(validator, formatter);
builder.WithOutputFrom(formatter);        // 출력 Executor 지정
var workflow = builder.Build();
```

### 6.2 Streaming 실행

```csharp
StreamingRun run = await InProcessExecution.RunStreamingAsync(workflow, inputMessage);
await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    if (evt is ExecutorCompletedEvent executorComplete)
        Console.WriteLine($"{executorComplete.ExecutorId}: {executorComplete.Data}");
    if (evt is WorkflowOutputEvent outputEvt)
        Console.WriteLine($"Workflow completed: {outputEvt.Data}");
}
```

### 6.3 Non-Streaming 실행

```csharp
Run result = await InProcessExecution.RunAsync(workflow, inputMessage);
foreach (WorkflowEvent evt in result.NewEvents)
{
    if (evt is WorkflowOutputEvent outputEvt)
        Console.WriteLine($"Final result: {outputEvt.Data}");
}
```

### 빌드 시 자동 검증

- **타입 호환성** - 연결된 Executor 간 메시지 타입
- **그래프 연결성** - 시작 Executor에서 모든 Executor 도달 가능
- **Executor 바인딩** - 모든 Executor 인스턴스화 확인
- **Edge 유효성** - 중복 및 무효 연결 검사

---

## 7. State (공유 상태)

### 7.1 상태 쓰기

```csharp
internal sealed partial class FileReadExecutor() : Executor("FileReadExecutor")
{
    [MessageHandler]
    private async ValueTask<string> HandleAsync(string message, IWorkflowContext context)
    {
        string fileContent = File.ReadAllText(message);
        string fileID = Guid.NewGuid().ToString("N");
        await context.QueueStateUpdateAsync(fileID, fileContent, scopeName: "FileContent");
        return fileID;
    }
}
```

### 7.2 상��� 읽기

```csharp
internal sealed partial class WordCountingExecutor() : Executor("WordCountingExecutor")
{
    [MessageHandler]
    private async ValueTask<int> HandleAsync(string message, IWorkflowContext context)
    {
        var fileContent = await context.ReadStateAsync<string>(message, scopeName: "FileContent")
            ?? throw new InvalidOperationException("File content state not found");
        return fileContent.Split([' ', '\n', '\r'], StringSplitOptions.RemoveEmptyEntries).Length;
    }
}
```

### 7.3 상태 격리 주의사항

- 빌드된 `Workflow`는 **불변(immutable)** 이지만, Executor 인스턴스는 공유될 수 있음
- 각 태스크/요청마다 Builder에서 **새 Workflow 인스턴스**를 생성하는 것을 권장
- Executor 공유가 필요하면 `IResettableExecutor` 구현 필요

### 7.4 IResettableExecutor

```csharp
internal sealed partial class AggregationExecutor()
    : Executor("AggregationExecutor"), IResettableExecutor
{
    private readonly List<string> _messages = [];

    [MessageHandler]
    private async ValueTask HandleAsync(string message, IWorkflowContext context)
    {
        this._messages.Add(message);
    }

    public ValueTask ResetAsync()
    {
        this._messages.Clear();
        return default;
    }
}
```

---

## 8. Human-in-the-Loop (HITL)

### 8.1 RequestPort 패턴

```csharp
// RequestPort 생성 (요청: NumberSignal, 응답: int)
var numberRequestPort = RequestPort.Create<NumberSignal, int>("GuessNumber");

JudgeExecutor judgeExecutor = new(42);
var workflow = new WorkflowBuilder(numberRequestPort)
    .AddEdge(numberRequestPort, judgeExecutor)
    .AddEdge(judgeExecutor, numberRequestPort)
    .WithOutputFrom(judgeExecutor)
    .Build();
```

### 8.2 JudgeExecutor 예제

```csharp
internal enum NumberSignal { Init, Above, Below }

internal sealed class JudgeExecutor() : Executor<int>("Judge")
{
    private readonly int _targetNumber;
    private int _tries;

    public JudgeExecutor(int targetNumber) : this()
    {
        this._targetNumber = targetNumber;
    }

    public override async ValueTask HandleAsync(int message, IWorkflowContext context, CancellationToken cancellationToken = default)
    {
        this._tries++;
        if (message == this._targetNumber)
            await context.YieldOutputAsync($"{this._targetNumber} found in {this._tries} tries!", cancellationToken);
        else if (message < this._targetNumber)
            await context.SendMessageAsync(NumberSignal.Below, cancellationToken: cancellationToken);
        else
            await context.SendMessageAsync(NumberSignal.Above, cancellationToken: cancellationToken);
    }
}
```

### 8.3 외부 응답 처리

```csharp
await using StreamingRun handle = await InProcessExecution.RunStreamingAsync(workflow, NumberSignal.Init);
await foreach (WorkflowEvent evt in handle.WatchStreamAsync())
{
    switch (evt)
    {
        case RequestInfoEvent requestInputEvt:
            int guess = GetUserInput(); // 외부 시스템/사람에게서 응답
            await handle.SendResponseAsync(requestInputEvt.Request.CreateResponse(guess));
            break;
        case WorkflowOutputEvent outputEvt:
            Console.WriteLine($"Workflow completed: {outputEvt.Data}");
            return;
    }
}
```

### 8.4 Tool Approval (에이전트 오케스트레이션 HITL)

```csharp
// 승인 필요 도구 래핑
var tools = new AITool[]
{
    AIFunctionFactory.Create(CheckStagingStatus),
    new ApprovalRequiredAIFunction(AIFunctionFactory.Create(DeployToProduction))
};

// 승인 요청 처리
await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    if (evt is RequestInfoEvent e &&
        e.Request.TryGetDataAs(out ToolApprovalRequestContent? approvalRequest))
    {
        await run.SendResponseAsync(
            e.Request.CreateResponse(approvalRequest.CreateResponse(approved: true)));
    }
}
```

---

## 9. Checkpointing (체크포인트)

### 9.1 체크포인트 캡처

```csharp
CheckpointManager checkpointManager = CheckpointManager.CreateInMemory();

StreamingRun run = await InProcessExecution.RunStreamingAsync(workflow, input, checkpointManager);
await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    if (evt is SuperStepCompletedEvent superStepEvt)
    {
        CheckpointInfo? checkpoint = superStepEvt.CompletionInfo?.Checkpoint;
    }
}

// Run에서 직접 접근
IReadOnlyList<CheckpointInfo> checkpoints = run.Checkpoints;
```

### 9.2 체크포인트에서 복원 (같은 Run)

```csharp
CheckpointInfo savedCheckpoint = run.Checkpoints[5];
await run.RestoreCheckpointAsync(savedCheckpoint);
await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    // 복원된 시점부터 재실행
}
```

### 9.3 체크포인트에서 재수화 (새 Run)

```csharp
StreamingRun newRun = await InProcessExecution.ResumeStreamingAsync(
    newWorkflow, savedCheckpoint, checkpointManager);
```

### 9.4 Executor 상태 저장/복원

```csharp
internal sealed partial class CustomExecutor() : Executor("CustomExecutor")
{
    private const string StateKey = "CustomExecutorState";
    private List<string> messages = new();

    [MessageHandler]
    private async ValueTask HandleAsync(string message, IWorkflowContext context)
    {
        this.messages.Add(message);
    }

    protected override ValueTask OnCheckpointingAsync(IWorkflowContext context, CancellationToken cancellation = default)
    {
        return context.QueueStateUpdateAsync(StateKey, this.messages);
    }

    protected override async ValueTask OnCheckpointRestoredAsync(IWorkflowContext context, CancellationToken cancellation = default)
    {
        this.messages = await context.ReadStateAsync<List<string>>(StateKey);
    }
}
```

---

## 10. Agent Orchestration 패턴

### 10.1 Sequential (순차 실행)

```csharp
static ChatClientAgent GetTranslationAgent(string targetLanguage, IChatClient chatClient) =>
    new(chatClient,
        $"You are a translation assistant who only responds in {targetLanguage}. " +
        $"Respond to any input by outputting the name of the input language and then translating the input to {targetLanguage}.");

var translationAgents = from lang in (string[])["French", "Spanish", "English"]
                        select GetTranslationAgent(lang, client);

var workflow = AgentWorkflowBuilder.BuildSequential(translationAgents);

var messages = new List<ChatMessage> { new(ChatRole.User, "Hello, world!") };
await using StreamingRun run = await InProcessExecution.RunStreamingAsync(workflow, messages);
await run.TrySendMessageAsync(new TurnToken(emitEvents: true));

await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    if (evt is AgentResponseUpdateEvent e)
        Console.Write(e.Update.Text);
    else if (evt is WorkflowOutputEvent outputEvt)
    {
        var result = outputEvt.As<List<ChatMessage>>()!;
        break;
    }
}
```

### 10.2 Concurrent (병렬 실행)

```csharp
var workflow = AgentWorkflowBuilder.BuildConcurrent(translationAgents);

// 실행 방식은 Sequential과 동일
// 모든 에이전트가 동일 입력에 대해 병렬 실행, 결과 자동 수집
```

### 10.3 Handoff (위임)

```csharp
ChatClientAgent historyTutor = new(client,
    "You provide assistance with historical queries...",
    "history_tutor",
    "Specialist agent for historical questions");

ChatClientAgent mathTutor = new(client,
    "You provide help with math problems...",
    "math_tutor",
    "Specialist agent for math questions");

ChatClientAgent triageAgent = new(client,
    "You determine which agent to use based on the user's homework question. ALWAYS handoff to another agent.",
    "triage_agent",
    "Routes messages to the appropriate specialist agent");

var workflow = AgentWorkflowBuilder.CreateHandoffBuilderWith(triageAgent)
    .WithHandoffs(triageAgent, [mathTutor, historyTutor])         // triage → 전문가
    .WithHandoffs([mathTutor, historyTutor], triageAgent)         // 전문가 → triage (복귀)
    .Build();

// 멀티턴 대화
List<ChatMessage> messages = new();
while (true)
{
    string userInput = Console.ReadLine()!;
    messages.Add(new(ChatRole.User, userInput));

    await using StreamingRun run = await InProcessExecution.RunStreamingAsync(workflow, messages);
    await run.TrySendMessageAsync(new TurnToken(emitEvents: true));

    List<ChatMessage> newMessages = new();
    await foreach (WorkflowEvent evt in run.WatchStreamAsync())
    {
        if (evt is AgentResponseUpdateEvent e)
            Console.Write(e.Update.Text);
        else if (evt is WorkflowOutputEvent outputEvt)
        {
            newMessages = outputEvt.As<List<ChatMessage>>()!;
            break;
        }
    }
    messages.AddRange(newMessages.Skip(messages.Count));
}
```

### 10.4 Group Chat (그룹 채팅)

```csharp
ChatClientAgent writer = new(client,
    "You are a creative copywriter...", "CopyWriter", "A creative copywriter agent");
ChatClientAgent reviewer = new(client,
    "You are a marketing reviewer...", "Reviewer", "A marketing review agent");

// Round-Robin 방식
var workflow = AgentWorkflowBuilder
    .CreateGroupChatBuilderWith(agents =>
        new RoundRobinGroupChatManager(agents)
        {
            MaximumIterationCount = 5
        })
    .AddParticipants(writer, reviewer)
    .Build();

var messages = new List<ChatMessage> { new(ChatRole.User, "Create a slogan for an eco-friendly EV.") };
await using StreamingRun run = await InProcessExecution.RunStreamingAsync(workflow, messages);
await run.TrySendMessageAsync(new TurnToken(emitEvents: true));

await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    if (evt is AgentResponseUpdateEvent update)
    {
        AgentResponse response = update.AsResponse();
        foreach (ChatMessage message in response.Messages)
            Console.WriteLine($"[{update.ExecutorId}]: {message.Text}");
    }
    else if (evt is WorkflowOutputEvent output)
    {
        var conversationHistory = output.As<List<ChatMessage>>();
        break;
    }
}
```

### 10.5 커스텀 Group Chat Manager

```csharp
public class ApprovalBasedManager : RoundRobinGroupChatManager
{
    private readonly string _approverName;

    public ApprovalBasedManager(IReadOnlyList<AIAgent> agents, string approverName)
        : base(agents)
    {
        _approverName = approverName;
    }

    protected override ValueTask<bool> ShouldTerminateAsync(
        IReadOnlyList<ChatMessage> history,
        CancellationToken cancellationToken = default)
    {
        var last = history.LastOrDefault();
        bool shouldTerminate = last?.AuthorName == _approverName &&
            last.Text?.Contains("approve", StringComparison.OrdinalIgnoreCase) == true;
        return ValueTask.FromResult(shouldTerminate);
    }
}
```

### 오케스트레이션 패턴 선택 가이드

| 패턴 | Builder 메서드 | 토폴로지 | 용도 |
|---|---|---|---|
| Sequential | `BuildSequential()` | 파이프라인 | 순차 처리, 번역 체인 |
| Concurrent | `BuildConcurrent()` | Fan-out + Aggregation | 병렬 다관점 분석 |
| Handoff | `CreateHandoffBuilderWith()` | 메시 (직접 연결) | 동적 위임, 트리아지 |
| Group Chat | `CreateGroupChatBuilderWith()` | 스타 (오케스트레이터) | 반복 개선, 협업 |
| Custom | `new WorkflowBuilder()` | 임의 DAG | 조건부/비즈니스 로직 |

> **참고**: Magentic Orchestration은 C#에서 아직 지원되지 않음 (Python만 지원)

---

## 11. Agents in Workflows

### TurnToken 패턴

에이전트를 워크플로에 배치하면, 메시지를 캐시한 후 `TurnToken`을 받아야 처리를 시작한다.

```csharp
await using StreamingRun run = await InProcessExecution.RunStreamingAsync(
    workflow, new ChatMessage(ChatRole.User, "Hello World!"));

// TurnToken을 보내야 에이전트가 처리 시작
await run.TrySendMessageAsync(new TurnToken(emitEvents: true));
```

### Agent Executor 옵션 (명시적 설정)

```csharp
var options = new AIAgentHostOptions
{
    EmitAgentUpdateEvents = true,      // 스트리밍 업데이트 이벤트 발행
    EmitAgentResponseEvents = true,    // 집계된 에이전트 응답 발행
    ReassignOtherAgentsAsUsers = true, // 다른 에이전트 메시지를 User 역할로 변환
    ForwardIncomingMessages = true,    // 수신 메시지를 다운스트림으로 전달
};

ExecutorBinding writerBinding = writerAgent.BindAsExecutor(options);
var workflow = new WorkflowBuilder(writerBinding)
    .AddEdge(writerBinding, reviewerAgent)
    .Build();
```

---

## 12. 핵심 클래스/인터페이스 레퍼런스

### 핵심 클래스

| 클래스 | 설명 |
|---|---|
| `Executor` / `Executor<T>` / `Executor<TIn, TOut>` | 워크플로 처리 단위 |
| `WorkflowBuilder` | 워크플로 그래프 구성 빌더 |
| `AgentWorkflowBuilder` | 에이전트 오케스트레이션 빌더 |
| `InProcessExecution` | 워크플로 실행 엔진 |
| `StreamingRun` | 스트리밍 실행 핸들 |
| `Run` | 비스트리밍 실행 결과 |
| `RequestPort` | HITL Request/Response 채널 |
| `CheckpointManager` | 체크포인트 관리자 |
| `CheckpointInfo` | 체크포인트 정보 |
| `TurnToken` | 에이전트 처리 시작 트리거 |
| `AIAgentHostOptions` | 에이전트 Executor 설정 |
| `ExecutorBinding` | Executor 바인딩 |
| `RoundRobinGroupChatManager` | 라운드 로빈 화자 선택 |
| `ApprovalRequiredAIFunction` | 승인 필요 도구 래퍼 |
| `ToolApprovalRequestContent` | 도구 승인 요청 데이터 |

### 인터페이스

| 인터페이스 | ���명 |
|---|---|
| `IWorkflowContext` | Executor 내 워크플로 상호작용 |
| `IResettableExecutor` | Executor 상태 리셋 |

### 어트리뷰트

| 어트리뷰트 | 설명 |
|---|---|
| `[MessageHandler]` | Executor 메시지 핸들러 (소스 생성기 연동) |

---

## 참고 문서

- [Workflows Overview](https://learn.microsoft.com/en-us/agent-framework/workflows/)
- [Executors](https://learn.microsoft.com/en-us/agent-framework/workflows/executors/)
- [Edges](https://learn.microsoft.com/en-us/agent-framework/workflows/edges/)
- [Events](https://learn.microsoft.com/en-us/agent-framework/workflows/events/)
- [Workflow Builder](https://learn.microsoft.com/en-us/agent-framework/workflows/workflows/)
- [State](https://learn.microsoft.com/en-us/agent-framework/workflows/state/)
- [Human-in-the-Loop](https://learn.microsoft.com/en-us/agent-framework/workflows/human-in-the-loop/)
- [Checkpoints](https://learn.microsoft.com/en-us/agent-framework/workflows/checkpoints/)
- [Agents in Workflows](https://learn.microsoft.com/en-us/agent-framework/workflows/agents-in-workflows/)
- [Orchestrations](https://learn.microsoft.com/en-us/agent-framework/workflows/orchestrations/)
- [GitHub 샘플](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/03-workflows)
