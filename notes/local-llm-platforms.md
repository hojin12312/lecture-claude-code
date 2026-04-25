# 로컬 LLM 에코시스템 플랫폼 참조

웹 조사 결과 요약 (2026-04-25)
OpenClaw 상세 내용은 `notes/openclaw-reference.md` 참조.

---

## HuggingFace Hub (huggingface.co)

### 개요

AI 모델·데이터셋·데모 앱을 공유하는 세계 최대 규모의 AI 공개 저장소.
2026-04 현재 모델 200만+ · 데이터셋 50만+ · Spaces 100만+. "AI 버전 GitHub"로 통함.

### 주요 기능

| 기능 | 설명 |
|---|---|
| **모델 허브** | 오픈소스 LLM 파일 검색·다운로드 (GGUF, safetensors 등) |
| **Spaces** | 웹 기반 AI 데모 앱 (코딩 없이 브라우저에서 바로 사용) |
| **Inference API** | 클라우드에서 모델 API 호출 (무료 월 100K 크레딧) |
| **AutoTrain** | UI 기반 모델 파인튜닝 (코딩 불필요) |
| **Chat UI** | ChatGPT 유사 채팅 인터페이스 (무료) |

### 무료/유료

- **무료**: 모델·데이터셋 다운로드, Spaces 무료 호스팅, Inference API 월 100K 크레딧
- **유료**: 전용 GPU 컴퓨팅($0.60/hr~), Inference Endpoints, Enterprise Hub

### 출처

- huggingface.co, huggingface.co/pricing

---

## Ollama (ollama.com)

### 개요

로컬 컴퓨터에서 LLM을 실행하기 위한 CLI 기반 오픈소스 도구.
`ollama run qwen3:14b` 한 줄로 모델 다운로드+실행. macOS·Linux·Windows 지원.

### 주요 기능

- **CLI**: `ollama run <모델명>` — 모델 자동 다운로드 후 즉시 대화
- **모델 라이브러리**: ollama.com/library — Llama / Qwen / Gemma / DeepSeek 등
- **OpenAI 호환 REST API**: `localhost:11434/v1/` 엔드포인트, 기존 OpenAI SDK로 바로 연결
- **멀티모달**: LLaVA 계열 이미지 입력 지원

### 최신 버전 (2026-04-25 기준, 1차 출처 검증 후)

| 버전 | 주요 변경 | 날짜 |
|---|---|---|
| **v0.21.2** | OpenClaw 온보딩 안정성, 모델 추천 순서 정렬, 웹 검색 플러그인 번들링 | **2026-04-23** |
| **v0.21.1** | Kimi CLI 통합, MLX logprobs/top-P/top-K, GLM4 MoE Lite | **2026-04-22** |
| **v0.21.0** | Gemma 4 MLX, Hermes Agent, GitHub Copilot CLI 통합, OpenCode 인라인 설정, OpenClaw `--yes` 버그픽스 | **2026-04-16** |
| **v0.20.5** | OpenClaw 채널 설정 (WhatsApp/Telegram/Discord) — **Ollama-OpenClaw 통합 시작점**, Gemma 4 Flash Attention | **2026-04-09** |
| **v0.7.0** | 멀티모달 신엔진 (Llama 4 / Gemma 3 / Qwen 2.5 VL / Mistral Small 3.1) | **2025-05-13** |
| **v0.6.2** | Gemma 3 멀티 이미지, AMD Strix Halo GPU 지원, /save 버그픽스 | **2025-03-18** |

> **이전 잘못된 기록 정정**: "v0.21.0 (2026-04-17): Kimi K2.5/GLM-5/MiniMax 지원, OpenClaw 통합" / "v0.6.2 (2026-03): Llama 4, Flash Attention 2.7, M4 Metal 3" — 1차 출처 대조 시 모두 시점·내용 오류였음. v0.6.2는 실제 1년 전(2025-03), Llama 4 지원은 v0.7.0부터.

### Ollama + MLX 통합 (2026-03-31)

Apple Silicon Mac에서 Ollama가 Apple의 **MLX 프레임워크**를 백엔드로 활용할 수 있게 됨. 기존 llama.cpp Metal 백엔드 대비:

- 프롬프트 처리 속도 **약 1.6×**
- 응답 생성 속도 **거의 2×**
- 모델 스케줄러 개선으로 다중 GPU·VRAM OOM 크래시 완화

MLX 백엔드는 Apple Silicon(M1~M5)에서만 활성화. macOS 26.2+ 권장 (M5 Neural Accelerators 사용 시 필수).

### 출처

- ollama.com, github.com/ollama/ollama/releases

---

## LM Studio (lmstudio.ai)

### 개요

GUI 데스크탑 앱으로 LLM을 로컬 실행. HuggingFace에서 모델을 검색·다운로드해
ChatGPT처럼 대화하거나, 로컬 OpenAI 호환 서버로 노출 가능.
macOS·Windows·Linux 지원. 완전 무료.

### 주요 기능

- **모델 검색 UI**: HuggingFace 모델을 앱 내에서 직접 검색·다운로드
- **채팅 UI**: 브라우저 없이 데스크탑 앱에서 대화, PDF 업로드 분석
- **로컬 API 서버**: OpenAI 호환 REST API (`localhost:1234/v1/`), Python/JS SDK 제공
- **LM Link** (v0.4.5+): 원격 LM Studio 머신에 Tailscale 암호화로 연결 — 팀 공유 가능
- **MCP Host** (v0.4.10+): MCP 서버 연결, 에이전트 도구 확장
- **다중 GPU 병렬 처리** (v0.4.10+): VRAM 분산 제어

### 최신 버전 (2026-04-25 기준, lmstudio.ai/changelog 직접 검증)

| 버전 | 주요 변경 | 날짜 |
|---|---|---|
| v0.4.12 | Qwen 3.6 지원, PDF export 개선, Windows MCP server OAuth 수정, Qwen 3.5 성능 개선 | 2026-04-17 |
| **v0.4.11** | **Gemma 4 채팅 템플릿 업데이트** (단일 변경) | 2026-04-10 |
| **v0.4.10** | **Gemma 4 tool call 안정화 + MCP 서버 OAuth 지원** | 2026-04-09 |
| v0.4.9 | Anthropic-compatible v1/messages output_config.effort 지원 | 2026-04-02 |
| **v0.4.7** | **math parsing 수정, Qwen 3.5/GLM tool calling 개선, 반응형 UI 수정** | 2026-03-18 |
| v0.4.5 | LM Link (원격 연결, Tailscale 암호화) | 2026-02 |

> **이전 잘못된 기록 정정 (lmstudio.ai/changelog 1차 출처 대조)**:
> - v0.4.11 "Continuous Batching, Stateful REST API" → changelog 미등장. 실제는 Gemma 4 채팅 템플릿만
> - v0.4.10 "다중 GPU 제어" → changelog 미등장. 실제는 MCP OAuth 지원
> - v0.4.7 "Linux ARM 지원" → changelog 미등장. 실제는 math parsing/tool calling 수정

추가 (2026-03): NVIDIA DGX Station GB300 Blackwell 지원, LM Studio 모델을 **Claude Code**에서 Anthropic-compatible API로 호출 가능.

### 출처

- lmstudio.ai, lmstudio.ai/changelog

---

## 플랫폼 비교 요약

| 항목 | HuggingFace | Ollama | LM Studio |
|---|---|---|---|
| 주요 역할 | 모델 저장소 + 클라우드 | 로컬 CLI 실행 | 로컬 GUI 실행 |
| 비개발자 친화도 | ⭐⭐⭐⭐⭐ (Spaces) | ⭐⭐⭐ (CLI) | ⭐⭐⭐⭐⭐ (GUI) |
| 인터넷 필요 | ✓ (클라우드) | 다운로드 시만 | 다운로드 시만 |
| OpenAI 호환 API | 별도 설정 필요 | ✓ 기본 제공 | ✓ 기본 제공 |
| 무료 여부 | 무료 + 유료 옵션 | 완전 무료 | 완전 무료 |
| 프라이버시 | 클라우드 | 완전 로컬 | 완전 로컬 |
