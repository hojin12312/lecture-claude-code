# Agent Frameworks — 2026-04 Reference

에이전트 프레임워크 버전·기능·라이선스·패턴 참조용 notes. `appendix/agent-frameworks.md` 작성의 근거 자료.

마지막 조사: 2026-04-25

---

## 1. 개요 — 2026-04 기준 프레임워크 지형

메이저 프레임워크 약 10종이 동시 성숙 단계. **3대 흐름**:

1. **벤더 SDK** — Claude Agent SDK (Anthropic), OpenAI Agents SDK, Google ADK
2. **오픈소스 범용** — LangGraph + LangChain, CrewAI, Pydantic AI, AG2
3. **엔터프라이즈 통합** — Microsoft Agent Framework, Amazon Bedrock Agents

**공통 수렴점 (2026년 큰 변화)**:
- **MCP (Model Context Protocol)** — 도구 상호운용 표준. 거의 전 프레임워크가 클라이언트·서버 지원
- **A2A (Agent-to-Agent)** — 에이전트 간 통신 표준
- **OpenTelemetry** — 관측성 표준. AutoGen v0.4·Claude Agent SDK·LangGraph·Pydantic AI 공통 지원

---

## 2. Claude Agent SDK (Anthropic)

### 기본 정보

- **최신 버전**: 2026-04 기준 활성 개발 중 (Python + TypeScript 병행 릴리스)
- **리포지토리**: `anthropics/claude-agent-sdk-python`, `anthropics/claude-agent-sdk-typescript`
- **라이선스**: MIT
- **관계**: Claude Code와 동일 엔진. Claude Code가 CLI/IDE 프론트엔드라면 Agent SDK는 **프로그래매틱 인터페이스**
- **문서**: https://code.claude.com/docs/en/agent-sdk/overview

### 핵심 개념 (2026-04)

- `ClaudeAgentOptions` — 에이전트 설정 객체 (시스템 프롬프트, 도구, 권한, skills)
- `skills` 옵션 — 메인 세션에 활성화할 skills 목록 (`"all"` / 리스트 / `[]`)
- `SessionStore` 프로토콜 — 5개 메서드로 세션 상태 저장/복원 (Python·TS 동등 지원)
- `list_subagents()` / `get_subagent_messages()` — 서브에이전트 transcript 검사
- `taskBudget` — API 측 토큰 예산 제어
- `agentProgressSummaries` — 주기적 AI 진행 상황 요약
- `enableChannel()` (TS) — MCP 채널 동적 활성화
- `mcp_set_servers`의 `permission_policy` — 원격 MCP 서버 per-tool 권한 정책

### 2026 주요 업데이트

- Opus 4.7 지원
- OpenTelemetry 선택적 지원 (양쪽 SDK)
- Cascading session deletion — `delete_session()`이 서브에이전트 transcript 디렉터리까지 삭제
- TypeScript SDK에서 SessionStore full parity 달성

### 아키텍처 패턴

- **단일 에이전트 루프** + **서브에이전트 위임** + **MCP 도구**
- Claude Code의 `Agent`/`Task` 도구를 SDK로 직접 호출 가능
- 하네스 설계를 Claude Code의 CLAUDE.md·Hooks·Skills 체계로 통일

### 강점

- Claude 모델에 최적화 (thinking·prompt caching·vision 자동 활용)
- Claude Code에서 익힌 설정 자산(CLAUDE.md·Skills·Hooks)이 그대로 SDK로 이식
- 최소 러닝커브 — 기존 Claude Code 사용자에게 학습 비용 제로

### 약점

- Claude 모델 종속 (다른 공급자 모델 못 씀)
- 커뮤니티 규모가 LangGraph·CrewAI보다 작음
- 엔터프라이즈 기능(워크플로우 엔진·중앙 관리 콘솔) 부족

---

## 3. LangGraph + LangChain 1.0

### 기본 정보

- **LangGraph**: v1.1.8 (2026-04-17 릴리스), 126k+ GitHub stars
- **LangChain**: v1.0 (2026년 상반기 마일스톤)
- **라이선스**: MIT
- **개발사**: LangChain Inc.
- **도입 사례**: Klarna, Uber, J.P. Morgan
- **리포지토리**: https://github.com/langchain-ai/langgraph

### 관계

LangChain 1.0부터 **`create_agent()`는 LangGraph 런타임 위에서 돈다**. 즉:

```
[LangChain 1.0]                   [LangGraph 1.x]
  create_agent  <- 고수준 래퍼       graph API  <- 저수준 런타임
  middleware       (빠른 시작)        node/edge/state   (세밀 제어)
```

LangChain은 **"빠르게 표준 ReAct 에이전트 만들기"**, LangGraph는 **"완전 커스텀 상태 머신"**. 하지만 엔진은 같음.

### 핵심 개념

**LangGraph**:
- 노드(Node) = 함수 또는 LLM 호출
- 엣지(Edge) = 조건부 흐름 (분기·루프·병렬)
- 상태(State) = 그래프 전체가 공유하는 타입드 딕셔너리
- **Durable state** (v1.0) — 서버 재시작·인터럽트 복구 자동
- **Deep Agent templates** (v1.1.3) — planning + filesystem + subagent 번들
- **Distributed runtime** (v1.1.3) — CLI 분산 실행 지원
- **Type-safe streaming/invoke** (v1.1.7+, 2026-03-10) — `version="v2"` opt-in 모드. `stream()`/`invoke()` 반환이 GraphOutput 타입, Pydantic/dataclass 자동 coercion. time-travel-with-interrupts·subgraphs 재생 시 stale RESUME 값 재사용 버그 수정.

**LangChain 1.0**:
- `create_agent(model, tools, middleware=[...])` — 한 줄 생성
- **Middleware** — 에이전트 루프의 hook 포인트에 끼워넣는 동작
  - `summarizationMiddleware` — 긴 대화 자동 요약
  - `humanInTheLoopMiddleware` — 사람 승인 삽입
  - `piiRedactionMiddleware` — 개인정보 자동 마스킹

### 강점

- **관측성 1위** — LangSmith 연동으로 노드별·토큰별 트레이싱
- 거대한 생태계 (200+ 툴 통합, 50+ 모델 프로바이더)
- 엔터프라이즈 도입 레퍼런스 최다

### 약점

- 학습 곡선 가파름 (특히 LangGraph 저수준 API)
- 코드가 verbose — 단순 에이전트도 수십 줄
- 주말 프로토타이핑에 부담

### 적합 사례

- 복잡한 상태 머신 (분기·루프가 많은 워크플로우)
- 프로덕션 장수 에이전트 (세션 복구·체크포인트 필수)
- LangSmith로 디버깅·관측 필요

---

## 4. CrewAI

### 기본 정보

- **최신 버전**: v1.14.2 stable (2026-04-17), v1.14.3a2 pre-release (2026-04-21)
- **GitHub stars**: 40,000+ (2026 초 기준, 2026-04 현재 44,600+)
- **라이선스**: MIT
- **리포지토리**: https://github.com/crewAIInc/crewAI
- **개발사**: CrewAI Inc. (독립 회사)

### 핵심 개념

- **Agent** — 역할(role)·목표(goal)·배경(backstory)·도구(tools)를 가진 "직원"
- **Task** — 설명(description)·기대 산출물(expected_output)을 가진 "업무 지시서"
- **Crew** — 여러 Agent + Task의 조합. 실행 단위
- **Process** — Crew 실행 방식
  - `sequential` — 순차 실행 (기본)
  - `hierarchical` — 매니저 에이전트가 워커에게 위임
  - `consensual` — 에이전트 투표

### Crews vs Flows

- **Crews** — 자율성 중심. "알아서 협업하세요"
- **Flows** — 엔터프라이즈 프로덕션. 명시적 흐름·분기·조건

### 2026 주요 업데이트 (v1.12~v1.14.2)

- **Agent skills** — Claude Code의 Skills와 유사한 특화 기능 주입
- **Native OpenAI-compatible providers** — OpenRouter, DeepSeek, Ollama, vLLM, Cerebras, Dashscope 직접 지원
- **Qdrant Edge memory backend** — 로컬 벡터 DB 기반 장기 기억
- **Hierarchical memory isolation** — 에이전트별 메모리 분리
- **Visual Studio** — 드래그&드롭 에이전트 디자이너 (비기술자 협업용)
- **Checkpoint TUI** (v1.14.2a2, 2026-04-10) — 트리 뷰·fork·입출력 편집 지원
- **Deploy validation CLI** (v1.14.2a3, 2026-04-13) — 배포 전 구성 검증
- **Reasoning/cache token tracking** (v1.14.2a2) — LLM 토큰 분류에 reasoning + cache creation 토큰 추가

### 강점

- **가장 빠른 프로토타이핑** — 수 시간 내 멀티에이전트 데모
- 역할 기반 비유가 직관적 (비기술 스테이크홀더 설명 쉬움)
- LangChain 의존성 제거 (완전 독립, 의존성 가벼움)

### 약점

- 고수준 추상화라 트러블슈팅 어려움 ("블랙박스")
- Mission-critical 시스템에 제어권 부족
- 관측성이 LangGraph만 못함

### 적합 사례

- 컨설팅·기획 자동화 (연구원→분석가→작가 파이프라인)
- 사내 RPA형 에이전트
- 비개발자 참여형 프로토타입

---

## 5. Microsoft Agent Framework (AutoGen + Semantic Kernel 후속)

### 기본 정보

- **버전**: 1.0 GA (2026-04-03 정식 릴리스)
- **언어**: .NET + Python 동시 지원 (LTS)
- **라이선스**: MIT
- **개발사**: Microsoft
- **문서**: https://learn.microsoft.com/en-us/agent-framework/overview/

### 역사·분기

AutoGen과 Semantic Kernel이 같은 팀·같은 기능 중복 → 통합 결정 (2025-10):

```
Semantic Kernel (엔터프라이즈, 세션·타입·필터)
      \
       +---> Microsoft Agent Framework (2026-04 v1.0)
      /
AutoGen v0.4 (비동기 actor model, 실험적)
```

- **AutoGen v0.7.x** — stable maintenance line (critical bug·보안 패치만, 신기능 X)
- **AG2** — AutoGen의 커뮤니티 포크 (Apache 2.0). "autogen.beta" 이름으로 리뉴얼
- **Semantic Kernel v1.x** — 유지보수 모드 (향후 지원, 신기능은 MS Agent Framework로)

### 핵심 개념

- **단일 에이전트 추상화** + **서비스 커넥터** (.NET/Python 공통)
- **Middleware hooks** — 에이전트 루프 개입점
- **Agent memory & context providers** — 장·단기 기억 분리
- **Graph-based workflows** — 명시적 실행 경로
- **멀티에이전트 오케스트레이션 패턴**
  - Sequential — 순차
  - Concurrent — 병렬
  - Handoff — 전달 (OpenAI 스타일)
  - Group chat — 대화방
  - **Magentic-One** — MS Research가 만든 일반 목적 멀티에이전트 패턴

### 강점

- **엔터프라이즈 준비 완료** — 타입 안전·필터·텔레메트리 내장
- .NET·Python 동등 지원 → Azure·Windows 스택 기업에 최적
- 2개의 프레임워크(AutoGen 단순성 + SK 엔터프라이즈) 통합

### 약점

- MS 스택 밖에서는 도입 레퍼런스 적음
- 신생 (v1.0이 2026-04 출시) → 장기 레퍼런스 부족
- AutoGen/SK 사용자는 마이그레이션 필요

### 적합 사례

- Azure 중심 엔터프라이즈 에이전트
- .NET 팀 (LangGraph·CrewAI 등은 Python만)
- Teams·M365·Dynamics 통합 워크플로우

---

## 6. OpenAI Agents SDK

### 기본 정보

- **최신 버전**: v0.14.3 (2026-04-20)
- **라이선스**: MIT
- **리포지토리**: https://github.com/openai/openai-agents-python
- **문서**: https://openai.github.io/openai-agents-python/

### 핵심 5개 Primitive

1. **Agents** — 지시문·도구·모델 번들
2. **Handoffs** — 다른 에이전트로 작업 이관 (도구로 노출됨: `transfer_to_refund_agent`)
3. **Guardrails** — 입력·출력·도구 단계의 병렬 검증. Tripwire 발동 시 에이전트 실행 중단
4. **Sessions** — 대화 상태 영속화
5. **Tracing** — 내장 트레이싱

### 2026 주요 업데이트 (v0.13 → v0.14.3)

- **any-LLM 어댑터** — OpenAI 외 모델도 호출 가능 (이전엔 OpenAI 전용)
- **Opt-in retry 정책**
- **MCP resource 지원** — MCP 서버의 resource 프리미티브 소비
- **Session persistence** 강화
- **Realtime model** — `gpt-realtime-1.5`로 업그레이드
- **Sandbox Execution** (v0.14, 2026-04-15 이후) — 에이전트가 파일 검사·명령 실행·코드 편집을 controlled 샌드박스 환경에서 수행
- **Model-Native Harness** (v0.14) — 에이전트가 파일·도구를 computer 수준에서 다루는 harness, native sandbox execution 내장. Python 우선, TypeScript 후속
- **Long-horizon task 지원** — 멀티 시간 규모 작업 대응

### 강점

- **간결** — "약 30줄로 멀티에이전트 트리아지" (공식 문서 예시)
- 철저히 opinionated — 생각 없이 빠르게 시작
- OpenAI의 built-in 도구(web search·file search·computer use) 일급 지원

### 약점

- OpenAI 모델·생태계 종속 강함 (any-LLM 어댑터가 있지만 1st-class는 OpenAI)
- 복잡한 상태 머신에 부적합 (handoff 중심 설계)
- 모델 유연성 제한적

### 적합 사례

- OpenAI 생태계 commit 팀
- 빠른 POC·간단 트리아지 에이전트
- 실시간 음성 (Realtime API 활용)

---

## 7. Pydantic AI

### 기본 정보

- **최신 버전**: v1.85.1 (2026-04-22)
- **언어**: Python 전용
- **라이선스**: MIT
- **개발사**: Pydantic (FastAPI·Pydantic으로 유명)

### 핵심 개념

- **타입 안전** — Pydantic 모델로 도구 입출력·에이전트 의존성 검증. 컴파일 타임 에러 잡음
- **Capabilities** (v1.71+) — 조합 가능한 행동 단위
- **AgentSpec** — YAML/JSON으로 에이전트 선언적 정의
- **Cross-provider Thinking** — 공급자 무관 추론 모드 통일 API
- **Provider-adaptive tools** — 같은 도구를 공급자별 포맷으로 자동 변환
- **25+ 공급자 지원** — OpenAI·Anthropic·Gemini·Cohere·Groq·Mistral·로컬 LLM 등
- **MCP·A2A 상호운용**
- v1.7x→v1.85 사이 마이너가 월 10+ 릴리스 — 리포지토리 커밋 주기가 가장 빠른 프레임워크 중 하나

### 강점

- **타입 안전이 1st-class** — 금융·의료·법률처럼 출력 포맷 정합성이 생명인 도메인에 최적
- 코드 품질 높음 (Pydantic 팀의 엔지니어링 수준)
- Python 생태계와 자연스러운 통합 (FastAPI·SQLAlchemy)

### 약점

- Python 전용
- 타입 중심이라 프로토타이핑에 오버헤드
- 시각 에디터 없음 (코드 퍼스트)

### 적합 사례

- 금융·의료·법률 (스키마 정합성 중요)
- 코드 품질 우선 팀
- Pydantic·FastAPI 기반 기존 스택

---

## 8. Google ADK (Agent Development Kit)

### 기본 정보

- **버전**: v2.0-alpha (그래프 런타임), v1.28 (안정)
- **언어**: Python·TypeScript·Go·Java (유일한 다중 언어 1st-class)
- **라이선스**: Apache 2.0
- **개발사**: Google
- **리포지토리**: https://github.com/google/adk-python

### 핵심 개념

- **Workflow agents** — Sequential·Parallel·Loop 3가지 내장 패턴
- **LLM-driven dynamic routing** — LLM이 상황에 따라 라우팅
- **Task API** (v2.0-alpha) — 에이전트 간 위임
- **Built-in evaluation** — step-by-step 평가 내장

### 2026 업데이트 (v1.28)

- Slack 통합
- BigQuery toolset (공식 마이그레이션)
- Anthropic streaming 지원
- Spanner Admin toolset

### 강점

- **다중 언어** — 유일하게 4개 언어 1st-class
- Vertex AI·Cloud Run 통합
- 내장 평가 프레임워크

### 약점

- 신생 (레퍼런스 적음)
- 언어별 SDK 성숙도 편차
- pre-built 도구 통합 수 적음

### 적합 사례

- 다중 언어 팀
- Google Cloud 네이티브 배포
- Vertex AI 중심 엔터프라이즈

---

## 9. Amazon Bedrock Agents

### 기본 정보

- **관리형 서비스** (AWS 콘솔·API)
- **라이선스**: AWS 상용 서비스
- **문서**: Amazon Bedrock Agents

### 핵심 개념

- 자연어 지시문만으로 작업 오케스트레이션 자동화
- Lambda·S3·DynamoDB·Knowledge Base 통합
- IAM·VPC·HIPAA·SOC/ISO 보안 1st-class

### 강점

- **엔터프라이즈 보안 최강** (컴플라이언스 인증 완비)
- 오케스트레이션 코드 불필요
- AWS 네이티브 통합 완벽

### 약점

- AWS 생태계 종속
- 제어·실험 유연성 낮음 (편의성과의 트레이드오프)
- 콘솔 의존적 학습 곡선

### 적합 사례

- AWS 엔터프라이즈
- 컴플라이언스·보안 최우선
- 관리형 인프라 선호

---

## 10. AG2 (AutoGen 커뮤니티 포크)

### 기본 정보

- **라이선스**: Apache 2.0
- **리포지토리**: ag2ai/ag2
- **위상**: AutoGen 원본의 커뮤니티 주도 포크

### 2026 업데이트 — AG2 Beta (autogen.beta)

Ground-up 재설계:
- Streaming
- 이벤트 기반 아키텍처
- 멀티 공급자 LLM 지원
- Dependency injection
- Typed tools
- First-class testing

### 강점·약점

- **강점**: 대화 기반 패러다임, 연구·실험에 최적, Apache 2.0
- **약점**: 프로덕션 준비 부족, 관측성 플랫폼 없음, 코드 실행 수동 샌드박싱 필요

---

## 11. 선택 매트릭스 (2026-04)

| 우선순위 | 권장 프레임워크 |
|---|---|
| Claude 생태계 · CLAUDE.md 이식 | **Claude Agent SDK** |
| 복잡 프로덕션 워크플로우 · 상태 머신 | **LangGraph** |
| 빠른 프로토타이핑 · 비개발자 협업 | **CrewAI** |
| OpenAI 생태계 · 간단 핸드오프 | **OpenAI Agents SDK** |
| 타입 안전 · 금융/의료/법률 | **Pydantic AI** |
| .NET · Azure 엔터프라이즈 | **Microsoft Agent Framework** |
| 다중 언어 · Google Cloud | **Google ADK** |
| AWS 엔터프라이즈 · 컴플라이언스 | **Amazon Bedrock Agents** |
| 연구 · 대화 기반 실험 | **AG2** |

---

## 12. 표준화 (2026 수렴점)

### MCP (Model Context Protocol)

- Anthropic 2024-11 발표, 2026년 업계 표준화
- **지원**: Claude Agent SDK (1st-class), OpenAI Agents SDK (v0.13 resource 지원), LangGraph, Pydantic AI, Microsoft Agent Framework, Google ADK
- 효과: 도구 재사용. 한 번 작성한 MCP 서버를 여러 프레임워크에서 공유

### A2A (Agent-to-Agent)

- Google 주도 제안, 여러 프레임워크가 채택 중
- 서로 다른 프레임워크의 에이전트끼리 통신 가능

### OpenTelemetry

- AutoGen v0.4부터 공식 지원 → 다른 프레임워크로 확산
- Claude Agent SDK, LangGraph, Pydantic AI 공통 지원
- 프레임워크 독립적 관측성

---

## 13. 출처

### 공식 문서·리포

- Claude Agent SDK: https://code.claude.com/docs/en/agent-sdk/overview
- Claude Agent SDK Python: https://github.com/anthropics/claude-agent-sdk-python
- Claude Agent SDK TS: https://github.com/anthropics/claude-agent-sdk-typescript
- LangGraph: https://github.com/langchain-ai/langgraph · https://docs.langchain.com/oss/python/langgraph/overview
- LangChain 1.0: https://blog.langchain.com/langchain-langgraph-1dot0/
- LangChain Middleware: https://blog.langchain.com/agent-middleware/
- CrewAI: https://github.com/crewAIInc/crewAI · https://docs.crewai.com/
- Microsoft Agent Framework: https://learn.microsoft.com/en-us/agent-framework/overview/
- Microsoft Agent Framework 1.0 announcement: https://devblogs.microsoft.com/agent-framework/microsoft-agent-framework-version-1-0/
- OpenAI Agents SDK: https://openai.github.io/openai-agents-python/ · https://github.com/openai/openai-agents-python
- AutoGen: https://github.com/microsoft/autogen
- AutoGen → Agent Framework 마이그레이션: https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/

### 비교·분석 기사

- Definitive Guide 2026 Agentic Frameworks: https://softmaxdata.com/blog/definitive-guide-to-agentic-frameworks-in-2026-langgraph-crewai-ag2-openai-and-more/
- Best Multi-Agent Frameworks 2026: https://gurusup.com/blog/best-multi-agent-frameworks-2026
- LangChain 1.0 vs LangGraph 1.0: https://www.clickittech.com/ai/langchain-1-0-vs-langgraph-1-0/
- AutoGen v0.4 재설계: https://www.microsoft.com/en-us/research/articles/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/
- MS Agent Framework 1.0 shipping: https://visualstudiomagazine.com/articles/2026/04/06/microsoft-ships-production-ready-agent-framework-1-0-for-net-and-python.aspx
