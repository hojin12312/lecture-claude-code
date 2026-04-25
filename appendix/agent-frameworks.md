# 에이전트 프레임워크 지형도 — Claude Code를 기준점으로

> 연관 문서: [Claude Code 3대 설정 축](./claude-code.md),
> [API 비용 최적화](./cost-optimization.md),
> [RAG 아키텍처](./rag-architecture.md),
> [LLM 평가](./llm-evaluation.md),
> [경량 모델의 빠른 발전](./small-model-advancement.md),
> [리즈닝 모델의 딜레마](./reasoning-models.md),
> [파라미터 착시: KV 캐시의 진실](./kv-cache.md),
> [안전한 AI는 더 약한 AI인가?](./safety-alignment.md)

[README §5](../README.md#5-하네스가-체급을-이긴다)에서 **Claude Code를 오케스트레이터로 삼고 외부 LLM 노드를 직접 연결**하는 패턴을 봤습니다. 그 외부 노드를 "맨손으로" 짤 수도 있지만, 업계에는 이미 성숙한 **에이전트 프레임워크**들이 있습니다.

2026년 4월 기준 10여 종의 프레임워크가 동시에 성숙 단계에 있습니다. 이 문서는 **Claude Code 사용자 관점에서** 지도처럼 펼쳐보고, 언제 어느 프레임워크를 고를지의 판단 기준을 정리합니다.

---

## 1. 에이전트 프레임워크란 — 하네스의 재사용 가능한 구현

[본문 §1](../README.md#1-뇌와-몸-에이전트의-해부학)에서 정의한 구조를 다시 꺼내면:

```
LLM (뇌) + Harness (몸) = Agent
```

하네스가 담는 것: 도구 호출 루프, 상태 관리, 에러 처리, 권한 제어, 관측성, 서브에이전트 스폰, 프롬프트 체이닝.

이걸 매번 맨손으로 짤 수도 있지만 반복이 심합니다. **에이전트 프레임워크**는 "자주 쓰는 하네스 패턴"을 라이브러리로 묶은 것입니다 — 개발자는 도구 정의·역할·흐름만 선언하고 루프·상태·에러는 프레임워크가 처리합니다.

- **Claude Code = 완제품 하네스** (터미널·IDE용 에이전트 CLI)
- **프레임워크 = 부품 하네스** (여러분 앱에 삽입할 SDK)

같은 뇌(Claude·GPT·Gemini)를 쓰더라도 어떤 몸에 태우느냐에 따라 결과물이 달라집니다.

---

## 2. 분류 축 — 프레임워크를 어떻게 구분할까

2026년 프레임워크 지형을 읽는 세 축:

### 2.1 아키텍처 패턴

| 패턴 | 비유 | 대표 프레임워크 |
|---|---|---|
| **Single-agent loop** | 한 명이 알아서 도구 돌리며 해결 | Claude Agent SDK · OpenAI Agents SDK |
| **Graph / State machine** | 흐름도를 코드로 그림 | LangGraph · Microsoft Agent Framework (graph workflow) |
| **Crew / Role-based** | 역할 분담된 팀 | CrewAI |
| **Conversation / Group chat** | 회의실에서 토론 | AG2 (AutoGen) · MS Agent Framework (group chat) |
| **Handoff** | 담당자에게 바통 넘김 | OpenAI Agents SDK |

실제 프레임워크는 여러 패턴을 동시 지원합니다. 예를 들어 Microsoft Agent Framework는 sequential·concurrent·handoff·group chat·Magentic-One을 모두 가집니다.

### 2.2 추상화 수준

```
  [High abstraction — quick start]
    CrewAI  ·  LangChain 1.0 create_agent  ·  Bedrock Agents
                        |
                        |  (같은 엔진이 밑에서 돎)
                        v
  [Low level — full control]
    LangGraph  ·  AutoGen (core API)  ·  raw Anthropic SDK
```

고수준은 **빠른 시작**, 저수준은 **세밀 제어**. 같은 벤더의 고/저수준이 존재하기도 합니다(LangChain/LangGraph, MS Agent Framework/내부 그래프).

### 2.3 벤더 결합도

| 결합도 | 모델 선택 | 대표 |
|---|---|---|
| Vendor-neutral | 아무 모델 | LangGraph · CrewAI · Pydantic AI · AG2 |
| Vendor-first (with adapter) | 자사 모델 1st-class + 타사 2nd-class | OpenAI Agents SDK (v0.14 any-LLM) · Claude Agent SDK |
| Stack-bound | 특정 클라우드 스택 | MS Agent Framework (Azure) · Bedrock Agents (AWS) · Google ADK (GCP) |

이 세 축을 쓰면 **"어떤 패턴 × 어떤 추상화 × 어떤 스택"** 에서 내 프로젝트가 서 있는지 위치가 잡힙니다.

---

## 3. Claude Code / Claude Agent SDK — 기준점

### 3.1 두 제품의 관계

같은 엔진, 다른 프론트엔드입니다:

```
+------------------------------------------------+
|           Same Core Engine                     |
|  (harness loop + subagent + CLAUDE.md + MCP)   |
+--------+-----------------------+---------------+
         |                       |
         v                       v
  [Claude Code]           [Claude Agent SDK]
  CLI / IDE                Python / TypeScript
  End-user agent           Embed in your app
```

Claude Code에서 익힌 **CLAUDE.md 계층, Skills, Hooks, 서브에이전트, MCP** 개념이 그대로 SDK에 옮겨집니다. 새로 배울 거의 없음.

### 3.2 SDK로 옮겨갈 때의 핵심 객체 (2026-04)

- `ClaudeAgentOptions` — 시스템 프롬프트·도구·권한·**skills**를 담는 설정 객체
- `SessionStore` 프로토콜 — 5개 메서드로 세션 저장/복원. Python·TS 동등 구현
- `list_subagents()` / `get_subagent_messages()` — 서브에이전트 transcript 감사
- `taskBudget` — API 측 토큰 예산. [비용 최적화 §6](./cost-optimization.md#6-추론-모델의-숨겨진-비용)에서 본 thinking 과금 제어를 SDK에 직접 노출
- `agentProgressSummaries` — 긴 작업에서 주기적 AI 요약 방출
- `mcp_set_servers`의 per-tool `permission_policy` — 원격 MCP 서버의 도구별 권한 제어

### 3.3 강점과 약점

**강점**
- Claude 모델 최적화 — thinking·prompt caching·vision이 자동 활용
- 하네스 자산 이식 제로 — `CLAUDE.md` 한 벌이 CLI와 SDK 양쪽에서 동작
- **Claude Code에서 디자인하고 SDK로 배포**하는 파이프라인이 자연스러움

**약점**
- Claude 모델 전용 — 타사 모델 못 붙임 ([비용 최적화 §4](./cost-optimization.md#4-모델-라우팅--작은-모델에-먼저-보내라)의 라우팅에서 nano로 초안→Opus로 검수 같은 이종 모델 파이프라인을 Agent SDK **단독으로는 구성 불가**)
- LangGraph·CrewAI 대비 커뮤니티·공개 레퍼런스 적음
- 엔터프라이즈 콘솔·중앙 관리 대시보드 없음

### 3.4 이 강의에서의 위치

**Claude Code·Agent SDK는 "기본 선택지"** 입니다. 특별한 이유가 없다면 여기서 출발하고, 부족한 지점에서만 다른 프레임워크를 고려합니다. 아래 섹션들은 "어떤 부족이 왔을 때 어디로 갈지"의 지도입니다.

---

## 4. LangGraph + LangChain 1.0 — 가장 많이 쓰이는 범용 엔진

### 4.1 두 제품의 관계

2026년의 중요한 변화: **LangChain 1.0의 `create_agent()`가 LangGraph 런타임 위에서 돌아갑니다.** 즉 두 제품은 경쟁이 아니라 한 스택의 상·하위 레이어입니다:

```
+---------------------------------------------+
|  LangChain 1.0                              |
|    create_agent(model, tools, middleware)   |  <- high-level wrapper
+---------------------------------------------+
|  LangGraph 1.x                              |
|    StateGraph / Node / Edge / Checkpoint    |  <- low-level runtime
+---------------------------------------------+
```

- **LangChain 1.0** — "표준 ReAct 에이전트 + middleware" 빠른 시작
- **LangGraph 1.1.8** (2026-04-17) — 그래프 노드/엣지를 직접 그리는 세밀 제어. `version="v2"` opt-in으로 타입 안전 stream/invoke 제공

### 4.2 LangGraph의 핵심 개념

- **Node** = 계산 단계 (함수 또는 LLM 호출)
- **Edge** = 흐름 (조건부·병렬·루프)
- **State** = 그래프 전체가 공유하는 타입드 딕셔너리
- **Checkpoint / Durable state** (v1.0 도입) — 서버 재시작·인터럽트 발생 시 자동 복구
- **Deep Agent templates** (v1.1.3) — planning + filesystem + subagent가 번들된 시작점
- **Distributed runtime** (v1.1.3) — CLI 기반 분산 실행

### 4.3 Middleware (LangChain 1.0의 시그니처)

에이전트 루프의 **고정된 hook 포인트**에 행동을 주입합니다:

```python
create_agent(model, tools, middleware=[
    summarizationMiddleware(),     # 긴 대화 자동 요약
    humanInTheLoopMiddleware(),    # 민감 도구 호출 전 사람 승인
    piiRedactionMiddleware(),      # 개인정보 자동 마스킹
])
```

개념적으로 Claude Code의 **Hooks**(PreToolUse·PostToolUse)와 유사합니다. 차이: LangChain middleware는 **에이전트 루프 내부**에 끼워넣는 코드 레벨 hook, Claude Code Hooks는 **셸 명령 실행**을 트리거하는 harness 레벨 hook. → [Claude Code 부록 §4 Hooks](./claude-code.md#4-hooks--이벤트-기반-자동화)

### 4.4 강점·약점

**강점**
- **관측성 1위** — LangSmith로 노드별·토큰별 트레이싱. 프로덕션 디버깅 SOTA
- 생태계 최대 — 200+ 도구 통합, 50+ 모델 프로바이더
- 엔터프라이즈 레퍼런스 (Klarna·Uber·J.P. Morgan)

**약점**
- 학습 곡선 가파름. 단순 에이전트도 수십 줄 코드
- 주말 프로토타이핑에 부담

### 4.5 언제 고르나

- 복잡한 분기·루프가 많은 상태 머신
- 장수 에이전트 (세션 재개·체크포인트 필수)
- LangSmith 관측성이 비즈니스 요구

---

## 5. CrewAI — 역할 기반 멀티에이전트의 대표

### 5.1 4개 개념으로 끝

- **Agent** — 역할(role)·목표(goal)·배경(backstory)·도구(tools)를 가진 "직원"
- **Task** — 설명·기대 산출물을 명시한 "업무 지시서"
- **Crew** — Agent + Task의 조합. 실행 단위
- **Process** — Crew 실행 방식: `sequential` · `hierarchical` · `consensual`(투표)

```
Crew
 |
 +-- Agent "Researcher"  (role: 연구원)  -> Task: 자료 수집
 +-- Agent "Analyst"     (role: 분석가)  -> Task: 패턴 추출
 +-- Agent "Writer"      (role: 작가)    -> Task: 초안 작성
 +-- Agent "Editor"      (role: 편집자)  -> Task: 검수
```

비개발자 스테이크홀더에게 **"에이전트 4명 팀"** 이라는 비유로 설명하기 쉽다는 게 CrewAI의 가장 큰 자산입니다.

### 5.2 Crews vs Flows

- **Crews** — 자율성 중심. 에이전트들이 알아서 협업
- **Flows** — 엔터프라이즈 프로덕션. 명시적 흐름·분기·조건 (LangGraph에 더 가까움)

### 5.3 2026 주요 업데이트 (v1.14.2 기준, 2026-04-17)

- **Agent skills** — Claude Code Skills와 유사한 특화 기능 주입
- **OpenAI-compatible provider native 지원** — OpenRouter, DeepSeek, Ollama, vLLM, Cerebras
- **Qdrant Edge memory backend** — 로컬 벡터 DB 기반 장기 기억
- **Hierarchical memory isolation** — 에이전트별 메모리 분리
- **Visual Studio** — 드래그&드롭 디자이너 (비기술자 협업)
- **Checkpoint TUI** (v1.14.2a2) — 실행 트리를 TUI에서 탐색·fork·입출력 편집
- **Reasoning/cache token 추적** — LLM 호출당 thinking 토큰·cache creation 토큰 분리 집계

### 5.4 강점·약점

**강점**
- 가장 빠른 프로토타이핑 (수 시간 내 멀티에이전트 데모)
- 역할 비유가 직관적
- LangChain 의존성 제로 (가볍고 독립)

**약점**
- 고수준 추상화 → 트러블슈팅 어려움 ("블랙박스")
- Mission-critical에 제어권 부족
- 관측성이 LangGraph만 못함

### 5.5 언제 고르나

- 컨설팅·기획 자동화 (연구→분석→작성→검수 파이프라인)
- 사내 RPA형 에이전트
- 비개발자 참여 프로토타입

---

## 6. Microsoft Agent Framework — AutoGen + Semantic Kernel 통합 후속

### 6.1 배경: 두 프레임워크의 합병

Microsoft는 오랜 기간 두 개의 중복된 에이전트 프레임워크를 유지했습니다:

- **Semantic Kernel** — 엔터프라이즈 기능(타입·필터·세션) 중심
- **AutoGen** — 대화 기반 멀티에이전트 실험 플랫폼

2025-10 통합 발표 → **2026-04-03 Microsoft Agent Framework 1.0 정식 출시**. .NET + Python 동시 LTS.

```
Semantic Kernel v1.x  ----->  MS Agent Framework 1.0  (2026-04-03 GA)
                        /
AutoGen v0.4 actor model ----  (v0.7.x는 유지보수만)
                        
AG2 (Apache 2.0 fork)   ----  별도 경로로 커뮤니티 주도 진화
```

### 6.2 멀티에이전트 오케스트레이션 패턴

하나의 프레임워크에 **5개 오케스트레이션 패턴 내장**:

- Sequential — 순차
- Concurrent — 병렬
- Handoff — 전달 (OpenAI 스타일)
- Group chat — 대화방 (AutoGen 스타일)
- **Magentic-One** — MS Research의 범용 멀티에이전트 패턴

중복이던 두 프레임워크가 통합되면서 **"이 작업엔 이 패턴" 선택지를 한 프레임워크 안에서 제공**하는 것이 MS Agent Framework의 차별화 축입니다.

### 6.3 강점·약점

**강점**
- 엔터프라이즈 준비 완료 — 타입 안전·필터·텔레메트리 1st-class
- .NET·Python 동등 지원 (다른 프레임워크는 대부분 Python 전용)
- Azure Foundry·Teams·Dynamics와 통합

**약점**
- MS 스택 밖 도입 레퍼런스 적음
- v1.0이 2026-04 출시 → 장기 레퍼런스 부족
- AutoGen/SK 사용자는 마이그레이션 필요

### 6.4 언제 고르나

- Azure 중심 엔터프라이즈
- .NET 팀 (다른 프레임워크는 Python 편중)
- Teams·M365·Dynamics 통합 워크플로우

---

## 7. OpenAI Agents SDK — 5개 primitive로 끝

### 7.1 철저히 opinionated한 5개 개념

1. **Agents** — 지시문 + 도구 + 모델 번들
2. **Handoffs** — 다른 에이전트로 작업 이관. LLM 관점에서는 도구로 노출 (`transfer_to_refund_agent`)
3. **Guardrails** — 입력·출력·도구 단계 병렬 검증. **Tripwire 발동 시 에이전트 실행 자체 차단** (토큰·도구 호출 발생 전 막음)
4. **Sessions** — 대화 상태 영속화
5. **Tracing** — 내장 트레이싱

"약 30줄로 멀티에이전트 트리아지"라는 공식 예시가 철학을 보여줍니다 — 간결·opinionated·즉시 시작.

### 7.2 Handoff 패턴 특화

LangGraph의 graph도, CrewAI의 crew도, AutoGen의 group chat도 본질적으로 "에이전트 간 제어 이관"이지만, OpenAI SDK는 **handoff를 모델이 호출하는 일반 도구로 노출**하는 방식을 택했습니다:

```
Triage Agent
   |
   | LLM이 tool_calls: [transfer_to_refund_agent] 생성
   v
Refund Agent   (SDK가 가로채 대상 에이전트로 이관)
```

도구와 핸드오프를 같은 메커니즘(LLM이 내는 tool call)으로 통일한 것이 이 SDK의 깔끔함입니다.

### 7.3 2026 업데이트 (v0.14.3, 2026-04-20)

- **any-LLM 어댑터** — OpenAI 외 모델도 호출 가능 (이전엔 사실상 OpenAI 전용)
- **MCP resource 지원** — MCP 서버의 resource 프리미티브 직접 소비
- **Sandbox Execution** (v0.14, 2026-04) — 에이전트가 controlled 환경 안에서 파일 검사·명령 실행·코드 편집 수행
- **Model-Native Harness** — 파일과 도구를 컴퓨터 레벨에서 다루는 내장 harness. Python 선행, TypeScript 후속
- **Long-horizon task** 지원 강화
- Opt-in retry 정책
- Realtime model → `gpt-realtime-1.5`

### 7.4 강점·약점

**강점**
- 간결 — 최소 코드로 빠른 출발
- OpenAI built-in 도구(web search·file search·computer use) 1st-class
- 음성 실시간 (Realtime API) 강력

**약점**
- OpenAI 생태계 종속 강함 (any-LLM이 있지만 OpenAI 일급)
- 복잡 상태 머신에 부적합
- 모델 유연성 제한적

### 7.5 언제 고르나

- OpenAI commit 팀
- 빠른 POC
- 실시간 음성 에이전트

---

## 8. 그 외 주요 프레임워크 — 한 문단 요약

### Pydantic AI (v1.85, 2026-04-22)

Pydantic 팀이 만든 **타입 안전 제일주의** 프레임워크. 도구 입출력·에이전트 의존성을 Pydantic 모델로 검증해 컴파일 타임에 에러를 잡습니다. 25+ 공급자 지원, MCP·A2A 상호운용, AgentSpec(YAML/JSON 선언적 에이전트)·Capabilities(조합 가능한 행동 단위) 체제. **금융·의료·법률** 등 출력 포맷 정합성이 생명인 도메인에 적합. Python 전용. 마이너 릴리스 주기가 매우 빠른 편(월 10+) — 프로덕션 고정 시 버전 pin 필수.

### Google ADK (v2.0-alpha / v1.28)

**유일하게 Python·TypeScript·Go·Java 4개 언어 1st-class** 지원. Vertex AI·Cloud Run 네이티브 통합, built-in 평가 프레임워크. Sequential/Parallel/Loop 3종 workflow + LLM-driven routing. 다중 언어 팀·GCP 중심 배포에 적합. 2026 업데이트로 Anthropic streaming·Slack·BigQuery·Spanner 통합 추가.

### Amazon Bedrock Agents

AWS 관리형. IAM·VPC·HIPAA·SOC/ISO **컴플라이언스 인증 완비**. 자연어 지시문만으로 Lambda·S3·DynamoDB·Knowledge Base 오케스트레이션. 오케스트레이션 코드 불필요. AWS 생태계 종속·콘솔 의존이 트레이드오프. **규제 산업 AWS 엔터프라이즈**에 적합.

### AG2 (autogen.beta)

AutoGen의 커뮤니티 포크(Apache 2.0). **AG2 Beta**로 ground-up 재설계 — streaming, 이벤트 기반, 멀티 공급자, DI, typed tools, first-class testing. 대화 기반 패러다임 유지. 연구·실험에 적합, 프로덕션 부족.

---

## 9. 선택 매트릭스

```
                [Abstraction]

 High   CrewAI         LangChain 1.0    Bedrock Agents
        (role crew)    create_agent     (managed)
        ^
        |              MS Agent Framework
        |              (workflows)
        |
        |              OpenAI Agents SDK
        |              (handoff)
        |
        |              Claude Agent SDK
        |              (single loop + MCP)
        |
        |              Pydantic AI
        |              (type-safe)
 Low    |              LangGraph        Google ADK
        |              (graph)          (multi-lang)
        +----------------------------------------->
          Vendor-neutral      Stack-bound
                [Vendor Coupling]
```

### 상황별 1순위

| 상황 | 권장 |
|---|---|
| Claude 중심 · CLAUDE.md 이식 | **Claude Agent SDK** |
| 복잡 프로덕션 상태 머신 | **LangGraph** |
| 빠른 프로토타이핑 · 비개발자 협업 | **CrewAI** |
| OpenAI commit · 간단 핸드오프 | **OpenAI Agents SDK** |
| 타입 안전 · 금융/의료/법률 | **Pydantic AI** |
| .NET · Azure 엔터프라이즈 | **MS Agent Framework** |
| 다중 언어 · Google Cloud | **Google ADK** |
| AWS 규제 산업 | **Bedrock Agents** |
| 연구 · 대화 기반 실험 | **AG2** |

### 의사결정 순서

1. **모델 선택이 고정되어 있는가?** → 그 벤더 1st-party SDK 먼저 검토 (Claude → Agent SDK, OpenAI → Agents SDK)
2. **벤더 중립이 필수인가?** → LangGraph · CrewAI · Pydantic AI
3. **상태 머신이 복잡한가?** → LangGraph (graph) 또는 MS Agent Framework (workflow)
4. **빨리 보여줘야 하는가?** → CrewAI (role crew) 또는 LangChain 1.0 (`create_agent`)
5. **출력 스키마 정합성이 생명인가?** → Pydantic AI
6. **회사 스택이 Azure/AWS/GCP로 고정되어 있는가?** → 해당 스택 관리형

---

## 10. Claude Code를 오케스트레이터로 — 통합 패턴

[README §5 하네스가 체급을 이긴다](../README.md#5-하네스가-체급을-이긴다)에서 본 작성-검수 루프 패턴은 "Claude Code 내부에서 여러 LLM 노드를 직접 설계"하는 접근이었습니다. 이 패턴을 외부 프레임워크와 결합하면 다음 세 축이 됩니다:

### 10.1 Claude Code 단독 (기본)

서브에이전트·Agent SDK만 사용. 모든 게 Anthropic 스택 안에서 해결.

```
[Claude Code Main]
   +-- Agent tool --> [Sub-agent Haiku] (탐색)
   +-- Agent tool --> [Sub-agent Opus]  (검수)
```

**적합**: Claude 하나로 커버되는 작업. 스택 단순성 우선.

### 10.2 Claude Code → MCP로 외부 프레임워크 호출

LangGraph·CrewAI로 만든 복잡 파이프라인을 **MCP 서버로 래핑**해서 Claude Code가 도구처럼 부릅니다.

```
[Claude Code]
    |
    v  (MCP tool call)
[LangGraph MCP server]
    |
    v
[complex graph with loops, branches, checkpoints]
```

**적합**: Claude Code가 대화 오케스트레이터, LangGraph가 백엔드 상태 머신. [RAG 아키텍처 §9](./rag-architecture.md#9-mcp를-통한-rag-연결)에서 본 RAG 분리 패턴과 같은 원리.

### 10.3 외부 프레임워크가 Claude 모델 호출

LangGraph·CrewAI·Pydantic AI 내부에서 Claude API(또는 Bedrock Claude)를 모델로 선택. 프레임워크 쪽 오케스트레이션 + Claude의 추론 품질.

**적합**: 프레임워크 기능(LangSmith 관측성·CrewAI 역할 설계)을 쓰고 싶되 모델은 Claude로.

### 선택 기준

| 우선순위 | 패턴 |
|---|---|
| 스택 단순성 · Claude Code 경험 극대화 | **10.1 단독** |
| 복잡 백엔드 분리 · Claude Code는 프런트 | **10.2 MCP 래핑** |
| 프레임워크 생태계 + Claude 품질 | **10.3 Claude를 모델로 주입** |

**비용 관점**: 10.2·10.3은 두 시스템 사이에 컨텍스트가 오가므로 캐싱 경계 설계가 중요합니다. → [비용 최적화 §2](./cost-optimization.md#2-프롬프트-캐싱--반복되는-입력의-90-할인)

---

## 11. 2026 수렴 표준 — MCP · A2A · OpenTelemetry

2026년의 큰 흐름은 **프레임워크 간 전환 비용이 낮아진 것**입니다. 세 표준이 수렴 중:

### 11.1 MCP (Model Context Protocol)

- Anthropic 2024-11 발표 → 2026년 업계 표준화
- **지원**: Claude Agent SDK · OpenAI Agents SDK (v0.14) · LangGraph · Pydantic AI · MS Agent Framework · Google ADK
- 효과: 한 번 작성한 MCP 서버를 여러 프레임워크에서 공유 → 도구 재사용

### 11.2 A2A (Agent-to-Agent)

Google 주도 제안. 서로 다른 프레임워크의 에이전트끼리 통신하는 프로토콜. LangGraph·CrewAI·Pydantic AI 채택 중.

### 11.3 OpenTelemetry

관측성 표준. AutoGen v0.4가 1st-class로 도입 → Claude Agent SDK·LangGraph·Pydantic AI 공통 지원. **프레임워크 독립 트레이싱**이 가능해짐.

> OTel 스팬은 eval 플랫폼(Braintrust·Langfuse·Arize Phoenix·Inspect AI)이 그대로 읽습니다. 프레임워크를 교체해도 eval 파이프라인이 유지되는 이유. → [LLM 평가 §7 프레임워크 지형](./llm-evaluation.md#7-2026-평가-프레임워크-지형)

### 의미

프레임워크 선택이 **5년 락인이 아닌 상황**이 됐습니다. 도구는 MCP로, 에이전트 간 통신은 A2A로, 관측성은 OpenTelemetry로 통일되면 프레임워크는 **오케스트레이션 엔진만** 맡습니다. 엔진을 교체해도 도구·관측·통신은 재사용.

---

## 12. 설계 체크리스트

새 에이전트 프로젝트 시작 시 점검 순서:

1. **모델이 고정되어 있나?** Claude → Agent SDK, OpenAI → Agents SDK, 벤더 중립 → LangGraph/CrewAI/Pydantic AI
2. **복잡도** — 단순 루프면 Agent SDK·OpenAI SDK, 분기 많은 상태 머신이면 LangGraph, 역할 기반 협업이면 CrewAI
3. **타입 안전 요구** — 금융·의료·법률이면 Pydantic AI 1순위
4. **프로덕션 관측성** — LangSmith 필요하면 LangGraph, OpenTelemetry로 충분하면 프레임워크 선택 폭 넓음
5. **팀 스택** — .NET이면 MS Agent Framework, Java/Go 섞이면 Google ADK
6. **규제 산업** — AWS Bedrock Agents 또는 Azure의 MS Agent Framework
7. **스택 단순성 vs 생태계** — Claude Agent SDK 단독 vs LangGraph 생태계 중 선택
8. **MCP 서버 분리** — RAG·사내 도구는 MCP로 분리해 프레임워크 교체 여지 확보
9. **비용 관점** — 이종 모델 라우팅이 필요하면 Claude Agent SDK 단독 불가. LangGraph·CrewAI·Pydantic AI가 유리 → [비용 최적화 §4](./cost-optimization.md#4-모델-라우팅--작은-모델에-먼저-보내라)

### 자주 틀리는 것

- **"가장 인기 있는 거 써야지"** — 126k stars LangGraph가 weekend PoC에는 오버킬. 복잡도에 맞게
- **프레임워크 먼저 고르고 모델 나중** — 반대 순서가 맞음. 모델이 도구 포맷·caching 행동을 결정
- **MCP 없이 시작** — 프레임워크 전환 여지를 스스로 닫음. 도구는 처음부터 MCP 서버로 분리 권장
- **관측성 후순위** — 프로덕션 전환 시점에 "트레이싱 없어서 디버깅 못 함"이 가장 흔한 지연 원인
- **Eval 없이 프로덕션 전환** — 에이전트는 평가 차원이 하나가 아님(task completion·tool call accuracy·pass^k·cost). 단일 숫자로 판단 금지. → [LLM 평가 §6 에이전트 평가의 특수성](./llm-evaluation.md#6-에이전트-평가의-특수성)

---

## 네비게이션

**왔던 길** — 이 부록에 도달하는 전형적인 경로

- "Claude Agent SDK / LangGraph / CrewAI / MS Agent Framework 뭘 골라야?" → 여기
- "MCP·A2A·OpenTelemetry가 왜 2026년에 표준으로 수렴했지?" → 여기
- "에이전트 평가 축이 LLM 호출 평가와 뭐가 달라?" → [LLM 평가](./llm-evaluation.md) → 여기

**갈 길** — 이어 읽으면 자연스러운 3개

1. [Claude Code 3대 설정 축](./claude-code.md) — 3대 설정 축이 Claude Agent SDK에 그대로 이식되는 원리
2. [LLM 평가](./llm-evaluation.md) — 에이전트 평가 특수성 (task completion·tool call accuracy·pass^k·cost)
3. [API 비용 최적화](./cost-optimization.md) — 모델 라우팅·caching을 프레임워크 계층에서 구현

---

## 연관 문서

- [Claude Code 3대 설정 축](./claude-code.md) — CLAUDE.md·Hooks·Skills·MCP의 개념. Claude Agent SDK에 그대로 이식되는 설정 자산
- [API 비용 최적화](./cost-optimization.md) — 모델 라우팅·캐싱. 프레임워크가 이종 모델을 섞을 때 비용 구조
- [RAG 아키텍처](./rag-architecture.md) — §9 MCP로 RAG 분리 패턴. 섹션 10.2와 같은 원리
- [LLM 평가](./llm-evaluation.md) — 프레임워크를 고른 뒤 "실제로 좋은지" 측정하는 eval 루프. OpenTelemetry 스팬이 eval 파이프라인의 입력
- [경량 모델의 빠른 발전](./small-model-advancement.md) — 작은 모델을 어디에 태울지의 근거
- [리즈닝 모델의 딜레마](./reasoning-models.md) — 에이전트 루프에 reasoning 모델을 넣을 때 thinking 비용·지연 관리
- [파라미터 착시: KV 캐시의 진실](./kv-cache.md) — 멀티턴 에이전트에서 KV 누적이 처리량 한계를 만드는 이유
- [안전한 AI는 더 약한 AI인가?](./safety-alignment.md) — 에이전트에 도구를 쥐여준 이후 권한·감사 로그·인간 개입 지점의 alignment 설계
- [notes/agent-frameworks-2026.md](../notes/agent-frameworks-2026.md) — 각 프레임워크 버전·기능·출처 참조표
