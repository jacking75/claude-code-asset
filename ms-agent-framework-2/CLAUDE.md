# Microsoft Agent Framework - Claude Code 스킬 & 에이전트 패키지

이 프로젝트는 **Microsoft Agent Framework (C#)** 로 AI 에이전트를 개발할 때 Claude Code가 참조하는 스킬 문서와 subagent 모음이다.

## 사용 방법

다른 C# 프로젝트에서 Microsoft Agent Framework를 사용할 때, `.claude/` 폴더를 해당 프로젝트로 복사하면 Claude Code가 자동으로 참조한다.

```bash
# 예: 대상 프로젝트에 전체 복사 (스킬 + subagent + 설정)
cp -r ./.claude/ /path/to/your-project/.claude/
```

## 포함된 스킬 (Skills)

| 파일 | 호출 방법 | 내용 |
|---|---|---|
| `.claude/skills/ms-agent-framework.md` | `/ms-agent` | 메인 스킬 - 에이전트 생성, Tools, 세션, 스트리밍, 미들웨어, Agent-as-Tool 등 전체 패턴 |
| `.claude/skills/ms-agent-providers.md` | `/ms-agent` 포함 | Provider별 가이드 - OpenAI, Azure OpenAI, Anthropic, Foundry 등 Provider별 설정 |
| `.claude/skills/ms-agent-workflows.md` | `/ms-workflow` | Workflows 스킬 - Executor, Edge, Events, State, HITL, Checkpoint, Agent Orchestration 패턴 |
| `.claude/skills/ms-agent-rag.md` | `/ms-rag` | RAG 스킬 - TextSearchProvider, File/Web Search, AIContextProvider, ChatHistoryMemoryProvider, Neo4j GraphRAG |
| `.claude/skills/ms-agent-skills-mcp-a2a.md` | `/ms-skills` `/mcp` `/a2a` | Skills/MCP/A2A 스킬 - Agent Skills 패키징, MCP 도구 연동, A2A 프로토콜 호스팅 |

## 포함된 Subagent (Agents)

| 파일 | 용도 | 사용 시나리오 |
|---|---|---|
| `.claude/agents/ms-agent-builder.md` | 에이전트 코드 생성 | 새 에이전트 프로젝트 스캐폴딩, Provider 설정, Tool/미들웨어 구현 |
| `.claude/agents/ms-agent-reviewer.md` | 에이전트 코드 리뷰 | 프레임워크 패턴 준수, 보안, 세션 관리, 프로덕션 체크리스트 점검 |
| `.claude/agents/ms-agent-migrator.md` | 마이그레이션 | Semantic Kernel/AutoGen → MS Agent Framework 코드 변환 |
| `.claude/agents/ms-workflow-builder.md` | 워크플로 설계/생성 | 멀티에이전트 오케스트레이션, 그래프 기반 워크플로, HITL, Checkpoint 구현 |
| `.claude/agents/ms-rag-builder.md` | RAG 설계/생성 | 검색 증강 생성, 문서 검색, 벡터 검색, 대화 메모리, 그래프 RAG 구현 |
| `.claude/agents/ms-skills-mcp-a2a-builder.md` | Skills/MCP/A2A | Agent Skills 패키징, MCP 도구 연동, A2A 프로토콜 호스팅 구현 |

### Subagent 사용 예시

Claude Code에서 Agent 도구를 통해 호출할 수 있다:

- **"에이전트 프로젝트 만들어줘"** → `ms-agent-builder`
- **"이 에이전트 코드 리뷰해줘"** → `ms-agent-reviewer`
- **"Semantic Kernel 코드를 마이그레이션해줘"** → `ms-agent-migrator`
- **"멀티에이전트 워크플로 만들어줘"** → `ms-workflow-builder`
- **"에이전트에 RAG 추가해줘"** → `ms-rag-builder`
- **"에이전트에 스킬 추가해줘"** → `ms-skills-mcp-a2a-builder`
- **"MCP 서버 연동해줘"** → `ms-skills-mcp-a2a-builder`
- **"A2A로 에이전트 노출해줘"** → `ms-skills-mcp-a2a-builder`

## 대상 프레임워크

- **Microsoft Agent Framework** (`Microsoft.Agents.AI`, `Microsoft.Agents.AI.Workflows`)
- **Microsoft.Extensions.AI** 추상화 기반
- C# / .NET 8+
- NuGet 패키지: `Microsoft.Agents.AI`, `Microsoft.Agents.AI.Workflows`, `Microsoft.Agents.AI.Foundry`, `Azure.AI.Projects`, `ModelContextProtocol`, `Microsoft.Agents.AI.Hosting.A2A.AspNetCore` 등 (prerelease)

## 참고

- 공식 문서: https://learn.microsoft.com/en-us/agent-framework/overview/?pivots=programming-language-csharp
- Workflows: https://learn.microsoft.com/en-us/agent-framework/workflows/
- RAG: https://learn.microsoft.com/en-us/agent-framework/agents/rag?pivots=programming-language-csharp
- Skills: https://learn.microsoft.com/en-us/agent-framework/agents/skills?pivots=programming-language-csharp
- A2A: https://learn.microsoft.com/en-us/agent-framework/integrations/a2a?pivots=programming-language-csharp
- AgentSkills.io: https://agentskills.io/
- GitHub 샘플: https://github.com/microsoft/agent-framework/tree/main/dotnet/samples
