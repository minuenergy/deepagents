# OpenClaw 코드 분석 및 워크플로우 설명

## 1. 프로젝트 개요

**OpenClaw**(구 Clawdbot, Moltbot)은 Peter Steinberger가 개발한 오픈소스 자율 AI 에이전트입니다. 사용자의 로컬 디바이스에서 실행되며, 메시징 플랫폼을 주요 UI로 활용하여 LLM 기반 태스크를 자율적으로 수행합니다.

- **언어**: TypeScript
- **라이선스**: MIT
- **GitHub Stars**: 145,000+ (2026년 2월 기준)
- **지원 모델**: Claude, GPT, DeepSeek, Llama 4, Mixtral 등 (모델 무관)

### 역사

| 시기 | 이름 | 비고 |
|------|------|------|
| 2025년 11월 | Clawdbot | 최초 공개 |
| 2026년 1월 27일 | Moltbot | Anthropic 상표권 이의로 변경 |
| 2026년 1월 30일 | OpenClaw | 현재 이름으로 변경 |

---

## 2. 핵심 아키텍처 (5계층 구조)

OpenClaw은 모놀리식 애플리케이션이 아닌, 역할별로 분리된 **5개 계층**의 서비스 집합체입니다.

```
┌─────────────────────────────────────────────────────┐
│                  Channel Adapters                    │
│  (WhatsApp, Telegram, Slack, Discord, Signal, ...)  │
└──────────────────────┬──────────────────────────────┘
                       │ 표준화된 메시지 포맷
                       ▼
┌─────────────────────────────────────────────────────┐
│                  Gateway Server                      │
│     (세션 관리, 라우팅, WebSocket 제어 평면)           │
│              ws://127.0.0.1:18789                    │
└──────────────────────┬──────────────────────────────┘
                       │ 세션별 큐 할당
                       ▼
┌─────────────────────────────────────────────────────┐
│                   Lane Queue                         │
│   (기본 직렬 실행, 명시적 병렬 실행 지원)              │
└──────────────────────┬──────────────────────────────┘
                       │ 실행 순서 보장
                       ▼
┌─────────────────────────────────────────────────────┐
│                  Agent Runner                        │
│  (모델 선택, 프롬프트 조립, 컨텍스트 윈도우 관리)       │
└──────────────────────┬──────────────────────────────┘
                       │ 추론 + 도구 호출
                       ▼
┌─────────────────────────────────────────────────────┐
│                  Agentic Loop                        │
│   (도구 호출 → 실행 → 결과 반영 → 반복)               │
└─────────────────────────────────────────────────────┘
```

---

## 3. 각 계층 상세 설명

### 3.1 Channel Adapter (채널 어댑터)

다양한 메시징 플랫폼의 입력을 **통합 메시지 포맷**으로 표준화하는 계층입니다.

**지원 채널** (12개+):
- WhatsApp (Baileys 라이브러리)
- Telegram (grammY 프레임워크)
- Slack (Bolt 프레임워크)
- Discord (discord.js)
- Google Chat, Signal, iMessage (BlueBubbles)
- Microsoft Teams, Matrix, Zalo, WebChat
- macOS/iOS/Android 네이티브

**역할**:
- 플랫폼별 메시지를 표준 포맷으로 변환
- 첨부파일(이미지, 파일) 추출
- 인바운드/아웃바운드 메시지 라우팅

### 3.2 Gateway Server (게이트웨이 서버)

모든 연결의 **단일 제어 평면(Control Plane)** 역할을 합니다.

```
포트: 18789 (기본)
프로토콜: WebSocket
설정 파일: ~/.openclaw/openclaw.json
```

**핵심 기능**:
- 세션 관리 및 라우팅
- 채널 → 에이전트 바인딩
- 멀티 에이전트 라우팅 (별도 workspace, agentDir, sessions)
- CLI, WebChat UI, 컴패니언 앱 연결 허브

**멀티 에이전트 구조**: 하나의 Gateway에서 여러 격리된 에이전트를 운영 가능하며, 인바운드 메시지를 바인딩 설정에 따라 적절한 에이전트로 라우팅합니다.

### 3.3 Lane Queue (레인 큐) - 핵심 혁신

OpenClaw의 가장 독자적인 아키텍처 혁신입니다.

**설계 원칙**: "Default Serial, Explicit Parallel" (기본 직렬, 명시적 병렬)

```
세션 A ──→ [Lane A] ──→ Task1 → Task2 → Task3  (직렬)
세션 B ──→ [Lane B] ──→ Task1 → Task2            (직렬)
                          └──→ [Parallel Lane] → 저위험 태스크 (병렬)
```

**큐 모드**:

| 모드 | 동작 |
|------|------|
| `steer` | 실행 중인 런에 메시지 주입. 각 도구 호출 후 큐 확인, 대기 메시지가 있으면 남은 도구 호출 건너뛰고 새 메시지 처리 |
| `followup` | 현재 턴 종료까지 대기 후 새 에이전트 턴 시작 |
| `collect` | 현재 턴 종료까지 메시지 수집, 일괄 처리 |

**핵심 이점**:
- 레이스 컨디션 방지
- 장애 격리
- 로그 가독성 유지

### 3.4 Agent Runner (에이전트 러너)

LLM 호출을 정밀하게 준비하는 계층입니다.

**4단계 실행 순서**:

1. **Model Resolver**: 에이전트/세션별 LLM 선택 (Claude Opus, Sonnet, GPT 등)
2. **System Prompt Builder**: 활성화된 스킬, 도구, 메모리 기반으로 시스템 프롬프트 동적 조립
3. **Session History Loader**: 로컬 저장소에서 영속적 대화 컨텍스트 로드
4. **Context Window Guard**: 토큰 한도 근접 시 히스토리 압축(compaction)

**타임아웃 설정**:
- `agent.wait`: 기본 30초
- 에이전트 런타임: 기본 600초 (10분)

### 3.5 Agentic Loop (에이전틱 루프)

실제 작업이 수행되는 **반복 실행 사이클**입니다.

```
┌───────────────────────────────────────┐
│                                       │
│   모델 추론 (LLM)                     │
│       │                               │
│       ▼                               │
│   도구 호출 제안                       │
│       │                               │
│       ▼                               │
│   도구 실행 (Shell, FS, Browser)      │
│       │                               │
│       ▼                               │
│   결과 반영 (Backfill)                │
│       │                               │
│       ▼                               │
│   완료 조건 확인                       │
│       │                               │
│   ┌───┴───┐                           │
│   │미완료 │──→ 루프 재시작             │
│   │완료   │──→ 최종 응답 스트리밍      │
│   └───────┘                           │
│                                       │
└───────────────────────────────────────┘
```

**라이프사이클 훅**:

| 훅 이름 | 시점 | 용도 |
|---------|------|------|
| `before_agent_start` | 에이전트 시작 전 | 컨텍스트 주입, 시스템 프롬프트 오버라이드 |
| `agent_end` | 에이전트 종료 시 | 최종 메시지 목록 검사 |
| `before_tool_call` | 도구 호출 전 | 도구 파라미터 인터셉트 |
| `after_tool_call` | 도구 호출 후 | 도구 결과 인터셉트 |
| `tool_result_persist` | 결과 저장 시 | 트랜스크립트 기록 전 결과 변환 |
| `before_compaction` | 압축 전 | 압축 사이클 관찰 |
| `after_compaction` | 압축 후 | 압축 사이클 관찰 |
| `message_received` | 메시지 수신 시 | 인바운드 처리 |
| `message_sending` | 메시지 전송 전 | 아웃바운드 처리 |
| `message_sent` | 메시지 전송 후 | 아웃바운드 확인 |
| `session_start/end` | 세션 시작/종료 | 세션 라이프사이클 관리 |
| `gateway_start/stop` | 게이트웨이 시작/종료 | 게이트웨이 라이프사이클 관리 |

---

## 4. 메시지 처리 전체 워크플로우

사용자가 메시지를 보낸 순간부터 응답을 받기까지의 전체 흐름입니다.

```
사용자 (WhatsApp/Telegram/Slack/...)
    │
    ▼
[1] Channel Adapter: 플랫폼별 메시지 → 통합 포맷 변환
    │
    ▼
[2] Gateway Server: 세션 식별 → 에이전트 바인딩 → 큐 할당
    │
    ▼
[3] Lane Queue: 직렬 큐 삽입 (기존 런 완료 대기)
    │
    ▼
[4] Agent Runner:
    ├── 모델 선택 (Claude/GPT/로컬 모델)
    ├── 스킬/도구/메모리 기반 시스템 프롬프트 동적 조립
    ├── 세션 히스토리 로드
    └── 컨텍스트 윈도우 검증 및 압축
    │
    ▼
[5] Agentic Loop:
    ├── LLM 추론
    ├── 도구 호출 (Shell, FileSystem, Browser, ...)
    ├── 결과 반영
    ├── 완료 조건 미충족 → 루프 반복
    └── 완료 → 최종 응답 생성
    │
    ▼
[6] 응답 스트리밍 → Channel Adapter → 사용자 플랫폼으로 전달
```

---

## 5. 스킬 시스템 (Skills)

스킬은 에이전트의 기능을 확장하는 **모듈형 능력 단위**입니다.

### 스킬 유형

| 유형 | 설명 |
|------|------|
| Bundled | OpenClaw에 기본 내장된 스킬 |
| Managed | ClawHub에서 설치 가능한 커뮤니티 스킬 |
| Workspace | 사용자가 직접 작성한 프로젝트별 스킬 |

### 주요 내장 도구

- **Shell**: 명령어 실행
- **FileSystem**: 파일 읽기/쓰기/편집
- **Browser**: Puppeteer 기반 웹 자동화 (Semantic Snapshots 사용)
- **Cron**: 예약 작업 실행
- **Calendar**: 일정 관리 연동

### 스킬 로딩 방식

스킬은 **동적 로딩**되므로, 사용하지 않는 스킬은 컨텍스트 윈도우 비용을 소모하지 않습니다. 시스템 프롬프트에 활성 스킬의 프롬프트만 주입됩니다.

### Semantic Snapshots (브라우저 혁신)

스크린샷 대신 **접근성 트리(Accessibility Tree)**를 파싱하여 웹 페이지를 이해합니다. 이 방식은 토큰 비용을 절감하고 정확도를 높입니다.

---

## 6. 메모리 시스템 (Memory)

영속적 저장소를 통해 컨텍스트, 사용자 선호도, 장기 대화 이력을 유지합니다.

- **형식**: 주로 Markdown 파일 기반
- **저장 위치**: 로컬 디바이스 (로컬 퍼스트 원칙)
- **기능**: 장기 대화 이력, 사용자 선호도, 학습된 패턴 유지

---

## 7. Lobster 워크플로우 셸

**Lobster**는 OpenClaw 네이티브 워크플로우 엔진으로, 스킬/도구를 **조합형 파이프라인**으로 구성합니다.

### 핵심 개념

```
스킬 A → 스킬 B → [승인 게이트] → 스킬 C → 결과
```

### 주요 특성

| 특성 | 설명 |
|------|------|
| One Call | 여러 도구 호출을 단일 Lobster 호출로 통합 |
| Approval Gates | 부작용 있는 작업(이메일 전송, 댓글 게시)에 명시적 승인 요구 |
| Resumable | 중단된 워크플로우를 토큰으로 재개 가능 |
| Deterministic | 결정론적 실행으로 예측 가능성 확보 |
| Token-Efficient | 매번 LLM 쿼리 대신 사전 정의된 파이프라인 실행 |

---

## 8. 보안 모델

### 실행 환경

| 시나리오 | 실행 방식 |
|----------|-----------|
| 메인 세션 (소유자) | 호스트에서 직접 실행 (Full Access) |
| 그룹/채널 세션 | 세션별 Docker 샌드박스 (`sandbox.mode: "non-main"`) |
| 알 수 없는 발신자 | 페어링 코드 발급 → 승인 후 허용 목록 등록 |

### 주의사항

- 기업 환경에서 잘못 설정 시 보안 위험 존재
- 스킬의 서플라이 체인 리스크 (ClawHub VirusTotal 연동으로 일부 완화)
- 이메일, 캘린더 등 민감 서비스 접근 권한 필요

---

## 9. 설치 및 설정

```bash
# 설치
npm install -g openclaw@latest

# 온보딩 (게이트웨이, 워크스페이스, 채널, 스킬 설정)
openclaw onboard --install-daemon
```

### 기본 설정 파일

```json
// ~/.openclaw/openclaw.json
{
  "agent": {
    "model": "anthropic/claude-opus-4-6"
  }
}
```

### 릴리스 채널

| 채널 | 설명 | npm dist-tag |
|------|------|-------------|
| Stable | 태그 릴리스 (`vYYYY.M.D`) | `latest` |
| Beta | 프리릴리스 태그 | `beta` |
| Dev | main 브랜치 HEAD | `dev` |

---

## 10. Deep Agents와의 비교

| 항목 | OpenClaw | Deep Agents (이 리포지토리) |
|------|----------|--------------------------|
| 언어 | TypeScript | Python |
| 프레임워크 | 자체 구현 | LangChain / LangGraph |
| 주 인터페이스 | 메시징 플랫폼 (WhatsApp, Telegram 등) | CLI / TUI (Textual) |
| 아키텍처 | Gateway 중심 5계층 | 미들웨어 기반 그래프 |
| 에이전트 실행 | Lane Queue 직렬화 | LangGraph 상태 머신 |
| 스킬 시스템 | ClawHub 마켓플레이스 | YAML 기반 로컬 스킬 |
| 샌드박스 | Docker 기본 내장 | Modal, Daytona, Runloop |
| 메모리 | Markdown 파일 기반 | SQLite 체크포인트 |
| 워크플로우 엔진 | Lobster (네이티브 파이프라인) | LangGraph 그래프 |

---

## 참고 자료

- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw 공식 문서](https://docs.openclaw.ai)
- [Agent Loop 문서](https://docs.openclaw.ai/concepts/agent-loop)
- [Lobster 워크플로우](https://docs.openclaw.ai/tools/lobster)
- [ClawHub (스킬 디렉토리)](https://github.com/openclaw/clawhub)
- [Lobster (워크플로우 셸)](https://github.com/openclaw/lobster)
- [DigitalOcean - What is OpenClaw?](https://www.digitalocean.com/resources/articles/what-is-openclaw)
- [OpenClaw Wikipedia](https://en.wikipedia.org/wiki/OpenClaw)
