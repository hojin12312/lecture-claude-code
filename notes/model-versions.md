# Reasoning 모델 버전 참조 (로컬 전용, gitignore)

강의 자료 내 모델명 최신화를 위한 참조 문서입니다.
업데이트 시 이 파일 먼저 갱신 → reasoning-models.md / memory-bandwidth.md 반영.

마지막 조사: 2026-04-25

---

## Reasoning 모델 현황

### OpenAI

| 모델명 | API ID | 컨텍스트 | Input 가격 | Output 가격 | 비고 |
|---|---|---|---|---|---|
| GPT-5.4 | `gpt-5.4` | 1M | $2.50/MTok | $15/MTok | Reasoning none~xhigh 조절 가능, 플래그십 |
| GPT-5.4 mini | `gpt-5.4-mini` | 400K | $0.75/MTok | $4.50/MTok | 코딩·서브에이전트 특화 |
| GPT-5.4 nano | `gpt-5.4-nano` | 400K | $0.20/MTok | $1.25/MTok | 고볼륨 단순 작업, MCP 지원 |

- Knowledge cutoff: 2025-08-31
- Reasoning 레벨: none / low / medium / high / xhigh (Claude budget_tokens와 같은 개념)
- o3, o4-mini는 구세대

### DeepSeek

| 모델명 | API ID | 출시일 | 상태 | 비고 |
|---|---|---|---|---|
| **DeepSeek-V4-Pro** | `deepseek-chat` 계열 | **2026-04-24** | GA | **1.6T MoE (49B active) / 1M context / Hybrid CSA+HCA + mHC + Muon optimizer / Native FP4+FP8 / MIT** |
| **DeepSeek-V4-Flash** | 동일 계열 | 2026-04-24 | GA | **284B (13B active) / 1M / 동일 아키텍처 / MIT** |
| DeepSeek-V3.1 | (구 세대) | — | 사용 가능하나 V4로 강등 | MLA 압축 |
| DeepSeek-R1-0528 | `deepseek-reasoner` | 2025-05-28 | GA | reasoning 계열 최신 |
| DeepSeek-R1 | `deepseek-reasoner` | 2025-01 | GA | |

#### DeepSeek V4 핵심

- **효율**: 1M-token 컨텍스트에서 single-token FLOPs **27%**, KV 캐시 **10%** (V3.2 대비)
- **새 어텐션**: Compressed Sparse Attention (CSA) + Heavily Compressed Attention (HCA)
- **mHC (manifold-constrained hyper-connections)**: 기존 residual 강화, 신호 전달 안정성
- **Muon optimizer**: 학습 안정성·수렴 가속
- **사전학습**: 32T+ 토큰. 사후학습은 Domain-specific SFT/RL (GRPO) → On-policy distillation
- **포맷**: MoE expert FP4 + 기타 FP8 — Native 혼합 정밀도 지원

#### DeepSeek V4 API 가격 (per 1M tokens)

| 모델 | Input cache hit | Input cache miss | Output |
|---|---|---|---|
| **V4-Pro** | $0.145 | $1.74 | $3.48 |
| **V4-Flash** | $0.028 | $0.14 | $0.28 |

- 둘 다 thinking mode 지원, 1M context, 384K max output
- 출처: huggingface.co/unsloth/DeepSeek-V4-Pro, api-docs.deepseek.com/quick_start/pricing

### Google Gemini

| 모델명 | API ID | 출시일 | 상태 | 비고 |
|---|---|---|---|---|
| Gemini 2.5 Pro | `gemini-2.5-pro` | 2025-03 (GA 2025-05) | **Stable** | thinking 내장, 이전 안정판 |
| Gemini 2.5 Flash | `gemini-2.5-flash` | 2025-05 | Stable | thinking_budget 0~24576 조절 가능 |
| Gemini 2.5 Flash-Lite | `gemini-2.5-flash-lite` | 2025 | Stable | 가장 빠르고 저렴 |
| Gemini 3 Flash | `gemini-3-flash-preview` | 2025-12-17 | Preview | |
| **Gemini 3.1 Pro** | `gemini-3.1-pro` | 2026-02-19 (GA 2026-04-01) | **GA (paid-only)** | 현재 프로덕션 권장, Deep Think, ARC-AGI-2 45.1% |
| Gemini 3.1 Flash-Lite | `gemini-3.1-flash-lite-preview` | 2026-03-03 | Preview | |
| Gemini 3.1 Flash Live | `gemini-3.1-flash-live-preview` | 2026-03-26 | Preview | 실시간 음성 |
| Gemini 3.1 Flash TTS | `gemini-3.1-flash-tts-preview` | 2026-04-15 | Preview | 음성 생성 |

- Gemini 3 Pro: 2026-03-09 deprecated/종료
- Gemini 3.1 Pro GA 전환(2026-04-01): Preview에서 paid-only GA로 승격, Vertex AI·Gemini API·Gemini 앱·NotebookLM 동시 배포

### Anthropic Claude

| 모델명 | API ID | 상태 | 컨텍스트 | Input | Output | 비고 |
|---|---|---|---|---|---|---|
| Claude Opus 4.7 | `claude-opus-4-7` | GA (2026-04-16) | 1M | $5/MTok | $25/MTok | Hybrid reasoning, 코딩·에이전트 frontier |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | GA (2026-02-17) | 1M (beta) | $3/MTok | $15/MTok | Extended Thinking 지원, 속도·성능 균형 |
| Claude Haiku 4.5 | `claude-haiku-4-5-20251001` | GA | 200K | $1/MTok | $5/MTok | Extended Thinking 미지원 |
| Claude Mythos Preview | 비공개 | **Limited Preview** (2026-04-07) | — | — | — | 11개 기업 한정, 보안 취약점 탐지 용도. GA 계획 없음 |

- Opus 4.7 가격은 4.6에서 불변 ($5 / $25)
- **새 토크나이저**: 동일 텍스트를 1x~1.35x 토큰 수로 처리 (최대 ~35% 증가). 실사용 비용도 최대 35% 상승 가능.
- Claude Mythos Preview: "Opus 4.7보다 능력 ahead but safety-untested" 포지션. 공개 배포 없음 — 언급할 때는 "limited research preview"로 명시.

---

## Reasoning 모델 thinking 조절 파라미터 (2026-04)

| 공급자 | 파라미터 | 지원 레벨 | 과금 |
|---|---|---|---|
| OpenAI GPT-5.4 | `reasoning.effort` | `none / low / medium / high / xhigh` | thinking 토큰 = output 가격 ($15/MTok) |
| Anthropic Claude Opus 4.7 | `thinking.effort` + `task_budget` | `low / medium / high / max` (adaptive, opt-in) | thinking 토큰 = output 가격 ($25/MTok) |
| Anthropic Claude Opus 4.6 이하 | `thinking.budget_tokens` (정수) | 최소 1024 | output 가격 ($25/MTok) |
| Google Gemini 3.1 Pro | `thinking_level` (권장) 또는 `thinking_budget` (backward compat) | `LOW / MEDIUM / HIGH` (HIGH = Deep Think Mini) | thinking 토큰 = output 가격 ($12/MTok ≤200K, $18 초과) |

### 평균 thinking 토큰 체감 (2026-04 조사)

| 설정 | 평균 thinking 토큰/req | req당 thinking 비용 |
|---|---|---|
| Gemini 3.1 Pro LOW | ~300 | ~$0.0036 |
| Gemini 3.1 Pro MEDIUM | ~2,000 | ~$0.024 |
| Gemini 3.1 Pro HIGH (Deep Think Mini) | ~8,000 | ~$0.096 |
| GPT-5.4 xhigh (긴 프롬프트) | ~20,000 | ~$0.30 |

- 일반 요청(thinking 없음) 대비 req당 **3-10배 비용**
- 토큰 수 × output 단가로 계산 (input이 아님)
- 에이전트 루프에서 thinking이 도구 호출 사이마다 끼면 비용 누적 가속

### 컨텍스트 구간별 단가 변화

| 공급자 | 기준 | 초과 시 |
|---|---|---|
| OpenAI GPT-5.4 | 272K 초과 | input 2배 ($2.50 → $5/MTok) |
| Google Gemini 3.1 Pro | 200K 초과 | input $2 → $4, output $12 → $18 |
| Anthropic Claude Opus 4.7 | 구간 없음 | 고정 ($5 / $25) |

### 주요 변화 (2026-04 기준)

- **Claude 4.7에서 `budget_tokens` 제거됨** — adaptive thinking + effort level로 전환. 기본은 비활성(opt-in).
- **Task Budget (Claude 4.7 신규)**: thinking + tool call + tool result + final output 전체 agentic loop의 토큰 한도를 지정. 루프 내부 개별 thinking 토큰 제어 대신 전체 비용 상한 설정 방식.
- **Gemini thinking_level로 이전**: `thinking_budget`는 backward compat 유지되지만 `thinking_level`이 예측가능성 기준 권장.
- **OpenAI GPT-5.5 출시** (2026-04-23): 가격이 GPT-5.4의 2배 ($5 / $30). 단, 공식 발표는 "more intelligent + more token efficient" — 동일 태스크에서 토큰 수 감소 가능. 체감 비용은 태스크별로 2~5배 수준.

---

## 로컬 오픈소스 모델 현황 (2026-04-25 조사)

### Qwen 시리즈 (Alibaba)

| 모델 | 파라미터 | 컨텍스트 | 비고 |
|---|---|---|---|
| Qwen3 4B/8B/14B/32B | dense | 128K | 기존 |
| Qwen3.5 9B | 9B dense | 262K | 2026-02 출시 |
| Qwen3.5 27B | 27B dense | 262K | 2026-02 출시 |
| Qwen3.5 35B-A3B | 35B MoE (3B active) | 262K | 2026-02 출시 |
| Qwen3.5 122B-A10B | 122B MoE (10B active) | 262K | |
| Qwen3.5 397B-A17B | 397B MoE (17B active) | 262K | |
| Qwen3.6 27B | 27B dense | 262K (최대 1M) | 2026-04-23 출시, DeltaNet 하이브리드 아키텍처 (16/64 레이어만 표준 어텐션), Apache 2.0 |
| Qwen3.6 35B-A3B | 35B MoE (3B active) | 1M | 2026-04-16 출시, 기본 thinking 모드 |

- Qwen3.5/3.6 공통: 201개 언어, Apache 2.0
- Qwen3.6 27B: DeltaNet(선형 어텐션) 레이어 다수, 표준 어텐션 레이어 16개만 → KV 캐시 대폭 감소
- Qwen3.6 35B-A3B: reasoning 기본 활성화, 1M 컨텍스트

### GLM 시리즈 (Z.AI / ZhipuAI)

| 모델 | 파라미터 | 컨텍스트 | 비고 |
|---|---|---|---|
| GLM-4.6 | 357B MoE | **200K** (4.5의 128K에서 확장) | huggingface.co/zai-org/GLM-4.6 직접 |
| GLM-4.7-Flash | 30B MoE (3B active) | 128K | GGUF Q4 ~18.5 GB, Q8 ~32 GB, BF16 ~60 GB (전체 30B 가중치 포함) |
| GLM-5.1 | **754B MoE** (40B active) | 203K | MLA·DSA 압축 (GLM_MOE_DSA), 2026-04-07 출시, MIT |
| GLM-5.1-Instruct | 동일 아키텍처 | 203K | enable_thinking=false 인스트럭트 모드 |

- GLM-5.1: 256 routed experts + 1 shared, 8+1 active, 78 layers
- GLM-4.7-Flash: "3B active"는 추론 시 활성화 파라미터이지만 GGUF는 전체 30B 가중치를 저장함 → dense 30B 수준의 파일 크기 (이전 기록 ~10/~18 GB는 오류)
- 출처: huggingface.co/zai-org/GLM-5.1, huggingface.co/zai-org/GLM-4.6, unsloth/GLM-5.1-GGUF
- **이전 오류 정정**: GLM-5.1 파라미터 744B → 754B

### Gemma4 시리즈 (Google)

| 모델 | 파라미터 | 컨텍스트 | 비고 |
|---|---|---|---|
| Gemma4 E2B | 2B dense | — | 엣지/모바일용 |
| Gemma4 E4B | 4B dense | — | 엣지/모바일용 |
| Gemma4 26B-A4B | 25.2B MoE (3.8B active) | 256K | 2026-04-03 출시, Apache 2.0, 멀티모달; Q4 17.0 GB, Q8 26.9 GB, BF16 50.5 GB |
| Gemma4 31B | 30.7B dense | **256K** | 60 레이어, sliding window 1024, **마지막 레이어 global attention**; Q4 19.6 GB, Q8 32.6 GB, BF16 61.4 GB |

- 26B-A4B: 128 experts (8 active + 1 shared), sliding window + global attention 교차
- 31B: hybrid attention pattern (local SWA + global), p-RoPE 메모리 최적화
- Shared KV Cache: 마지막 N 레이어가 앞 레이어의 KV 상태 재사용 → KV 효율 향상
- 출처: huggingface.co/google/gemma-4-31b 직접 검증
- **이전 오류 정정**: Gemma4 31B 컨텍스트 128K → 256K

### Mistral 시리즈 (Mistral AI)

| 모델 | 파라미터 | 컨텍스트 | 비고 |
|---|---|---|---|
| Mistral Small 4 (2603) | 119B MoE (**6B active per token, 8B with embedding/output**) | **256K** | 2026-03-16 NVIDIA GTC 발표, Apache 2.0, 멀티모달, Q4 73.8 GB, Q8 126 GB, BF16 238 GB |
| Mistral Medium 3 | 미공개 | 미공개 | 2026-04-09 출시, API-only (weights 비공개), EU AI Act 컴플라이언스 메타데이터 내장, 유럽 언어 특화 |

- 128 experts, 4 active per token
- instruct·reasoning·vision·coding 4종 기능 통합 단일 체크포인트
- NVFP4 공식 버전 존재 (mistralai/Mistral-Small-4-119B-2603-NVFP4)
- Mistral Medium 3: "open weights" 주장하는 일부 기사가 있으나 공식 사이트는 API-only — 로컬 실행 불가로 간주
- 출처: mistral.ai/news/mistral-small-4 (1차 출처 직접 검증)
- **이전 오류 정정**: 컨텍스트 262K → 256K, active 6.5B → 6B (8B with embedding)

### Llama 시리즈 (Meta)

| 모델 | 파라미터 | 컨텍스트 | 비고 |
|---|---|---|---|
| Llama 3.2 3B | 3.2B dense | 128K | |
| Llama 3.3 70B | 70.6B dense | 128K | Q4 42.5 GB, Q8 75 GB, BF16 141 GB |
| Llama 4 Scout | 109B MoE (17B active) | 10M | 2026-04-05 출시, 16 experts; Q4 65.4 GB, Q8 115 GB, BF16 216 GB |
| Llama 4 Maverick | 402B MoE (17B active) | 1M | 2026-04-05 출시, 128 experts; Q4 243 GB, Q8 426 GB, BF16 801 GB |
| Llama 3.1 405B | 405B dense | 128K | Q4 245 GB, Q8 430 GB, BF16 810 GB |

### Qwen3-Coder (Alibaba)

| 모델 | 파라미터 | 컨텍스트 | 비고 |
|---|---|---|---|
| Qwen3-Coder-Next | ~80B MoE (3B active) | 256K | 코딩 에이전트 특화; Q4 48.7 GB, Q8 84.8 GB |

### Nemotron 시리즈 (NVIDIA)

| 모델 | 파라미터 | 아키텍처 | 비고 |
|---|---|---|---|
| Nemotron Cascade 2 | 30B (3B active) | Mamba-Transformer MoE 하이브리드 (Nano 베이스 포스트트레이닝) | Q4 24.7 GB, Q8 33.6 GB, BF16 63.2 GB, **262K 컨텍스트**, **2026-03-16 출시 (arXiv 2603.19220)**, IMO/IOI 2025 gold-medal, ICPC 10/12 gold |
| Nemotron 3 Nano | 31.6B (3.2B active) | Mamba-Transformer MoE 하이브리드 | Q4 24.7 GB, Q8 33.5 GB, **1M 컨텍스트** |
| Nemotron 3 Super | 120B (12B active) | MoE | Q4 87.0 GB, Q8 128.5 GB, SWE-bench 60.47%, 1M 컨텍스트 |

### MiniMax / Kimi 등

| 모델 | 파라미터 | 컨텍스트 | API 가격 (input/output per MTok) |
|---|---|---|---|
| MiniMax M2 | 230B MoE (10B active) | **128K** | huggingface.co/MiniMaxAI/MiniMax-M2 직접 검증 |
| MiniMax M2.5 | MoE | 198K | $0.30 / $1.20 (SWE-bench 80.2%) |
| MiniMax M2.7 | 229B MoE (~10B active) | 205K | $0.30 / $1.20; Q4 139 GB, Q8 243 GB, BF16 457 GB; 로컬 가능 |
| **Kimi K2 (이전)** | 1T MoE (32B active, 384 experts, MLA, 61 layers) | **128K** | huggingface.co/moonshotai/Kimi-K2-Instruct |
| Kimi K2.5 | 동일 계열 | 262K | 클라우드 + GGUF 변환본 |
| **Kimi K2.6** | **1T MoE (32B active, 384 experts: 8 routed + 1 shared, MLA)** | **256K** | **Modified MIT License** — HF 가중치 공개 + Ollama `:cloud` 태그. 멀티모달. 300 sub-agents / 4,000 coordinated steps. SWE-Bench Pro 58.6 / Verified 80.2 / HLE-Full 54.0 |
| Kimi K2.6 GGUF (unsloth) | — | — | BF16 2.05 TB / UD-Q8_K_XL 595 GB / UD-Q4_K_XL 584 GB / UD-Q2_K_XL 340 GB. Q8과 Q4 차이 10 GB뿐 (lossless 권장) |

- 출처: huggingface.co/moonshotai/Kimi-K2-Instruct, huggingface.co/MiniMaxAI/MiniMax-M2, huggingface.co/unsloth/Kimi-K2.6-GGUF
- **이전 오류 정정**:
  - Kimi K2 컨텍스트 262K → **128K** (262K는 K2.5 값)
  - Kimi K2.6 라이선스 "미공개·클라우드 전용" → **Modified MIT, HF 가중치 공개**
  - MiniMax M2 (128K), MiniMax M2.7 (205K) 분리 명시

---

## 강의 자료 내 언급 현황

### appendix/reasoning-models.md
- 도입부 모델 목록: `GPT-5.4, DeepSeek R1-0528, Gemini 2.5 Pro / 3.1 Pro(Preview), Claude Extended Thinking`
- 코드 예시 모델: `claude-opus-4-7` ✅

### appendix/memory-bandwidth.md
- 모델 표: Qwen3, Gemma4, DeepSeek-V3.1, R1-0528, Llama4 Scout 등
  (별도 갱신 필요 시 해당 파일 직접 확인)

---

## 주요 벤치마크 점수 (2026-04 조사)

경량/중형 모델이 과거 대형 모델의 성능을 따라잡는 근거로 사용. 출처는 각 모델의 HuggingFace 모델 카드, BenchLM.ai, 공식 기술 보고서.

### 2026년 경량·중형 모델

| 모델 | 크기 (active) | HumanEval | LiveCodeBench v6 | SWE-bench Verified | GPQA Diamond | AIME25/26 | MMLU-Pro |
|---|---|---|---|---|---|---|---|
| Qwen3.6 27B | 27B dense | — | 83.9 | 77.2 | 87.8 | 94.1 (AIME26) | — |
| Qwen3.6 35B-A3B | 35B (3B) | — | — | — | — | — | — |
| Gemma4 26B-A4B | 25B (3.8B) | 78.5 (외부 인용, 1차 출처 보류) | 77.1 | — | 82.3 | 88.3 (AIME26) | 82.6 |
| Nemotron Cascade 2 | 30B (3B) | — | 87.2 | **Pass@1 49.9 / Pass@4 65.2** (OpenHands fine-tune 단계) — 최종 RL 후 50.2 | — | 98.6 (AIME25, tool-RL) | — |
| Mistral Small 4 | 119B (6.5B) | 92.0 (외부 인용, Mistral Large 3 점수와 혼동 의심) | — | — | — | — | — |
| **Qwen3.5 397B-A17B** | 397B (17B) | — | **83.6** | **76.2** (Pro 50.9) | **88.4** | **91.3 (AIME26)** | **87.8** |

추가 수치 (1차 출처 직접 검증):
- **Nemotron Cascade 2**: 2026-03-16 출시, arXiv 2603.19220
  - IMO 2025: gold-medal (정확 점수는 NVIDIA 페이지에 미공개)
  - IOI 2025: **439.28/600** (gold)
  - ICPC World Finals: **10/12 = Gold Medal** (이전 "동메달 수준" 기록은 외부 인용 오류)
  - SWE-bench Verified (OpenHands): fine-tune 단계 **Pass@1 49.9 / Pass@4 65.2**, 전체 RL 파이프라인 후 50.2
- **Qwen3.6 27B**: 코딩 카테고리 전체 111개 모델 중 #9위, 평균 86.9점

> **Qwen3.5 397B-A17B 벤치마크 표 채움 근거**: 이전 README에서 "27B가 397B를 이긴다" 헤드라인을 쓰면서도 397B 행을 공란(`—`)으로 두어 직접 비교가 표 안에 부재했던 모순을 해소함.

### 기준선 모델 (비교용)

| 모델 | 크기 | 출시 | HumanEval | MMLU | 비고 |
|---|---|---|---|---|---|
| GPT-4 (original) | ~1.8T est. | 2023-03 | ~85 | 86 | 당시 flagship |
| Llama 3.1 405B | 405B dense | 2024-07 | 89.0 | 87.3 (MMLU 5-shot) | 오픈소스 최대 dense |
| Qwen3.5 397B-A17B (Reasoning) | 397B (17B) | 2026-02 | 84.9 | — | 경량 대비 큰 격차 없음 |

### 핵심 관찰

1. **Qwen3.6 27B vs Qwen3.5 397B**: 크기 1/15, LiveCodeBench v6는 오히려 27B가 비슷 (83.9). SWE-bench Pro에서 27B가 더 높게 나오는 agentic coding 케이스 존재 → "Agentic Coding Benchmarks에서 397B MoE 초과" 헤드라인 근거.
2. **Nemotron Cascade 2 (3B active)**: 2024 Llama 3.1 405B의 HumanEval 89.0%를 LiveCodeBench v6 87.2% + 수학 올림피아드 금메달 성능으로 효과적으로 넘어섬. 활성 파라미터 1/135.
3. **Gemma4 26B-A4B (3.8B active)**: Llama 4 Scout (17B active) HumanEval 72.3 대비 78.5 — active 파라미터 1/4.5 수준으로 더 높은 성능.
4. **Mistral Small 4 (6.5B active)**: HumanEval 92% — Llama 3.1 405B의 89.0%를 넘어섬. 활성 파라미터 1/62.

### 주요 출처
- Nemotron Cascade 2 기술 보고서: https://research.nvidia.com/labs/nemotron/nemotron-cascade-2/
- Qwen3.6-27B 공식 블로그: https://qwen.ai/blog?id=qwen3.6-27b
- BenchLM.ai 리더보드 (벤치마크 검증): https://benchlm.ai/
- Gemma 4 벤치마크: https://benchlm.ai/models/gemma-4-26b-a4b
- Mistral Small 4: https://mistral.ai/news/mistral-small-4

---

## 주요 참조 링크
- OpenAI 모델: https://platform.openai.com/docs/models
- DeepSeek API: https://api-docs.deepseek.com
- Google Gemini: https://ai.google.dev/gemini-api/docs/models
- Anthropic 모델: https://platform.claude.com/docs/en/about-claude/models
