# Microsoft Agent Framework Skills, MCP, A2A (C#)

Agent Skills(포터블 지식 패키지), MCP(Model Context Protocol) 도구 연동, A2A(Agent-to-Agent) 프로토콜 호스팅 패턴을 안내하는 스킬.

## 트리거 조건

다음 상황에서 이 스킬을 활성화한다:
- 코드에서 `AgentSkillsProvider`, `AgentInlineSkill`, `AgentClassSkill`, `MCPToolDefinition`, `McpClientFactory`, `MapA2A` 등을 사용할 때
- 사용자가 "스킬", "Skills", "MCP", "A2A", "원격 에이전트", "에이전트 간 통신" 등을 언급할 때
- 에이전트에 포터블 지식 패키지를 추가하거나, MCP 서버를 연동하거나, A2A 엔드포인트를 호스팅하려 할 때

---

## 1. Agent Skills 개요

### Skills vs Tools 비교

| 구분 | Tool | Skill |
|---|---|---|
| **제공 내용** | 단일 호출 가능 액션 | 지침 + 참조 자료 + 선택적 스크립트 |
| **사용 방식** | 액션 필요 시 호출 | 관련 태스크 시 로드, 지침 읽고 참조 |
| **컨텍스트 비용** | 도구 스키마가 항상 프롬프트에 포함 | 이름/설명(~100 토큰)만 포함, 나머지 온디맨드 |
| **이식성** | 등록한 에이전트에 종속 | 호환 에이전트 어디서든 사용 가능 |
| **최적 용도** | 개별 액션 (DB 쿼리, 이메일 전송) | 도메인 전문성 (정책, 가이드라인) |

> 핵심 비유: **Tools = 동사** (search, book, validate), **Skills = 전문지식** (정책 지식, 코드 리뷰 가이드)

### Progressive Disclosure (4단계)

1. **Advertise** (~100 tokens) - 시스템 프롬프트에 이름/설명 주입
2. **Load** (< 5000 tokens) - `load_skill` 도구로 전체 SKILL.md 로드
3. **Read resources** - `read_skill_resource` 도구로 보충 파일 로드
4. **Run scripts** - `run_skill_script` 도구로 번들 스크립트 실행

### SKILL.md 디렉토리 구조

```
expense-report/
├── SKILL.md                    # Required — frontmatter + instructions
├── scripts/
│   └── validate.py             # 실행 가능한 스크립트
├── references/
│   └── POLICY_FAQ.md           # 참조 문서 (온디맨드 로드)
└── assets/
    └── template.md             # 템플릿/정적 리소스
```

### SKILL.md Frontmatter

```yaml
---
name: expense-report
description: File and validate employee expense reports according to company policy.
license: Apache-2.0
compatibility: Requires python3
metadata:
  author: contoso-finance
  version: "2.1"
---
```

| 필드 | 필수 | 설명 |
|---|---|---|
| `name` | Yes | 최대 64자, 소문자/숫자/하이픈만, 부모 디렉토리명과 일치 |
| `description` | Yes | 최대 1024자 |
| `license` | No | 라이선스 |
| `compatibility` | No | 환경 요구사항 |
| `metadata` | No | 임의 key-value |
| `allowed-tools` | No | 사전 승인 도구 목록 (Experimental) |

---

## 2. Skills 구현 패턴

### 2.1 File-based Skills

```csharp
using Microsoft.Agents.AI;

// 디렉토리에서 자동 탐색
var skillsProvider = new AgentSkillsProvider(
    Path.Combine(AppContext.BaseDirectory, "skills"));

// 복수 디렉토리
var skillsProvider = new AgentSkillsProvider([
    Path.Combine(AppContext.BaseDirectory, "company-skills"),
    Path.Combine(AppContext.BaseDirectory, "team-skills"),
]);

// 에이전트에 연결
AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
    .GetResponsesClient()
    .AsAIAgent(new ChatClientAgentOptions
    {
        Name = "SkillsAgent",
        ChatOptions = new() { Instructions = "You are a helpful assistant." },
        AIContextProviders = [skillsProvider],
    },
    model: deploymentName);
```

### 2.2 리소스/스크립트 탐색 커스터마이즈

```csharp
var fileOptions = new AgentFileSkillsSourceOptions
{
    AllowedResourceExtensions = [".md", ".txt"],
    ResourceDirectories = ["docs", "templates"],
    AllowedScriptExtensions = [".py"],
    ScriptDirectories = ["scripts", "tools"],
};

var skillsProvider = new AgentSkillsProvider(
    Path.Combine(AppContext.BaseDirectory, "skills"),
    fileOptions: fileOptions);
```

### 2.3 Script 실행 활성화

```csharp
var skillsProvider = new AgentSkillsProvider(
    Path.Combine(AppContext.BaseDirectory, "skills"),
    SubprocessScriptRunner.RunAsync);
```

> **보안 경고**: `SubprocessScriptRunner`는 데모 전용. 프로덕션에서는 샌드박싱, 리소스 제한, 입력 검증, 감사 로깅 필요.

### 2.4 Code-defined Skills (AgentInlineSkill)

```csharp
var codeStyleSkill = new AgentInlineSkill(
    name: "code-style",
    description: "Coding style guidelines and conventions for the team",
    instructions: """
        Use this skill when answering questions about coding style.
        1. Read the style-guide resource for the full set of rules.
        2. Answer based on those rules.
        """)
    .AddResource("style-guide", """
        # Team Coding Style Guide
        - Use 4-space indentation (no tabs)
        - Maximum line length: 120 characters
        - Use type annotations on all public methods
        """);

var skillsProvider = new AgentSkillsProvider(codeStyleSkill);
```

### 2.5 Dynamic Resources

```csharp
var projectInfoSkill = new AgentInlineSkill(
    name: "project-info",
    description: "Project status and configuration information",
    instructions: "Use this skill for questions about the current project.")
    .AddResource("environment", () =>
    {
        string env = Environment.GetEnvironmentVariable("APP_ENV") ?? "development";
        string region = Environment.GetEnvironmentVariable("APP_REGION") ?? "us-east-1";
        return $"Environment: {env}, Region: {region}";
    })
    .AddResource("team-roster", "Alice Chen (Tech Lead), Bob Smith (Backend Engineer)");
```

### 2.6 Code-defined Scripts

```csharp
using System.Text.Json;

var unitConverterSkill = new AgentInlineSkill(
    name: "unit-converter",
    description: "Convert between common units using a conversion factor",
    instructions: """
        1. Review the conversion-table resource to find the correct factor.
        2. Use the convert script, passing the value and factor.
        3. Present the result clearly with both units.
        """)
    .AddResource("conversion-table", """
        | From       | To         | Factor   |
        |------------|------------|----------|
        | miles      | kilometers | 1.60934  |
        | pounds     | kilograms  | 0.453592 |
        """)
    .AddScript("convert", (double value, double factor) =>
    {
        double result = Math.Round(value * factor, 4);
        return JsonSerializer.Serialize(new { value, factor, result });
    });
```

### 2.7 Class-based Skills (AgentClassSkill\<T\>)

```csharp
using System.ComponentModel;
using System.Text.Json;
using Microsoft.Agents.AI;

internal sealed class UnitConverterSkill : AgentClassSkill<UnitConverterSkill>
{
    public override AgentSkillFrontmatter Frontmatter { get; } = new(
        "unit-converter",
        "Convert between common units using a multiplication factor.");

    protected override string Instructions => """
        1. Review the conversion-table resource to find the correct factor.
        2. Use the convert script, passing the value and factor.
        3. Present the result clearly with both units.
        """;

    [AgentSkillResource("conversion-table")]
    [Description("Lookup table of multiplication factors for common unit conversions.")]
    public string ConversionTable => """
        | From       | To         | Factor   |
        |------------|------------|----------|
        | miles      | kilometers | 1.60934  |
        | pounds     | kilograms  | 0.453592 |
        """;

    [AgentSkillScript("convert")]
    [Description("Multiplies a value by a conversion factor and returns the result as JSON.")]
    private static string ConvertUnits(double value, double factor)
    {
        double result = Math.Round(value * factor, 4);
        return JsonSerializer.Serialize(new { value, factor, result });
    }
}

// 등록
var skillsProvider = new AgentSkillsProvider(new UnitConverterSkill());
```

### 2.8 Builder (혼합 소스 조합)

```csharp
var skillsProvider = new AgentSkillsProviderBuilder()
    .UseFileSkill(Path.Combine(AppContext.BaseDirectory, "skills"))
    .UseSkill(volumeConverterSkill)       // AgentInlineSkill
    .UseSkill(temperatureConverter)        // AgentClassSkill
    .UseFileScriptRunner(SubprocessScriptRunner.RunAsync)
    .Build();
```

### 2.9 Builder 필터링

```csharp
var approvedSkillNames = new HashSet<string> { "expense-report", "code-style" };

var skillsProvider = new AgentSkillsProviderBuilder()
    .UseFileSkill(Path.Combine(AppContext.BaseDirectory, "skills"))
    .UseFilter(skill => approvedSkillNames.Contains(skill.Frontmatter.Name))
    .Build();
```

### 2.10 Script Approval (인간 승인)

```csharp
var skillsProvider = new AgentSkillsProvider(
    skillPath: Path.Combine(AppContext.BaseDirectory, "skills"),
    options: new AgentSkillsProviderOptions { ScriptApproval = true });

// 또는 Builder
var skillsProvider = new AgentSkillsProviderBuilder()
    .UseFileSkill(Path.Combine(AppContext.BaseDirectory, "skills"))
    .UseScriptApproval(true)
    .Build();
```

### 2.11 DI (Dependency Injection)

```csharp
// DI 설정
ServiceCollection services = new();
services.AddSingleton<ConversionService>();
IServiceProvider serviceProvider = services.BuildServiceProvider();

// 에이전트에 서비스 프로바이더 전달
AIAgent agent = azureOpenAIClient.GetResponsesClient()
    .AsAIAgent(
        options: new ChatClientAgentOptions
        {
            Name = "ConverterAgent",
            ChatOptions = new() { Instructions = "You are a helpful assistant." },
            AIContextProviders = [skillsProvider],
        },
        model: deploymentName,
        services: serviceProvider);

// InlineSkill에서 DI 사용
var distanceSkill = new AgentInlineSkill(
    name: "distance-converter",
    description: "Convert between distance units.",
    instructions: "...")
    .AddResource("distance-table", (IServiceProvider sp) =>
        sp.GetRequiredService<ConversionService>().GetDistanceTable())
    .AddScript("convert", (double value, double factor, IServiceProvider sp) =>
        sp.GetRequiredService<ConversionService>().Convert(value, factor));

// ClassSkill에서 DI 사용 - 메서드 파라미터에 IServiceProvider 추가
[AgentSkillResource("weight-table")]
private static string GetWeightTable(IServiceProvider serviceProvider) =>
    serviceProvider.GetRequiredService<ConversionService>().GetWeightTable();
```

### 2.12 커스텀 시스템 프롬프트

```csharp
var skillsProvider = new AgentSkillsProvider(
    skillPath: Path.Combine(AppContext.BaseDirectory, "skills"),
    options: new AgentSkillsProviderOptions
    {
        SkillsInstructionPrompt = """
            You have skills available. Here they are:
            {skills}
            {resource_instructions}
            {script_instructions}
            """
    });
```

> 필수 플레이스홀더: `{skills}`, `{resource_instructions}`, `{script_instructions}`

---

## 3. MCP (Model Context Protocol) 도구

### 3.1 Hosted MCP Tools (Foundry 관리)

```csharp
using Azure.AI.Projects;
using Azure.Identity;
using Microsoft.Agents.AI;

// MCP Tool 정의
var mcpTool = new MCPToolDefinition(
    serverLabel: "microsoft_learn",
    serverUrl: "https://learn.microsoft.com/api/mcp");
mcpTool.AllowedTools.Add("microsoft_docs_search");

// 서버 측 에이전트 생성
var aiProjectClient = new AIProjectClient(new Uri(endpoint), new DefaultAzureCredential());
var agentVersion = await aiProjectClient.AgentAdministrationClient.CreateAgentVersionAsync(
    "AgentName",
    new ProjectsAgentVersionCreationOptions(
        new DeclarativeAgentDefinition(model)
        {
            Instructions = "You are a helpful assistant.",
            Tools = { mcpTool }
        }));

AIAgent agent = aiProjectClient.AsAIAgent(agentVersion);

// Tool Approval 설정
var runOptions = new ChatClientAgentRunOptions()
{
    ChatOptions = new()
    {
        RawRepresentationFactory = (_) => new ThreadAndRunOptions()
        {
            ToolResources = new MCPToolResource(serverLabel: "microsoft_learn")
            {
                RequireApproval = new MCPApproval("never"),  // "never", "always"
            }.ToToolResources()
        }
    }
};

// 실행
AgentSession session = await agent.CreateSessionAsync();
var response = await agent.RunAsync("Search Azure AI docs about MCP", session, runOptions);
```

### 3.2 Local MCP Tools (MCP C# SDK)

```bash
dotnet add package ModelContextProtocol --prerelease
```

```csharp
using ModelContextProtocol.Client;

// MCP 클라이언트 연결
await using var mcpClient = await McpClientFactory.CreateAsync(new StdioClientTransport(new()
{
    Name = "MCPServer",
    Command = "npx",
    Arguments = ["-y", "--verbose", "@modelcontextprotocol/server-github"],
}));

// MCP 도구 목록 가져오기
var mcpTools = await mcpClient.ListToolsAsync();

// 에이전트에 MCP 도구 제공
AIAgent agent = new AIProjectClient(new Uri(endpoint), new DefaultAzureCredential())
     .AsAIAgent(
         model: deploymentName,
         instructions: "You answer questions related to GitHub repositories.",
         tools: [.. mcpTools.Cast<AITool>()]);

Console.WriteLine(await agent.RunAsync("Summarize the last four commits to microsoft/semantic-kernel?"));
```

### 3.3 Agent를 MCP Server로 노출

```csharp
using Microsoft.Agents.AI;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ModelContextProtocol.Server;

AIAgent agent = new AIProjectClient(new Uri(endpoint), new DefaultAzureCredential())
    .AsAIAgent(model: "gpt-4o-mini", instructions: "You are good at telling jokes.", name: "Joker");

McpServerTool tool = McpServerTool.Create(agent.AsAIFunction());

HostApplicationBuilder builder = Host.CreateEmptyApplicationBuilder(settings: null);
builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithTools([tool]);

await builder.Build().RunAsync();
```

```bash
dotnet add package Microsoft.Extensions.Hosting --prerelease
dotnet add package ModelContextProtocol --prerelease
```

---

## 4. A2A (Agent-to-Agent) 프로토콜

### 4.1 A2A 사용 시점

| 시나리오 | 설명 |
|---|---|
| 서비스 경계 | 마이크로서비스 간 에이전트 통신 |
| 팀 경계 | 다른 팀 에이전트 사용 (코드 접근 불가) |
| 조직 경계 | 서드파티 에이전트 연동 |
| 독립적 진화 | 에이전트별 다른 릴리스 주기/언어 |

### 4.2 NuGet 패키지

```bash
dotnet add package Microsoft.Agents.AI.Hosting.A2A.AspNetCore --prerelease
dotnet add package Azure.AI.Projects --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.Foundry --prerelease
dotnet add package Microsoft.AspNetCore.OpenApi
dotnet add package Swashbuckle.AspNetCore
```

### 4.3 A2A 서버 전체 예제 (Program.cs)

```csharp
using A2A.AspNetCore;
using Azure.AI.Projects;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Hosting;
using Microsoft.Extensions.AI;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();
builder.Services.AddSwaggerGen();

string endpoint = builder.Configuration["AZURE_OPENAI_ENDPOINT"]
    ?? throw new InvalidOperationException("AZURE_OPENAI_ENDPOINT is not set.");
string deploymentName = builder.Configuration["AZURE_OPENAI_DEPLOYMENT_NAME"]
    ?? throw new InvalidOperationException("AZURE_OPENAI_DEPLOYMENT_NAME is not set.");

IChatClient chatClient = new AIProjectClient(
        new Uri(endpoint), new DefaultAzureCredential())
    .GetProjectOpenAIClient()
    .GetProjectResponsesClient()
    .AsIChatClient(deploymentName);

builder.Services.AddSingleton(chatClient);

// 에이전트 등록
var pirateAgent = builder.AddAIAgent("pirate", instructions: "You are a pirate. Speak like a pirate.");

var app = builder.Build();

app.MapOpenApi();
app.UseSwagger();
app.UseSwaggerUI();

// A2A 엔드포인트 매핑
app.MapA2A(pirateAgent, path: "/a2a/pirate", agentCard: new()
{
    Name = "Pirate Agent",
    Description = "An agent that speaks like a pirate.",
    Version = "1.0"
});

app.Run();
```

### 4.4 복수 에이전트 A2A 노출

```csharp
var mathAgent = builder.AddAIAgent("math", instructions: "You are a math expert.");
var scienceAgent = builder.AddAIAgent("science", instructions: "You are a science expert.");

app.MapA2A(mathAgent, "/a2a/math");
app.MapA2A(scienceAgent, "/a2a/science");
```

### 4.5 AgentCard 설정

```csharp
app.MapA2A(agent, "/a2a/my-agent", agentCard: new()
{
    Name = "My Agent",
    Description = "A helpful agent that assists with tasks.",
    Version = "1.0",
});
```

| AgentCard 속성 | 설명 |
|---|---|
| `Name` | 에이전트 이름 |
| `Description` | 에이전트 설명 |
| `Version` | 버전 |
| `Url` | 자동 할당 |
| `Capabilities` | 에이전트 기능 |

### 4.6 A2A 엔드포인트

```
GET  /a2a/{agent}/v1/card           # AgentCard 조회
POST /a2a/{agent}/v1/message        # 메시지 전송
POST /a2a/{agent}/v1/message:stream # 스트리밍 메시지
```

---

## 5. 핵심 클래스/인터페이스 레퍼런스

### Skills

| 클래스 | 설명 |
|---|---|
| `AgentSkillsProvider` | 스킬 → 에이전트 연결 (AIContextProvider 구현) |
| `AgentSkillsProviderBuilder` | 복합 소스 빌더 |
| `AgentSkillsProviderOptions` | 옵션 (ScriptApproval, SkillsInstructionPrompt, DisableCaching) |
| `AgentInlineSkill` | 코드 정의 인라인 스킬 |
| `AgentClassSkill<T>` | 클래스 기반 스킬 추상 베이스 |
| `AgentSkillFrontmatter` | name, description 등 메타데이터 |
| `AgentFileSkillsSourceOptions` | 파일 탐색 옵션 |
| `[AgentSkillResource]` | 리소스 어트리뷰트 |
| `[AgentSkillScript]` | 스크립트 어트리뷰트 |
| `SubprocessScriptRunner` | 서브프로세스 스크립트 실행기 (데모용) |

### MCP

| 클래스 | 설명 |
|---|---|
| `MCPToolDefinition` | Hosted MCP 도구 정의 (serverLabel, serverUrl) |
| `MCPToolResource` | MCP 도구 리소스 (RequireApproval) |
| `MCPApproval` | 승인 모드 ("never", "always") |
| `McpClientFactory` | Local MCP 클라이언트 팩토리 |
| `StdioClientTransport` | stdio 전송 |
| `McpServerTool` | MCP 서버 도구 래퍼 |

### A2A

| 클래스/메서드 | 설명 |
|---|---|
| `builder.AddAIAgent()` | 에이전트 DI 등록 |
| `app.MapA2A()` | A2A 엔드포인트 매핑 |
| `AgentCard` | 에이전트 메타데이터 |

---

## 6. NuGet 패키지 요약

| 패키지 | 용도 |
|---|---|
| `Microsoft.Agents.AI` | 핵심 (AgentSkillsProvider 등) |
| `Microsoft.Agents.AI.Foundry` | Foundry (Hosted MCP) |
| `Microsoft.Agents.AI.Hosting.A2A.AspNetCore` | A2A ASP.NET Core |
| `Azure.AI.Projects` | Azure AI Projects |
| `Azure.Identity` | Azure 인증 |
| `ModelContextProtocol` | MCP C# SDK |
| `Microsoft.Extensions.Hosting` | MCP 서버 호스팅 |
| `Microsoft.AspNetCore.OpenApi` | OpenAPI |
| `Swashbuckle.AspNetCore` | Swagger UI |

---

## 참고 문서

- [Agent Skills](https://learn.microsoft.com/en-us/agent-framework/agents/skills?pivots=programming-language-csharp)
- [Hosted MCP Tools](https://learn.microsoft.com/en-us/agent-framework/agents/tools/hosted-mcp-tools?pivots=programming-language-csharp)
- [Local MCP Tools](https://learn.microsoft.com/en-us/agent-framework/agents/tools/local-mcp-tools?pivots=programming-language-csharp)
- [A2A Integration](https://learn.microsoft.com/en-us/agent-framework/integrations/a2a?pivots=programming-language-csharp)
- [AgentSkills.io](https://agentskills.io/)
- [GitHub 샘플](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples)
