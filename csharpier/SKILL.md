---
name: csharpier
description: C# 코드를 CSharpier로 포맷하거나 CSharpier 자체를 설정·운영해야 할 때 항상 사용한다. 트리거 예시는 "csharpier", "C# 포맷터", "C# 코드 정리", ".cs 파일 포맷팅", "CSharpier 설치/설정", "csharpier format/check 실행", ".csharpierrc / .csharpierignore 작성", "CSharpier MSBuild 통합", "CSharpier pre-commit hook", "CI에서 csharpier --check 실패" 등이다. 사용자가 ".NET / C# 프로젝트의 코드 스타일을 통일해달라"거나 "이 솔루션을 포맷해달라"고 말하면 명시적으로 CSharpier를 언급하지 않아도 이 스킬을 사용한다. Prettier, Black, gofmt, clang-format 같은 다른 언어 포맷터 또는 단순 리팩터링·네이밍 변경처럼 화이트스페이스/스타일과 무관한 작업에는 사용하지 않는다.
---

# CSharpier Skill

CSharpier는 C#용 opinionated 코드 포맷터다. JavaScript의 Prettier, Python의 Black과 같은 위치를 가지며, 라인 길이만 기준으로 일관된 레이아웃을 생성한다. 이 스킬은 Claude가 다음을 잘 처리하도록 돕는다.

- 솔루션/프로젝트에 CSharpier 설치
- `csharpier format` / `csharpier check` 정확히 실행
- `.csharpierrc` / `.csharpierignore` 작성
- MSBuild 자동 포맷, Husky pre-commit hook, CI 파이프라인 통합
- 흔한 오류 진단

## 우선순위 결정 트리

작업을 시작하기 전에 먼저 다음을 판단한다.

1. 프로젝트 루트에서 `.config/dotnet-tools.json` 존재 여부와 그 안에 `csharpier`가 있는지 확인 (`Read` 또는 `Grep`).
2. 없으면 글로벌 설치 여부 확인: `dotnet tool list -g`.
3. 설치 버전 확인: `dotnet csharpier --version`.
   - **버전 1.x**: 명령어가 `csharpier format`, `csharpier check`이다.
   - **버전 0.x**: 명령어가 `csharpier`(기본 포맷), `csharpier --check`이다.
   - 이 스킬에서는 1.x 형식을 기준으로 설명한다. 0.x를 발견하면 사용자에게 업그레이드를 권하거나 명령을 변환해 사용한다.

## 설치

### 로컬(권장) — 버전을 매니페스트로 고정

```bash
dotnet new tool-manifest          # .config/dotnet-tools.json 생성
dotnet tool install csharpier     # 매니페스트에 설치
dotnet tool restore               # 팀원 / CI 환경
```

로컬 설치를 권하는 이유는, CSharpier는 마이너 버전 사이에서도 포맷 결과가 달라질 수 있어 로컬·CI 버전을 맞춰두지 않으면 "내 PC에선 통과했는데 CI에서 실패"가 빈발하기 때문이다.

### 글로벌

```bash
dotnet tool install csharpier -g
```

### MSBuild 통합 — 빌드 시 자동 포맷/검증

`.csproj`에 추가한다.

```xml
<ItemGroup>
  <PackageReference Include="CSharpier.MsBuild" Version="*" PrivateAssets="all">
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

동작 방식은 다음과 같다.

- **Debug 빌드**: 프로젝트 폴더의 파일을 자동 포맷한다.
- **Release 빌드**: `csharpier --check`를 실행해 미포맷 파일이 있으면 빌드를 실패시킨다.

제어용 MSBuild 프로퍼티는 다음과 같다.

- `<CSharpier_Bypass>true</CSharpier_Bypass>` — CSharpier 단계 전체 건너뛰기
- `<CSharpier_Check>true</CSharpier_Check>` — Debug에서도 강제로 check 모드
- `<CSharpier_LogLevel>Information</CSharpier_LogLevel>` — `--log-level` 값 전달
- `<CSharpier_UnformattedAsWarnings>true</CSharpier_UnformattedAsWarnings>` — 미포맷을 에러가 아닌 경고로 처리

## 자주 쓰는 명령

| 목적 | 명령 |
|------|------|
| 현재 디렉터리 전체 포맷 | `dotnet csharpier format .` |
| 단일 파일 포맷 | `dotnet csharpier format Path/To/File.cs` |
| CI 검증(미포맷이면 exit 1) | `dotnet csharpier check .` |
| stdin → stdout 포맷 | `cat File.cs \| dotnet csharpier format --write-stdout` |
| 커스텀 설정 파일 사용 | `dotnet csharpier format . --config-path ./build/.csharpierrc` |
| 커스텀 ignore 파일 사용 | `dotnet csharpier format . --ignore-path ./build/.csharpierignore` |
| 자동 생성 파일까지 포맷 | `dotnet csharpier format . --include-generated` |
| 검증 단계 건너뛰기(테스트용) | `dotnet csharpier format . --skip-validation` |
| 변경은 쓰지 않음(리허설) | `dotnet csharpier format . --skip-write` |
| 다중 파일 파이프(IDE용) | `dotnet csharpier --pipe-multiple-files` |
| MSBuild 형식 메시지 | `dotnet csharpier check . --log-format msbuild` |
| 미포맷을 경고로 강등 | `dotnet csharpier check . --check-ignore-warnings` |

## 설정 파일 (.csharpierrc)

해석 순서는 다음과 같다.

1. 대상 파일과 같은 위치 또는 상위 디렉터리의 `.csharpierrc` / `.csharpierrc.json` / `.csharpierrc.yaml`
2. 위가 없으면 `.editorconfig`
3. `.csharpierrc*`가 같은 디렉터리에 있으면 `.editorconfig`보다 우선한다.

YAML 예시다.

```yaml
printWidth: 100
useTabs: false
indentSize: 4
endOfLine: auto       # auto | lf | crlf
overrides:
  - files: "*.cst"
    formatter: csharp
```

`.editorconfig`만 사용한다면 다음과 같다.

```ini
[*.cs]
max_line_length = 100
indent_style = space
indent_size = 4
end_of_line = lf
```

설정 파일 위치가 비표준이면 `--config-path`로 전달한다. `.csharpierrc`, `.yaml`, `.editorconfig` 모두 지원한다.

## 코드 제외 (.csharpierignore)

### 파일 단위 — gitignore 문법

```
# 자동 생성 클라이언트는 제외
src/Generated/**/*.cs
# 단, 이 파일은 다시 포함
!src/Generated/Special.cs
```

같은 디렉터리 트리에서는 `.csharpierignore`가 `.gitignore`보다 우선한다. `.gitignore`로 무시된 파일을 다시 포맷 대상으로 끌어오려면 `.csharpierignore`에 negated 패턴(`!path`)을 넣는다. 다른 위치를 쓰려면 `--ignore-path`를 사용한다.

### 구간 단위 — 주석

```csharp
// csharpier-ignore
public static readonly int[,] Lookup =
{
    { 1, 2, 3 },
    { 4, 5, 6 },
};
```

`// csharpier-ignore`는 바로 다음 노드(문장 또는 멤버) 하나만 건너뛴다. 의도적으로 정렬을 유지하고 싶은 lookup 테이블, ASCII 아트 주석 같은 곳에 적합하다.

## Pre-commit 훅 (Husky.NET)

```bash
dotnet new tool-manifest          # 필요 시
dotnet tool install Husky
dotnet husky install
git config core.hooksPath .husky
```

`.husky/pre-commit` 예시다.

```bash
#!/usr/bin/env sh
. "$(dirname "$0")/_/husky.sh"

dotnet tool restore
dotnet csharpier check .
```

스테이징된 `.cs` 파일만 검사하고 싶다면 다음을 사용한다.

```bash
files=$(git diff --name-only --cached --diff-filter=ACMR -- '*.cs')
[ -z "$files" ] && exit 0
echo "$files" | xargs dotnet csharpier check
```

## CI 통합

GitHub Actions 최소 예시다.

```yaml
- uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '8.0.x'
- run: dotnet tool restore
- run: dotnet csharpier check .
```

CI에서는 항상 로컬 매니페스트(`.config/dotnet-tools.json`)에 버전을 고정해 두고 `dotnet tool restore`를 먼저 실행한다. 이렇게 하지 않으면 dev/CI 버전이 어긋나 같은 코드가 실패할 수 있다. 미포맷을 빌드 실패가 아닌 경고로 강등하려면 `--check-ignore-warnings`를 추가한다.

## 워크플로 1 — "내 C# 프로젝트를 포맷해줘"

1. CSharpier 설치 여부 확인 (`Read .config/dotnet-tools.json` 또는 `dotnet tool list -g`).
2. 없으면 로컬 설치를 권장하고 동의를 받은 뒤 설치한다.
3. 솔루션 루트에서 `dotnet csharpier format .` 실행.
4. 변경 규모를 사용자에게 보여준다: `git diff --stat`, 필요하면 `git diff` 일부를 인용한다.
5. 사용자가 강제 적용을 원하면 다음 중 하나 이상을 제안한다: `.csharpierignore` 작성, `CSharpier.MsBuild` 패키지 추가, Husky pre-commit, CI step.

## 워크플로 2 — "CI에서 csharpier --check가 실패한다"

1. 로컬에서 동일 명령으로 재현: `dotnet csharpier check . --no-cache`.
2. 실패하면 `dotnet csharpier format .`으로 일괄 수정 후 diff를 확인한다.
3. CSharpier는 의견이 강한 포맷터다. 사용자가 결과 스타일이 마음에 안 든다고 해도 코드를 수동 변경하지 말고 `.csharpierrc`(`printWidth` 등)나 override를 조정해야 일관성이 유지된다.
4. CI와 로컬 버전 비교:
   - 로컬: `dotnet csharpier --version`
   - CI: `.config/dotnet-tools.json`에 고정되어 있는지 확인
   - 두 값이 다르면 보통 이게 원인이다. 매니페스트에 버전을 명시하고 `dotnet tool restore`를 CI에 추가한다.

## 워크플로 3 — "기존 .editorconfig를 살리고 싶다"

`.csharpierrc`를 만들지 말고 `.editorconfig`만 사용한다. CSharpier는 `[*.cs]` 섹션의 `max_line_length`, `indent_style`, `indent_size`, `end_of_line`을 인식한다. `.csharpierrc`를 같은 트리에 두면 `.editorconfig`가 무시되니 주의한다.

## 자주 만나는 함정

- **버전 드리프트**: 로컬과 CI가 다른 마이너 버전을 쓰면 같은 코드가 한쪽에서만 실패한다. 항상 로컬 매니페스트로 고정한다.
- **자동 생성 파일**: SDK가 만든 파일과 `<autogenerated>` / `<auto-generated>` 주석을 가진 파일은 기본 무시 대상이다. `--include-generated` 없이는 포맷되지 않는다. 빌드 시 재생성되는 파일은 가급적 포맷하지 않는다.
- **이중 포맷**: `CSharpier.MsBuild` + Husky pre-commit + IDE save-on-format을 동시에 켜면 같은 변경이 여러 번 발생해 혼란스럽다. 보통 둘 중 하나만 둔다.
- **에디터 ↔ CLI 불일치**: VS / VS Code / Rider 확장이 CLI와 다른 결과를 낸다면 확장이 다른 CSharpier 버전을 들고 있을 가능성이 높다. 확장을 갱신하거나 매니페스트 버전과 맞춘다.
- **설정 위치**: `.csharpierrc`는 대상 파일과 같은 디렉터리 또는 상위에 있어야 적용된다. 솔루션 외부에 두면 무시된다.

## CSharpier가 하지 *않는* 일

- **린터가 아니다**: 미사용 변수, 잠재 버그, 네이밍 위반을 잡지 않는다. Roslyn analyzers, `dotnet format analyzers`, StyleCop, SonarAnalyzer 등과 조합한다.
- **스타일 규칙(네이밍, var vs 명시 타입 등)을 강제하지 않는다**: 이런 규칙은 `.editorconfig`의 IDE/CA 룰과 `dotnet format style`로 처리한다.
- **자동 import 정리, using 정렬을 하지 않는다**: 필요하면 `dotnet format`을 함께 사용한다.

## 파일 작성 시 주의

- `.csharpierrc`, `.csharpierignore`를 새로 만들 때는 절대 기존 사용자 파일을 덮어쓰지 말고, 먼저 `Read`로 존재 여부를 확인한 뒤 변경 사항을 사용자에게 보여주고 합의 후 작성한다.
- `dotnet csharpier format .`은 파일을 바로 수정한다. 큰 솔루션이라면 먼저 `--skip-write`로 영향 범위를 가늠하거나, 변경 전 git 상태를 깨끗하게 해두라고 권한다.

## 참고 링크

- 공식 문서 입구: https://csharpier.com/docs/About
- CLI 옵션 전체: https://csharpier.com/docs/CLI
- 설정: https://csharpier.com/docs/Configuration
- 무시 규칙: https://csharpier.com/docs/Ignore
- MSBuild 패키지: https://csharpier.com/docs/MsBuild
- Pre-commit: https://csharpier.com/docs/Pre-commit
- 저장소: https://github.com/belav/csharpier
- NuGet (CLI): https://www.nuget.org/packages/CSharpier
- NuGet (MSBuild): https://www.nuget.org/packages/CSharpier.MsBuild
