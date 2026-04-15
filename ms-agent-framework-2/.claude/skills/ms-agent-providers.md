# Microsoft Agent Framework - Provider별 에이전트 생성 가이드

Provider(추론 서비스)별 에이전트 생성 패턴을 안내하는 스킬. 각 Provider의 NuGet 패키지, 인증 방식, URL 형식, 지원 기능을 포함한다.

## 트리거 조건

- 사용자가 특정 Provider(OpenAI, Azure OpenAI, Anthropic, Foundry, Ollama 등)로 에이전트를 만들려 할 때
- Provider 선택을 고민하거나 비교를 요청할 때
- 인증 방식이나 엔드포인트 설정을 물어볼 때

---

## 1. Provider별 NuGet 패키지 및 URL

| AI 서비스 | SDK | NuGet 패키지 | URL 형식 |
|---|---|---|---|
| Foundry Models (Azure OpenAI SDK) | Azure OpenAI | `Azure.AI.OpenAI` | `https://ai-foundry-<resource>.services.ai.azure.com/` |
| Foundry Models (OpenAI SDK) | OpenAI | `OpenAI` | `https://ai-foundry-<resource>.services.ai.azure.com/openai/v1/` |
| Foundry Models (Inference SDK) | Azure AI Inference | `Azure.AI.Inference` | `https://ai-foundry-<resource>.services.ai.azure.com/models` |
| Foundry Agents | Azure AI Projects | `Azure.AI.Projects` + `Microsoft.Agents.AI.Foundry` | `https://ai-foundry-<resource>.services.ai.azure.com/api/projects/ai-project-<project>` |
| Azure OpenAI (Azure OpenAI SDK) | Azure OpenAI | `Azure.AI.OpenAI` | `https://<resource>.openai.azure.com/` |
| Azure OpenAI (OpenAI SDK) | OpenAI | `OpenAI` | `https://<resource>.openai.azure.com/openai/v1/` |
| OpenAI | OpenAI | `OpenAI` | URL 불필요 |
| Anthropic via Foundry | Anthropic Foundry | `Anthropic.Foundry` | Resource name 필요 |
| Anthropic (직접) | Anthropic | `Anthropic` | URL 불필요 |

---

## 2. Provider별 기능 지원 매트릭스

| 추론 서비스 | 서비스 채팅 기록 | InMemory/Custom 채팅 기록 |
|---|:---:|:---:|
| Microsoft Foundry Agent | O | X |
| Foundry Models ChatCompletion | X | O |
| Foundry Models Responses | O | O |
| Foundry Anthropic | X | O |
| Azure OpenAI ChatCompletion | X | O |
| Azure OpenAI Responses | O | O |
| Anthropic | X | O |
| OpenAI ChatCompletion | X | O |
| OpenAI Responses | O | O |

---

## 3. Provider별 에이전트 생성 코드

### 3.1 Azure AI Foundry (AIProjectClient) - 권장

가장 간단하고 통합된 방식. Azure AI Foundry 프로젝트와 직접 연동한다.

```csharp
dotnet add package Microsoft.Agents.AI.Foundry --prerelease
dotnet add package Azure.AI.Projects --prerelease
dotnet add package Azure.Identity
```

```csharp
using System;
using Azure.AI.Projects;
using Azure.Identity;
using Microsoft.Agents.AI;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME")
    ?? "gpt-4o-mini";

AIAgent agent = new AIProjectClient(new Uri(endpoint), new DefaultAzureCredential())
    .AsAIAgent(
        model: deploymentName,
        instructions: "You are a friendly assistant. Keep your answers brief.",
        name: "HelloAgent");
```

**환경 변수:**
- `AZURE_OPENAI_ENDPOINT` (필수) - Azure AI Foundry 프로젝트 엔드포인트
- `AZURE_OPENAI_DEPLOYMENT_NAME` (선택) - 모델 배포 이름

### 3.2 OpenAI (직접)

```bash
dotnet add package OpenAI
dotnet add package Microsoft.Agents.AI --prerelease
```

```csharp
using OpenAI;
using Microsoft.Agents.AI;

OpenAIClient client = new OpenAIClient(new ApiKeyCredential(apiKey));

AIAgent agent = client.AsAIAgent(
    model: "gpt-4o-mini",
    instructions: "You are a helpful assistant.",
    name: "OpenAIAgent");
```

### 3.3 Azure OpenAI (Azure OpenAI SDK + Azure 자격증명)

```bash
dotnet add package Azure.AI.OpenAI
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI --prerelease
```

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;

var clientOptions = new OpenAIClientOptions()
{
    Endpoint = new Uri("https://<resource>.openai.azure.com/")
};

// API Key 인증
OpenAIClient client = new OpenAIClient(
    new ApiKeyCredential(apiKey), clientOptions);

// 또는 Azure AD 인증
OpenAIClient client = new OpenAIClient(
    new BearerTokenPolicy(new DefaultAzureCredential(), "https://ai.azure.com/.default"),
    clientOptions);

AIAgent agent = client.AsAIAgent(
    model: "gpt-4o-mini",
    instructions: "You are a helpful assistant.",
    name: "AzureOpenAIAgent");
```

### 3.4 Azure OpenAI (OpenAI SDK 호환)

```bash
dotnet add package OpenAI
dotnet add package Microsoft.Agents.AI --prerelease
```

```csharp
using OpenAI;
using Microsoft.Agents.AI;

var clientOptions = new OpenAIClientOptions()
{
    Endpoint = new Uri("https://<resource>.openai.azure.com/openai/v1/")
};

OpenAIClient client = new OpenAIClient(
    new ApiKeyCredential(apiKey), clientOptions);

AIAgent agent = client.AsAIAgent(
    model: "gpt-4o-mini",
    instructions: "You are a helpful assistant.",
    name: "AzureViaOpenAISDK");
```

### 3.5 Anthropic (직접)

```bash
dotnet add package Anthropic
dotnet add package Microsoft.Agents.AI --prerelease
```

```csharp
using Anthropic;
using Microsoft.Agents.AI;

var client = new AnthropicClient() { ApiKey = apiKey };

AIAgent agent = client.AsAIAgent(
    model: "claude-sonnet-4-20250514",
    instructions: "You are a helpful assistant.",
    name: "ClaudeAgent");
```

### 3.6 Anthropic via Azure AI Foundry

```bash
dotnet add package Anthropic.Foundry
dotnet add package Microsoft.Agents.AI --prerelease
```

```csharp
using Anthropic.Foundry;
using Microsoft.Agents.AI;

var client = new AnthropicFoundryClient(
    new AnthropicFoundryApiKeyCredentials(apiKey, resourceName));

AIAgent agent = client.AsAIAgent(
    model: deploymentName,
    instructions: "You are a helpful assistant.",
    name: "FoundryClaudeAgent");
```

### 3.7 Foundry Models (Azure AI Inference SDK)

```bash
dotnet add package Azure.AI.Inference
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI --prerelease
```

```csharp
using Azure.AI.Inference;
using Azure.Identity;
using Microsoft.Agents.AI;

// Azure AI Inference를 통해 모델 접근
var endpoint = new Uri("https://ai-foundry-<resource>.services.ai.azure.com/models");
// IChatClient를 구성한 후 ChatClientAgent로 래핑
```

### 3.8 커스텀 IChatClient (Ollama 등)

`IChatClient`를 구현하는 모든 서비스를 사용할 수 있다.

```csharp
using Microsoft.Agents.AI;

// 임의의 IChatClient 구현체
IChatClient chatClient = GetMyChatClient(); // Ollama, LM Studio 등

var agent = new ChatClientAgent(chatClient,
    instructions: "You are a helpful assistant.");
```

---

## 4. Provider 선택 가이드

### Azure 환경이라면
- **Azure AI Foundry** → `AIProjectClient` + `Microsoft.Agents.AI.Foundry` (가장 통합적)
- **Azure OpenAI만** → `Azure.AI.OpenAI` 또는 `OpenAI` SDK

### 클라우드 비종속이라면
- **OpenAI** → `OpenAI` SDK
- **Anthropic** → `Anthropic` SDK
- **로컬 모델** → Ollama 등 `IChatClient` 구현체

### 멀티 Provider가 필요하다면
- `IChatClient` 추상화를 활용하여 런타임에 Provider를 교체한다
- 각 Provider의 `IChatClient` 구현체를 DI로 주입한다

---

## 5. 인증 모범 사례

```csharp
// 개발 환경: DefaultAzureCredential (편리하지만 프로덕션 비권장)
new DefaultAzureCredential()

// 프로덕션: 특정 자격증명 사용
new ManagedIdentityCredential()           // Azure Managed Identity
new ClientSecretCredential(tenant, client, secret)  // Service Principal
new AzureCliCredential()                  // 로컬 개발용
```

> **Warning**: `DefaultAzureCredential`은 개발에는 편리하지만 프로덕션에서는 지연 시간, 의도치 않은 자격증명 탐색, 보안 위험이 있다. 프로덕션에서는 `ManagedIdentityCredential` 같은 특정 자격증명을 사용한다.

---

## 참고 문서

- [Agent Types & Providers](https://learn.microsoft.com/en-us/agent-framework/agents/)
- [Your First Agent](https://learn.microsoft.com/en-us/agent-framework/get-started/your-first-agent?pivots=programming-language-csharp)
- [GitHub 샘플](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples)
