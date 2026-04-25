# Notes 검증 로그 (2026-04-25)

> `VERIFICATION-FEEDBACK-2026-04-25.md` 의 정정 사항을 1차 출처 직접 fetch로 재검증한 기록입니다.
> 분류 체계: `확정` / `수정` / `보강` / `삭제` / `보류`

---

## 1. Ollama 릴리즈 (github.com/ollama/ollama/releases)

| 버전 | 검증 결과 | 분류 |
|---|---|---|
| **v0.6.2** | **2025-03-18**, Gemma 3 멀티 이미지, AMD Strix Halo GPU 지원, /save 버그픽스. **Llama 4 / Flash Attention 2.7 / M4 Metal 3 모두 changelog 미등장** | 수정 |
| **v0.7.0** | **2025-05-13**, 멀티모달 신엔진 시작 (Llama 4, Gemma 3, Qwen 2.5 VL, Mistral Small 3.1) | 보강 |
| **v0.20.5** | **2026-04-09**, OpenClaw 채널 설정 (WhatsApp/Telegram/Discord), Gemma 4 Flash Attention | 보강 |
| **v0.21.0** | **2026-04-16**, Gemma 4 MLX, Hermes Agent, GitHub Copilot CLI 통합, OpenCode 인라인 설정, openclaw --yes 버그픽스 | 수정 |
| **v0.21.1** | **2026-04-22**, Kimi CLI 통합, MLX logprobs/top-P/top-K, GLM4 MoE Lite 성능, Gemma 4 structured output 수정 | 보강 |
| **v0.21.2** | **2026-04-23**, OpenClaw 온보딩 안정성, 모델 추천 순서 정렬, 웹 검색 플러그인 번들링 | 수정 |

**잘못된 기존 기록**:
- "v0.21.0 (2026-04-17): Kimi K2.5 / GLM-5 / MiniMax 지원, OpenClaw 통합" → **전체 사실관계 오류**
- "v0.6.2 (2026-03): Llama 4, Flash Attention v2.7, M4 Metal 3 최적화" → **1년 차이의 시점·내용 모두 오류** (v0.6.2는 실제 2025-03-18, Llama 4는 v0.7.0부터)
- "OpenClaw Ollama 통합 시작: v0.21.0" → **v0.20.5부터 채널 설정 지원**

---

## 2. LM Studio 릴리즈 (lmstudio.ai/changelog)

| 버전 | 검증 결과 | 분류 |
|---|---|---|
| **v0.4.7** (2026-03-18) | math parsing 수정, Qwen 3.5/GLM tool calling 개선, 반응형 UI 수정. **Linux ARM 지원 미등장** | 수정 |
| **v0.4.10** (2026-04-09) | Gemma 4 tool call 안정화, **MCP 서버 OAuth 지원**. **"다중 GPU 제어" 미등장** | 수정 |
| **v0.4.11** (2026-04-10) | **Gemma 4 채팅 템플릿 업데이트만**. **"Continuous Batching" 미등장** | 수정 |
| **v0.4.12** (2026-04-17) | Qwen 3.6 지원, PDF export 개선, Windows MCP server OAuth 수정 | 확정 |

---

## 3. NVIDIA 데이터센터 GPU

| 항목 | 검증 결과 | 분류 | 출처 |
|---|---|---|---|
| **H200 SXM/NVL** | 둘 다 141 GB HBM3e, 4.8 TB/s 동일 | 확정 | nvidia.com/en-us/data-center/h200 |
| **B200** | **192 GB HBM3e** (2 × 4 HBM3e stacks × 24 GB/stack), 8 TB/s | 수정 (180→192) | snel.com, technical.city, slyd.com |
| **B300 (Blackwell Ultra)** | **288 GB HBM3e+, 8 TB/s, "Shipping Now"**, DGX B300 8-GPU 시스템 = 2.1 TB | 보강 | nvidia.com/en-us/data-center/dgx-b300 |

---

## 4. Apple Silicon (Mac Studio M3 Ultra)

| 항목 | 검증 결과 | 분류 | 출처 |
|---|---|---|---|
| **Mac Studio M3 Ultra 512GB 옵션** | **2026-03 단종**, 현재 최대 256 GB | 수정 | tomshardware.com/tech-industry/apple-pulls-512-mac-studio-upgrade-option |
| **256GB 업그레이드 가격** | **$1,600 → $2,000 (+$400 인상)** | 보강 | 동일 |
| **AI RAM 스퀴즈** | 글로벌 메모리 칩 부족, AI 수요 압박 | 보강 | 동일 |

---

## 5. 모델 컨텍스트·파라미터 (HuggingFace 모델 카드 직접)

| 모델 | 검증 결과 | 분류 | 출처 |
|---|---|---|---|
| **GLM-4.6** | **컨텍스트 200K** (4.5의 128K → 200K 확장), 357B MoE | 확정 | huggingface.co/zai-org/GLM-4.6 |
| **GLM-5.1** | **754B MoE** (피드백 744B 잘못, 754B 정확), GLM_MOE_DSA 아키텍처. Q4_K_XL ~466 GB, BF16 ~1.51 TB | 수정 (744→754) | huggingface.co/zai-org/GLM-5.1, unsloth/GLM-5.1-GGUF |
| **MiniMax M2** | **230B MoE / 10B active**, **컨텍스트 128K** (SWE-bench 평가 기준) | 보강 | huggingface.co/MiniMaxAI/MiniMax-M2 |
| **Kimi K2** | **128K 컨텍스트** (262K가 아님), 1T MoE / 32B active / 384 experts (8+1 active) / MLA / 61 layers | 수정 (262K→128K) | huggingface.co/moonshotai/Kimi-K2-Instruct |
| **Gemma 4 31B** | **256K 컨텍스트** (128K가 아님), 30.7B dense, 60 레이어, sliding window 1024, 마지막 레이어 global attention | 수정 (128K→256K) | huggingface.co/google/gemma-4-31b |
| **Mistral Small 4** | **256K 컨텍스트** (262K가 아님), **6B active per token (8B with embedding/output)**, 119B 총 | 수정 | mistral.ai/news/mistral-small-4 |
| **DeepSeek V4-Pro** | **1.6T MoE / 49B active / 1M 컨텍스트 / FP4+FP8 / MIT** / Hybrid CSA+HCA + mHC + Muon | 신규 보강 | huggingface.co/unsloth/DeepSeek-V4-Pro |
| **DeepSeek V4-Pro 효율** | **1M 컨텍스트에서 V3.2 대비 single-token FLOPs 27%, KV 캐시 10%** | 신규 보강 | 동일 |
| **DeepSeek V4 가격** | **V4-Pro $0.145/$1.74/$3.48 per 1M (input cache hit / miss / output)**. **V4-Flash $0.028/$0.14/$0.28** | 신규 보강 | api-docs.deepseek.com/quick_start/pricing |

---

## 6. Qwen3.5 397B-A17B 벤치마크

| 항목 | 검증 결과 | 분류 |
|---|---|---|
| LiveCodeBench v6 | **83.6** | 보강 |
| SWE-bench (raw) | 79.3 | 보강 |
| GPQA Diamond | **88.4** | 보강 |
| MMLU-Pro | **87.8** | 보강 |
| AIME 2025 | **91.3** | 보강 |

---

## 7. Nemotron Cascade 2

| 항목 | 검증 결과 | 분류 |
|---|---|---|
| **출시일** | **2026-03-16**, arXiv 2603.19220 | 보강 |
| **IMO 2025 / IOI 2025** | 둘 다 **gold-medal** | 확정 |
| **SWE-bench Verified (OpenHands)** | Cascade RL 최종 **50.2**, fine-tuning 단계 **Pass@1 49.9 / Pass@4 65.2** | 보강 |

---

## 8. OpenClaw

| 항목 | 검증 결과 | 분류 | 출처 |
|---|---|---|---|
| **GitHub URL** | **github.com/openclaw/openclaw** (개인 GitHub은 github.com/steipete) | 수정 | github.com/openclaw/openclaw |
| **별 수** | **363K** (2026-04-25 시점, 피드백 시점 337K에서 추가 증가) | 보강 | 동일 |
| **라이선스** | MIT | 확정 | 동일 |
| **Ollama 지원 시작** | **v0.20.5부터** (피드백 정정 일치) | 확정 | github.com/ollama/ollama/releases |
| **개발자 OpenAI 합류 (2026-02-14)** | 1차 출처 미확인 | 보류 | — |

---

## 9. Anthropic API 정확화

| 항목 | 검증 결과 | 분류 | 출처 |
|---|---|---|---|
| **Prompt Caching TTL** | **기본 5분 (1.25× 가격), 옵션 1시간 (2× 가격)** | 수정 | platform.claude.com/docs/en/docs/build-with-claude/prompt-caching |
| **Cache 읽기** | **0.1× 가격** | 보강 | 동일 |
| **Alignment Faking 주 모델** | **Claude 3 Opus** (Sonnet 3.5는 보조 실험), 12% scratchpad 관찰, RL 후 78%로 급증 | 수정 | anthropic.com/news/alignment-faking |
| **Opus 4.7 manual budget_tokens** | **400 에러로 미지원** — adaptive thinking + effort 강제 | 수정 | platform.claude.com/docs/en/docs/build-with-claude/extended-thinking |
| **Opus 4.7 interleaved thinking** | **adaptive 모드에서 자동 활성화**, beta header 불필요 | 보강 | 동일 |
| **Opus 4.7 thinking display** | **기본 "omitted"**, summarized 받으려면 명시 필요 | 보강 | 동일 |

---

## 10. Claude Code 공식 문서

| 항목 | 검증 결과 | 분류 |
|---|---|---|
| **Claude Code URL** | **code.claude.com/docs/en** (`docs.anthropic.com/en/docs/claude-code/*` 와 `docs.claude.com/en/docs/claude-code/*` 모두 301 리다이렉트) | 수정 |
| **Anthropic API URL** | **platform.claude.com/docs/en/** | 수정 |
| **Hook 데이터 전달** | **stdin JSON + jq 파싱** (환경변수 아님) | 수정 |
| **Hook 환경변수 (5개만)** | `$CLAUDE_PROJECT_DIR`, `$CLAUDE_PLUGIN_ROOT`, `$CLAUDE_PLUGIN_DATA`, `$CLAUDE_ENV_FILE`, `$CLAUDE_CODE_REMOTE` | 보강 |
| **Hook 이벤트 종 수** | **30+ 종** (SessionStart/End, UserPromptSubmit/Expansion, Stop/StopFailure, PreToolUse/PostToolUse/PostToolUseFailure/PostToolBatch, PermissionRequest/Denied, SubagentStart/Stop, TeammateIdle, TaskCreated/Completed, InstructionsLoaded, ConfigChange, CwdChanged, FileChanged, WorktreeCreate/Remove, PreCompact/PostCompact, Notification, Elicitation/ElicitationResult) | 수정 |
| **Skill 저장 위치** | **4단계: Enterprise > Personal > Project > Plugin** | 수정 |
| **SKILL.md frontmatter** | **15+ 필드**: name, description, when_to_use, argument-hint, arguments, disable-model-invocation, user-invocable, allowed-tools, model, effort, context (fork), agent, hooks, paths, shell | 수정 |
| **Skill 치환자** | `$ARGUMENTS`, `$ARGUMENTS[N]`, `$N` (`$0`/`$1`), `$<name>`, `${CLAUDE_SESSION_ID}`, `${CLAUDE_SKILL_DIR}` | 보강 |
| **Skill 동적 컨텍스트** | `` !`<command>` `` 인라인, ` ```! ` 멀티라인 | 보강 |

---

## 11. 보류 항목 재시도 (라운드3)

| 항목 | 라운드3 재시도 결과 | 분류 |
|---|---|---|
| **Mac Studio M4 Max 128GB 정확 USD** | Apple Store / Best Buy / Microcenter / appleinsider 모두 차단/타임아웃. ~$3,499 추정만 유지 | 여전히 보류 |
| **Mistral Small 4 HumanEval 92.0** | 다수 외부 매체에서 일관 보고 (chats-llm, mindstudio, digitalapplied, marktechpost, neuronad). mistral.ai 공식 페이지는 LCB·AIME 위주 보고로 HumanEval 직접 명시 없음 → "외부 인용 일관" 정도로 신뢰. 1차 명시 없음 | 외부 인용 신뢰 |
| **Gemma 4 HumanEval 78.5** | 외부 인용만, 1차 출처 차단 | 여전히 보류 |
| **DeepSeek V4 GGUF 변환본** | unsloth 아직 미배포, Native FP4/FP8이 1차 채널 시그널 | 여전히 보류 |
| **OpenClaw 개발자 OpenAI 합류 (2026-02-14)** | **확정** — TechCrunch (2026-02-15 보도), Fortune, Sam Altman 트윗 (X), Wikipedia (OpenClaw + Peter Steinberger 두 페이지 모두), 본인 블로그 (steipete.me/posts/2026/openclaw) 다중 1차 출처 확인. OpenClaw는 재단으로 이전, 오픈·독립 유지. OpenAI 후원 | **확정** (보류 해제) |



---

## 12. 적용 방침

이 검증 로그를 근거로 다음 라운드들에서 정정을 적용합니다:
1. **1라운드**: notes/ 8개 파일 정정 (수치·날짜·URL 위주)
2. **2라운드**: appendix/ + README.md 본문 전파
3. **3라운드**: 보류 항목 재시도
4. **4라운드**: 신규 발견 (DeepSeek V4 등) 보강

`두 글 공존 방침` (2026-04-24): lecture-claude-code 는 강의용 톤·분량 보존이 우선. 정정은 (1) 순수 수치 오기 + (2) API 스펙 현행화 위주.
