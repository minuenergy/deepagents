# Deep Agents 설정 및 활용 가이드

> 이 가이드는 `deepagents` 코드베이스 분석과 공식 문서를 기반으로 작성되었습니다.
> Deep Agents는 LangGraph 위에 구축된 배터리 포함형(agent harness) 프레임워크입니다.

---

## 목차

1. [Agent 설정 및 책임 분배](#1-agent-설정-및-책임-분배)
2. [새로운 Tool 등록 및 사용](#2-새로운-tool-등록-및-사용)
3. [Skill 등록 및 관리](#3-skill-등록-및-관리)
4. [Workflow 모니터링](#4-workflow-모니터링)
5. [Checkpoint 설정 및 Memory 저장/로드](#5-checkpoint-설정-및-memory-저장로드)

---

## 1. Agent 설정 및 책임 분배

### 1-1. 핵심 진입점: `create_deep_agent()`

> 코드 위치: `libs/deepagents/deepagents/graph.py:50`

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-5-20250929",  # 또는 "openai:gpt-4o" 등
    tools=[...],                    # 커스텀 도구
    system_prompt="...",            # 에이전트 역할/지침
    middleware=[...],               # 추가 미들웨어
    subagents=[...],                # 서브에이전트 목록
    skills=["./skills/"],           # 스킬 디렉토리 경로
    memory=["./AGENTS.md"],         # 메모리 파일 경로
    backend=FilesystemBackend(...), # 스토리지 백엔드
    checkpointer=...,              # 체크포인터 (상태 영속화)
    store=...,                     # 크로스-스레드 장기 메모리 스토어
    interrupt_on={...},            # HITL (Human-in-the-Loop) 설정
)
```

### 1-2. 기본 내장 도구 (자동 제공)

에이전트 생성 시 자동으로 아래 도구들이 미들웨어를 통해 주입됩니다:

| 도구 | 미들웨어 | 설명 |
|------|----------|------|
| `write_todos`, `read_todos` | `TodoListMiddleware` | 계획 수립 및 진행 추적 |
| `read_file`, `write_file`, `edit_file`, `ls`, `glob`, `grep` | `FilesystemMiddleware` | 파일 시스템 조작 |
| `execute` | `FilesystemMiddleware` (SandboxBackend 필요) | 셸 명령 실행 |
| `task` | `SubAgentMiddleware` | 서브에이전트 위임 |

### 1-3. 기본 미들웨어 스택 (자동 구성)

> 코드 위치: `libs/deepagents/deepagents/graph.py:226-255`

`create_deep_agent()` 호출 시 아래 미들웨어가 순서대로 자동 구성됩니다:

```
1. TodoListMiddleware          → 계획/TODO 도구 주입
2. MemoryMiddleware            → AGENTS.md 메모리 로드 (memory= 지정 시)
3. SkillsMiddleware            → 스킬 시스템 (skills= 지정 시)
4. FilesystemMiddleware        → 파일 조작 도구 주입
5. SubAgentMiddleware          → task 도구 (서브에이전트) 주입
6. SummarizationMiddleware     → 컨텍스트 자동 요약/압축
7. AnthropicPromptCachingMiddleware → 프롬프트 캐싱 (Anthropic 모델)
8. PatchToolCallsMiddleware    → 도구 호출 보정
9. [사용자 커스텀 미들웨어]     → middleware= 매개변수로 추가
10. HumanInTheLoopMiddleware   → HITL (interrupt_on= 지정 시)
```

### 1-4. 서브에이전트로 책임 분배

> 코드 위치: `libs/deepagents/deepagents/middleware/subagents.py:22-78`

#### 서브에이전트의 핵심 가치
- **컨텍스트 격리**: 서브에이전트는 독립 컨텍스트 윈도우에서 실행 → 메인 에이전트 컨텍스트 오염 방지
- **전문화**: 서브에이전트마다 고유 system_prompt, tools, model 지정 가능
- **병렬 실행**: 여러 서브에이전트 동시 실행으로 지연 최소화
- **멀티 모델**: 서브에이전트마다 다른 LLM 사용 가능 (비용/속도 최적화)

#### 서브에이전트 정의 방법

**방법 A: 인라인 딕셔너리**
```python
research_subagent = {
    "name": "researcher",
    "description": "웹에서 주제를 조사하고 결과를 파일로 저장합니다",
    "system_prompt": "당신은 리서치 전문가입니다...",
    "tools": [web_search],                    # 선택: 커스텀 도구
    "model": "anthropic:claude-haiku-4-5-20251001",  # 선택: 모델 오버라이드
    "middleware": [...],                       # 선택: 추가 미들웨어
    "skills": ["./skills/research/"],          # 선택: 스킬 경로
}

agent = create_deep_agent(subagents=[research_subagent])
```

**방법 B: YAML 외부 파일 (예시 프로젝트 패턴)**

> 참고: `examples/content-builder-agent/subagents.yaml`

```yaml
researcher:
  description: >
    웹에서 주제를 조사하고 결과를 파일로 저장합니다.
  model: anthropic:claude-haiku-4-5-20251001
  system_prompt: |
    당신은 리서치 전문가입니다.
    ## 사용 가능한 도구
    - web_search(query, max_results=5)
    - write_file(file_path, content)
  tools:
    - web_search
```

```python
# YAML 로드 후 create_deep_agent()에 전달
subagents = load_subagents("subagents.yaml")
agent = create_deep_agent(subagents=subagents)
```

**방법 C: CompiledSubAgent (사전 컴파일된 에이전트)**

> 코드 위치: `libs/deepagents/deepagents/middleware/subagents.py:81-110`

```python
from deepagents import CompiledSubAgent

# LangGraph나 create_agent로 만든 커스텀 에이전트를 서브에이전트로 사용
compiled = CompiledSubAgent(
    name="custom-agent",
    description="커스텀 워크플로우를 실행하는 에이전트",
    runnable=my_custom_langgraph_agent,  # state에 'messages' 키 필수
)

agent = create_deep_agent(subagents=[compiled])
```

#### 내장 General-Purpose 서브에이전트

`create_deep_agent()`는 자동으로 **general-purpose** 서브에이전트를 생성합니다.
이 서브에이전트는 메인 에이전트와 동일한 system_prompt, tools, model을 상속합니다.
컨텍스트 격리 목적으로 사용하기에 적합합니다.

#### 멀티 에이전트 구조 설계 예시

```
Main Agent (claude-sonnet)
├── system_prompt: "콘텐츠 작성 전문가..."
├── tools: [generate_cover, generate_social_image]
├── memory: ["./AGENTS.md"]      ← 브랜드 가이드
├── skills: ["./skills/"]        ← blog-post, social-media
│
├── Subagent: researcher (claude-haiku) ← 비용 절감
│   ├── tools: [web_search]
│   └── system_prompt: "리서치 전문가..."
│
├── Subagent: code-reviewer (gpt-4o) ← 다른 모델
│   ├── tools: [read_file, grep]
│   └── system_prompt: "코드 리뷰어..."
│
└── Subagent: general-purpose ← 자동 생성 (메인과 동일)
```

---

## 2. 새로운 Tool 등록 및 사용

### 2-1. `@tool` 데코레이터로 커스텀 도구 생성

```python
from langchain_core.tools import tool

@tool
def web_search(query: str, max_results: int = 5) -> dict:
    """웹에서 최신 정보를 검색합니다.

    Args:
        query: 검색 쿼리 (구체적으로 작성)
        max_results: 반환할 결과 수
    """
    from tavily import TavilyClient
    client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])
    return client.search(query, max_results=max_results)
```

> 핵심: docstring이 에이전트가 도구를 언제/어떻게 사용할지 결정하는 데 사용됩니다.

### 2-2. 도구를 에이전트에 등록

**메인 에이전트에 직접 등록:**
```python
agent = create_deep_agent(
    tools=[web_search, generate_cover, my_custom_tool],
)
```

**서브에이전트에만 등록:**
```python
agent = create_deep_agent(
    subagents=[{
        "name": "researcher",
        "description": "...",
        "system_prompt": "...",
        "tools": [web_search],  # 이 서브에이전트만 사용
    }],
)
```

### 2-3. 미들웨어를 통한 도구 등록

커스텀 미들웨어를 작성하여 도구를 동적으로 주입할 수 있습니다:

```python
from langchain.agents.middleware.types import AgentMiddleware, ModelRequest, ModelResponse

class CustomToolMiddleware(AgentMiddleware):
    """커스텀 도구를 에이전트에 주입하는 미들웨어."""

    def wrap_model_call(self, request, handler):
        # request를 수정하거나 도구 결과를 후처리 가능
        return handler(request)
```

### 2-4. MCP (Model Context Protocol) 도구 연동

> 설정 파일: `.mcp.json`

```json
{
  "mcpServers": {
    "langchain-docs": {
      "type": "http",
      "url": "https://docs.langchain.com/mcp"
    }
  }
}
```

MCP 서버의 도구를 `langchain-mcp-adapters`를 통해 에이전트에 연결할 수 있습니다.

### 2-5. Skill 내장 스크립트를 도구로 활용

스킬 디렉토리 안의 Python 스크립트도 도구처럼 사용 가능합니다:

```
skills/
└── web-research/
    ├── SKILL.md           # 스킬 메타데이터 + 지침
    └── scripts/
        └── search.py      # 에이전트가 execute 도구로 실행
```

---

## 3. Skill 등록 및 관리

### 3-1. Skill 시스템 개요

> 코드 위치: `libs/deepagents/deepagents/middleware/skills.py`

스킬은 **Progressive Disclosure** 패턴을 따릅니다:
1. 에이전트는 처음에 스킬의 **이름과 설명만** 봅니다
2. 사용자의 요청이 스킬에 매치되면 **SKILL.md 전체 내용을 읽습니다**
3. SKILL.md의 지침을 따라 작업을 수행합니다

### 3-2. Skill 디렉토리 구조

```
skills/
├── blog-post/              # 스킬 이름 = 디렉토리 이름
│   ├── SKILL.md            # 필수: YAML 프론트매터 + 마크다운 지침
│   └── scripts/            # 선택: 보조 스크립트
│       └── helper.py
│
└── social-media/
    ├── SKILL.md
    └── templates/
        └── linkedin.md
```

### 3-3. SKILL.md 작성법

> Agent Skills 명세 기준: https://agentskills.io/specification

```markdown
---
name: blog-post
description: 블로그 포스트, 튜토리얼, 교육 기사 작성 시 사용하는 스킬
license: MIT
allowed-tools: web_search generate_cover
---

# Blog Post Writing Skill

## When to Use This Skill
- 블로그 포스트 또는 기사 작성 요청 시
- 튜토리얼/가이드 제작 시

## Workflow
1. `task` 도구로 researcher 서브에이전트에 조사 위임
2. 조사 결과 파일 읽기
3. 정해진 구조로 글 작성
4. 커버 이미지 생성

## Output Structure
blogs/<slug>/post.md
blogs/<slug>/hero.png
```

**YAML 프론트매터 필수 필드:**

| 필드 | 필수 | 제약 | 설명 |
|------|------|------|------|
| `name` | O | 최대 64자, 소문자+하이픈, 디렉토리명과 일치 | 스킬 식별자 |
| `description` | O | 최대 1024자 | 스킬이 하는 일 |
| `license` | X | - | 라이선스 |
| `compatibility` | X | 최대 500자 | 환경 요구사항 |
| `allowed-tools` | X | 스페이스 구분 | 사전 승인된 도구 목록 |
| `metadata` | X | key-value 맵 | 추가 메타데이터 |

### 3-4. Skill 등록 방법

```python
agent = create_deep_agent(
    skills=[
        "./skills/base/",      # 기본 스킬 (낮은 우선순위)
        "./skills/user/",      # 사용자 스킬
        "./skills/project/",   # 프로젝트 스킬 (높은 우선순위, 동명 시 override)
    ],
    backend=FilesystemBackend(root_dir="/path/to/project"),
)
```

**우선순위**: 나중에 지정된 소스가 높은 우선순위 → 동일 이름의 스킬이 있으면 후자가 승리

### 3-5. Skill 소스 레이어링 패턴

```python
# 레이어 우선순위: base < user < project < team
skills=[
    "/skills/base/",      # 프레임워크 기본 스킬
    "/skills/user/",      # 사용자 개인 스킬
    "/skills/project/",   # 프로젝트별 스킬
    "/skills/team/",      # 팀 공유 스킬 (최고 우선순위)
]
```

### 3-6. 스킬이 서브에이전트에도 전달되는 방식

`create_deep_agent()` 내부에서 `skills=` 매개변수를 지정하면:
- 메인 에이전트의 미들웨어 스택에 `SkillsMiddleware` 추가
- 내장 general-purpose 서브에이전트에도 동일한 `SkillsMiddleware` 추가
- 커스텀 서브에이전트는 별도 `skills` 필드로 지정 가능

---

## 4. Workflow 모니터링

### 4-1. 내장 모니터링: TODO 시스템

> `TodoListMiddleware`가 자동 주입하는 `write_todos` 도구

에이전트가 자체적으로 복잡한 작업을 단계별로 분해하고 진행 상황을 추적합니다:

```python
# 에이전트가 자동으로 사용:
write_todos([
    {"content": "조사 수행", "status": "completed"},
    {"content": "초안 작성", "status": "in_progress"},
    {"content": "이미지 생성", "status": "pending"},
])
```

### 4-2. LangSmith를 통한 프로덕션 모니터링

> 참고: https://www.langchain.com/langsmith/observability

LangSmith는 Deep Agents의 실행을 추적하고 분석하는 통합 플랫폼입니다:

**주요 개념:**
- **Run**: 개별 단계 (LLM 호출, 도구 호출 등)
- **Trace**: 에이전트의 단일 실행 전체 (수십~수백 개의 Run으로 구성)
- **Thread**: 사용자와 에이전트 간 전체 대화

**설정 방법:**
```bash
# 환경변수 설정
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY=<your-key>
export LANGCHAIN_PROJECT=my-deep-agent  # 또는 DEEPAGENTS_LANGSMITH_PROJECT
```

```python
# 에이전트 실행 시 자동으로 LangSmith에 트레이싱
agent = create_deep_agent(...)
result = agent.invoke({"messages": [("user", "블로그 작성해줘")]})
# → LangSmith 대시보드에서 전체 실행 흐름 확인 가능
```

**LangSmith에서 확인 가능한 것:**
- 전체 에이전트 실행 플로우 (LLM 호출 → 도구 호출 → 서브에이전트)
- 각 단계의 입력/출력, 토큰 사용량, 지연시간
- 서브에이전트의 중첩 트레이스
- 에러 발생 지점 추적

**CLI 디버깅: LangSmith Fetch**
```bash
# IDE나 코드 에이전트에서 LangSmith 트레이스에 직접 접근
langsmith fetch <trace-id>
```

**Polly (AI 기반 트레이스 분석):**
LangSmith 앱 내에서 AI가 트레이스/스레드 데이터를 분석해주는 기능입니다.

### 4-3. 스트리밍을 통한 실시간 모니터링

```python
# 실시간 스트리밍으로 에이전트 동작 관찰
async for chunk in agent.astream(
    {"messages": [("user", task)]},
    config={"configurable": {"thread_id": "my-thread"}},
    stream_mode="values",
):
    if "messages" in chunk:
        for msg in chunk["messages"]:
            # AIMessage, ToolMessage 등을 실시간으로 처리
            print(msg)
```

### 4-4. Summarization에 의한 자동 컨텍스트 관리

> 코드 위치: `libs/deepagents/deepagents/middleware/summarization.py`

`SummarizationMiddleware`가 자동으로:
- 모델의 `max_input_tokens`의 **85%** 도달 시 이전 대화를 요약/압축
- 대용량 도구 결과를 파일 시스템으로 자동 이관 (토큰 절약)

---

## 5. Checkpoint 설정 및 Memory 저장/로드

### 5-1. 두 가지 메모리 체계

Deep Agents에서 메모리는 두 가지 레벨로 운영됩니다:

| 구분 | 메커니즘 | 범위 | 용도 |
|------|----------|------|------|
| **단기 메모리** | Checkpointer | 스레드 내 | 대화 히스토리, 에이전트 상태 영속화 |
| **장기 메모리** | Store + AGENTS.md | 스레드 간 | 사용자 선호도, 프로젝트 지식, 학습 내용 |

### 5-2. Checkpoint 설정 (단기 메모리)

#### InMemorySaver (개발/테스트용)
```python
from langgraph.checkpoint.memory import MemorySaver

agent = create_deep_agent(
    checkpointer=MemorySaver(),
)

# thread_id로 대화 상태 유지
config = {"configurable": {"thread_id": "conversation-1"}}
result1 = agent.invoke({"messages": [("user", "안녕하세요")]}, config=config)
result2 = agent.invoke({"messages": [("user", "방금 뭐라고 했죠?")]}, config=config)
# → result2에서 이전 대화 기억
```

#### SQLiteSaver (로컬 영속화)
```python
from langgraph.checkpoint.sqlite import SqliteSaver

with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    agent = create_deep_agent(checkpointer=checkpointer)
    # 프로세스 재시작 후에도 상태 복원 가능
```

#### PostgresSaver (프로덕션)
```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@host:5432/db"
)
agent = create_deep_agent(checkpointer=checkpointer)
```

#### 체크포인트로 가능한 것들
- **대화 재개**: 같은 `thread_id`로 이전 대화 이어가기
- **Time Travel**: 이전 체크포인트로 되돌아가 상태 검사
- **Human-in-the-Loop**: `interrupt_on`으로 특정 도구 실행 전 일시정지 → 승인 후 재개
- **장애 복구**: 에러 발생 시 마지막 체크포인트에서 재시작

### 5-3. AGENTS.md 기반 장기 메모리

> 코드 위치: `libs/deepagents/deepagents/middleware/memory.py`

`MemoryMiddleware`는 `AGENTS.md` 파일을 로드하여 시스템 프롬프트에 주입합니다.
에이전트는 `edit_file` 도구로 이 파일을 수정하여 **학습 내용을 영속적으로 기록**합니다.

#### 설정 방법
```python
from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend

agent = create_deep_agent(
    memory=[
        "~/.deepagents/AGENTS.md",        # 글로벌 메모리 (사용자 선호도)
        "./.deepagents/AGENTS.md",         # 프로젝트별 메모리
    ],
    backend=FilesystemBackend(root_dir="/"),
)
```

#### AGENTS.md 파일 예시

> 참고: `examples/content-builder-agent/AGENTS.md`

```markdown
# Content Writer Agent

당신은 테크 기업의 콘텐츠 작성자입니다.

## Brand Voice
- 전문적이지만 친근한 톤
- 명확하고 직접적인 표현

## Writing Standards
1. 능동태 사용
2. 가치를 먼저 제시
3. 문단당 하나의 아이디어

## 사용자 선호도 (학습됨)
- JavaScript 예제 선호
- 코드 블록에 항상 언어 태그 포함
```

#### 메모리 자동 학습 메커니즘

`MemoryMiddleware`는 에이전트에게 다음 지침을 주입합니다:

1. **암묵적/명시적 학습**: 사용자가 수정 피드백을 주면 WHY를 캡처하여 패턴으로 기록
2. **즉시 저장**: 기억할 정보를 발견하면 다른 작업보다 먼저 `edit_file`로 AGENTS.md 업데이트
3. **영구적 개선**: 실수를 단순 수정이 아닌, 근본 원칙으로 기록

**저장해야 하는 것:**
- 사용자 역할/행동 지침
- 작업 피드백 (무엇이 잘못되었고 어떻게 개선할지)
- 도구 사용에 필요한 정보 (Slack 채널 ID, 이메일 등)
- 발견된 패턴/선호도 (코딩 스타일, 컨벤션)

**저장하지 말아야 하는 것:**
- 일시적 정보 ("오늘 늦을 거야")
- 일회성 작업 ("25*4는?")
- API 키, 비밀번호 등 자격증명

### 5-4. Store 기반 크로스 스레드 장기 메모리

Checkpointer는 스레드 내 상태만 유지합니다.
**스레드 간** 정보를 공유하려면 LangGraph의 `Store` 인터페이스를 사용합니다.

```python
from langgraph.store.memory import InMemoryStore
# 프로덕션에서는 PostgresStore, RedisStore 등 사용

store = InMemoryStore()

agent = create_deep_agent(
    checkpointer=MemorySaver(),
    store=store,           # 크로스 스레드 메모리
    backend=StoreBackend,  # 스토어 기반 백엔드
)
```

**Store vs Checkpointer 비교:**

```
Checkpointer (단기 메모리)          Store (장기 메모리)
├── 스레드 범위                     ├── 전역 범위 (모든 스레드)
├── 대화 히스토리 유지               ├── 사용자 프로필/선호도
├── 에이전트 상태 스냅샷             ├── 학습된 패턴
├── thread_id로 접근                ├── namespace/key로 접근
└── 자동 저장                       └── 명시적 put/get
```

### 5-5. Backend 시스템 (파일 스토리지)

> 코드 위치: `libs/deepagents/deepagents/backends/`

| 백엔드 | 용도 | 영속성 |
|--------|------|--------|
| `StateBackend` | 기본값. 에이전트 상태 내 임시 저장 | 없음 (실행 중만) |
| `FilesystemBackend` | 로컬 디스크 읽기/쓰기 | 있음 |
| `StoreBackend` | LangGraph Store 기반 | Store 구현에 따라 다름 |
| `CompositeBackend` | 경로별 다른 백엔드 라우팅 | 혼합 |
| `LocalShellBackend` | 로컬 셸 명령 실행 | - |
| `SandboxBackend` | 샌드박스 환경 실행 | - |

#### FilesystemBackend 사용 예시
```python
from deepagents.backends import FilesystemBackend

agent = create_deep_agent(
    backend=FilesystemBackend(root_dir="/path/to/project"),
    memory=["./AGENTS.md"],     # root_dir 기준 상대 경로
    skills=["./skills/"],
)
```

### 5-6. 전체 메모리 아키텍처 조합 예시

```python
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.store.memory import InMemoryStore
from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend

# 1. Checkpoint: 대화 상태 영속화
checkpointer = PostgresSaver.from_conn_string("postgresql://...")

# 2. Store: 크로스 스레드 장기 메모리
store = InMemoryStore()  # 프로덕션에서는 PostgresStore 사용

# 3. Backend: 파일 시스템 접근
backend = FilesystemBackend(root_dir="/my/project")

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-5-20250929",
    memory=["./AGENTS.md"],         # 프로젝트 컨텍스트 + 학습 내용
    skills=["./skills/"],           # 재사용 가능한 워크플로우
    subagents=[researcher, reviewer],
    checkpointer=checkpointer,      # 대화 재개, time travel
    store=store,                    # 스레드 간 정보 공유
    backend=backend,                # 디스크 읽기/쓰기
    interrupt_on={"edit_file": True},  # 파일 수정 전 사람 승인
)

# 실행: 같은 thread_id로 대화 이어가기
config = {"configurable": {"thread_id": "project-session-1"}}
result = agent.invoke(
    {"messages": [("user", "블로그 포스트 작성해줘")]},
    config=config,
)
```

### 5-7. 메모리 저장 후 다시 로드하는 흐름

```
[세션 1]
사용자: "코드 예제는 항상 TypeScript로 해줘"
    ↓
에이전트: edit_file("./AGENTS.md") → "사용자 선호: TypeScript 코드 예제" 추가
    ↓
Checkpointer → 대화 상태 자동 저장 (PostgresSaver)

[세션 2] (새로운 thread_id)
    ↓
MemoryMiddleware → AGENTS.md 로드 → 시스템 프롬프트에 주입
    ↓
에이전트: "아, 이 사용자는 TypeScript를 선호하는구나" (장기 메모리에서 복원)
    ↓
사용자: "이전 대화 이어서 해줘"
    → 같은 thread_id 사용 시: Checkpointer에서 대화 히스토리 복원
    → 다른 thread_id 사용 시: AGENTS.md의 학습 내용만 유지
```

---

## 참고 리소스

| 리소스 | URL |
|--------|-----|
| GitHub 리포지토리 | https://github.com/langchain-ai/deepagents |
| Deep Agents 개요 | https://docs.langchain.com/oss/python/deepagents/overview |
| 커스터마이징 | https://docs.langchain.com/oss/python/deepagents/customization |
| 서브에이전트 | https://docs.langchain.com/oss/python/deepagents/subagents |
| 스킬 시스템 | https://docs.langchain.com/oss/python/deepagents/skills |
| 미들웨어 | https://docs.langchain.com/oss/python/deepagents/middleware |
| 장기 메모리 | https://docs.langchain.com/oss/python/deepagents/long-term-memory |
| Harness 기능 | https://docs.langchain.com/oss/python/deepagents/harness |
| HITL (Human-in-the-Loop) | https://docs.langchain.com/oss/python/deepagents/human-in-the-loop |
| API 레퍼런스 | https://reference.langchain.com/python/deepagents/ |
| LangSmith 모니터링 | https://www.langchain.com/langsmith/observability |
| LangSmith 배포 | https://docs.langchain.com/langsmith/deployments |
| LangGraph 영속성 | https://docs.langchain.com/oss/python/langgraph/persistence |
| LangSmith 디버깅 블로그 | https://blog.langchain.com/debugging-deep-agents-with-langsmith/ |
| PyPI | https://pypi.org/project/deepagents/ |

---

## 핵심 코드 파일 위치 (이 리포지토리 기준)

| 파일 | 설명 |
|------|------|
| `libs/deepagents/deepagents/graph.py` | `create_deep_agent()` 메인 팩토리 |
| `libs/deepagents/deepagents/__init__.py` | 공개 API exports |
| `libs/deepagents/deepagents/middleware/subagents.py` | 서브에이전트 시스템 |
| `libs/deepagents/deepagents/middleware/skills.py` | 스킬 미들웨어 |
| `libs/deepagents/deepagents/middleware/memory.py` | 메모리(AGENTS.md) 미들웨어 |
| `libs/deepagents/deepagents/middleware/filesystem.py` | 파일 시스템 도구 |
| `libs/deepagents/deepagents/middleware/summarization.py` | 자동 요약/컨텍스트 압축 |
| `libs/deepagents/deepagents/backends/protocol.py` | 백엔드 인터페이스 정의 |
| `libs/deepagents/deepagents/backends/filesystem.py` | 파일 시스템 백엔드 |
| `libs/deepagents/deepagents/backends/store.py` | Store 기반 백엔드 |
| `examples/content-builder-agent/` | 종합 예제 (메모리+스킬+서브에이전트+커스텀 도구) |
| `examples/deep_research/` | 멀티스텝 리서치 에이전트 예제 |
| `examples/text-to-sql-agent/` | SQL 에이전트 예제 |
