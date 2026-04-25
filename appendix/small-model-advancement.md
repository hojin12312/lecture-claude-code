# 397B를 이긴 27B: 경량 모델이 대형 모델을 따라잡는 이유

> 연관 문서: [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md),
> [리즈닝 모델의 딜레마](./reasoning-models.md),
> [파라미터 착시: KV 캐시의 진실](./kv-cache.md),
> [API 비용 최적화](./cost-optimization.md),
> [에이전트 프레임워크 지형도](./agent-frameworks.md),
> [RAG 아키텍처](./rag-architecture.md)

2024년 Llama 3.1 405B는 오픈소스 최강 dense 모델이었습니다. HumanEval 89%, MMLU 87%로 GPT-4급 성능을 처음으로 로컬에서 돌릴 수 있게 만들었죠. 로딩에 BF16 810 GB가 필요했습니다.

2026년 4월, Qwen3.6 27B(dense)는 같은 유형의 코딩 벤치마크에서 Qwen3.5 397B-A17B와 동등하거나 더 높은 점수를 냅니다. BF16 기준 53.8 GB — **1/15 크기**. Nemotron Cascade 2는 활성 파라미터 3B만으로 IMO·IOI 2025에서 금메달 수준 성적을 냅니다. 이런 흐름은 단순한 점진적 개선이 아니라 **2025-2026년에 본격화된 현상**입니다.

---

## 구체 사례 — 벤치마크 수치로 본 역전

### 코딩 (LiveCodeBench v6 / SWE-bench)

| 모델 | 크기 (active) | LiveCodeBench v6 | SWE-bench Verified |
|---|---|---|---|
| Llama 3.1 405B | 405B dense | (당시 벤치마크 버전 달라 직접 비교 제한) | — |
| **Qwen3.5 397B-A17B (Reasoning)** | 397B (17B active) | **83.6** | **76.2** |
| **Nemotron Cascade 2** | 30B (3B active) | **87.2** | 50.2 (RL 후), Pass@1 49.9 / Pass@4 65.2 (fine-tune 단계, OpenHands) |
| **Qwen3.6 27B** | 27B dense | **83.9** | **77.2** |
| Gemma4 26B-A4B | 25B (3.8B active) | 77.1 | — |
| Kimi K2.5 1T | ~1,000B (32B active) | 85.0 | — |

- **27B vs 397B 직접 비교**: Qwen3.6 27B (LiveCodeBench v6 83.9 / SWE-bench Verified 77.2) 가 Qwen3.5 397B-A17B (83.6 / 76.2) 와 동급 또는 약간 우위. 활성 파라미터 1/15 (17B → dense 27B 전체 활성), 총 파라미터 1/15. 같은 코드 패밀리에서의 세대 점프.
- Nemotron Cascade 2는 **LiveCodeBench v6에서 Kimi K2.5 1T(활성 32B)를 앞섬**. 활성 파라미터 1/10 이하로.
- Qwen3.6 27B는 SWE-bench Verified 77.2, 코딩 카테고리 전체 111개 모델 중 9위.

### 수학·리즈닝

| 모델 | 크기 (active) | AIME 2025/26 | GPQA Diamond | MMLU-Pro | 올림피아드 |
|---|---|---|---|---|---|
| GPT-4 (2023) | ~1.8T est. | ~15 | ~35 | — | 실패 |
| Llama 3.1 405B | 405B dense | ~30 | ~51 | — | — |
| **Qwen3.5 397B-A17B** | 397B (17B active) | **91.3** (AIME26) | **88.4** | **87.8** | — |
| **Nemotron Cascade 2** | 30B (3B active) | **98.6** (AIME25, tool-RL) | — | — | **IMO/IOI 2025 gold-medal, IOI 439.28/600, ICPC World Finals 10/12 gold** (NVIDIA 공식; 출시 2026-03-16, arXiv 2603.19220) |
| **Qwen3.6 27B** | 27B dense | **94.1** (AIME26) | **87.8** | — | — |
| Gemma4 26B-A4B | 25B (3.8B active) | 88.3 (AIME26) | 82.3 | 82.6 | — |

GPT-4 초기 대비 **AIME 점수가 3-6배 상승**한 동시에 크기는 1/50 이하로 줄었습니다. 특히 Nemotron Cascade 2의 올림피아드 금메달은 "소형 모델이 인간 최상위권 경쟁 수학을 풀 수 있다"는 질적 변화입니다.

### HumanEval (코딩 기본기)

| 모델 | 크기 (active) | HumanEval |
|---|---|---|
| GPT-4 (2023) | ~1.8T est. | ~85 |
| Llama 3.1 405B | 405B dense | 89.0 |
| Qwen3.5 397B-A17B (Reasoning) | 397B (17B active) | 84.9 |
| **Mistral Small 4** | 119B (**6B active per token, 8B with embed**) | **92.0** (외부 인용; mistral.ai 공식에서는 LiveCodeBench/AIME 위주 보고. Mistral Large 3 점수와 혼동 가능성 있음) |
| Gemma4 26B-A4B | 25B (3.8B active) | 78.5 (외부 인용) |

Mistral Small 4는 활성 파라미터 6B만으로 2024년 flagship을 넘어섭니다 (HumanEval 92.0이 정확하다는 전제 — 1차 출처 차단으로 보류).

---

## 왜 이런 일이 가능한가

경량 모델의 약진은 네 가지 기술적 축이 동시에 진보한 결과입니다.

### 1. Post-training RL의 혁신

기초 모델의 사전학습(pre-training)은 여전히 대규모 GPU가 필요합니다. 하지만 **post-training 단계에서 작은 모델에 강력한 reasoning·coding·tool-use 능력을 주입**하는 방법이 급속히 발전했습니다.

- **RLVR (Reinforcement Learning with Verifiable Rewards)**: 정답이 자동 검증 가능한 태스크(수학·코딩·게임)로 RL 강화. 모델이 스스로 `<thinking>` 안에서 백트래킹·반례 탐색을 학습. (자세한 내용은 [리즈닝 모델의 딜레마](./reasoning-models.md) 참조)
- **Cascade RL** (NVIDIA Nemotron): 단계별로 난이도를 올려가며 도메인별 최고 성능 teacher 모델을 교체하는 다단계 RL 파이프라인. Nemotron Cascade 2의 core 기법.
- **Multi-domain On-Policy Distillation**: 수학·코딩·추론 각 도메인에서 해당 도메인 최고 모델의 출력을 on-policy로 distill. 작은 모델이 여러 전문가의 지식을 흡수.

결과: 27-35B 모델이 과거 400B dense 모델의 reasoning 수준에 도달.

### 2. MoE 효율성

**MoE(Mixture of Experts)** 구조는 총 파라미터는 크게 유지하되 추론 시 일부만 활성화합니다. 2025년까지는 "파라미터는 많이, 계산은 적게" 정도의 효과였지만, 2026년에는 **expert 다양성 자체가 품질에 기여**하는 것이 확인됐습니다.

- Gemma 4 26B-A4B: 128 experts + 1 shared expert, token당 8+1 활성
- Mistral Small 4: 128 experts, token당 4 활성
- Nemotron Cascade 2: 30B 총합, token당 3B 활성

같은 활성 파라미터 수(3B)라도 저장된 expert가 더 다양하면 태스크별로 더 전문화된 처리 경로가 선택됩니다. Nemotron Cascade 2가 활성 3B로 수학 올림피아드를 푸는 이유 중 하나입니다.

### 3. 아키텍처 혁신

표준 Transformer의 제약(quadratic attention, KV 캐시 증가)을 우회하는 하이브리드 구조가 실용화됐습니다.

- **DeltaNet 하이브리드** (Qwen3.6 27B): 64개 레이어 중 16개만 표준 어텐션, 나머지는 linear attention variant. KV 캐시가 동급 모델의 1/4 수준. 1M 컨텍스트 확장 가능.
- **Mamba-Transformer 하이브리드** (Nemotron 3 Nano/Cascade 2): State-space 모델(Mamba)로 긴 컨텍스트의 상태를 압축 관리. 1M 컨텍스트 기본 지원.
- **MLA (Multi-head Latent Attention)** (DeepSeek): KV를 저차원 잠재 벡터로 압축. 671B 모델의 KV 캐시를 표준 대비 60배 이상 축소.
- **Sliding Window + Global Attention 교차** (Gemma 4): 대부분 레이어는 로컬만, 일부만 글로벌. Shared KV Cache로 뒷 레이어가 앞 레이어의 KV 재사용.

이런 구조는 **같은 파라미터 예산으로 더 긴 컨텍스트와 더 복잡한 reasoning을 가능하게** 합니다.

> 네 가지 구조의 수식·압축비·레이어 배치 상세는 [KV 캐시의 진실 — 어텐션 구조 혁신](./kv-cache.md#어텐션-구조-혁신-2026년의-kv-압축-계보) 참조.

### 4. Distillation의 대중화

한때 연구자 영역이었던 distillation이 오픈소스 모델 출시 파이프라인의 표준 단계가 됐습니다. 대형 모델이 생성한 고품질 추론 데이터로 작은 모델을 학습시키면, 작은 모델이 대형 모델 출력의 **패턴**은 학습하되 **파라미터 수의 비효율은 상속하지 않습니다**.

예: DeepSeek-R1에서 distill한 32B·14B·7B 모델들이 각각 원본 대비 파라미터의 1/20, 1/50, 1/100 크기로 reasoning 성능의 60-80%를 보존.

---

## 여전한 한계

경량 모델이 모든 면에서 대형을 대체하는 것은 아닙니다.

### Instruction Following은 여전히 크기 의존

[소형 모델의 instruction following 한계](./memory-bandwidth.md#소형-모델의-instruction-following-한계)에서 다룬 현상은 지금도 유효합니다. 벤치마크 점수가 높아도:

- **긴 시스템 프롬프트 무시**: 작은 모델은 3000 토큰 이상의 상세한 지시사항 일부를 잊는 경향.
- **복잡한 형식 제약 실패**: "JSON만 반환", "이 17개 규칙을 모두 지켜라" 같은 제약에서 큰 모델보다 실수율 높음.
- **다중 턴에서 페르소나 붕괴**: 10턴 이상 대화가 이어지면 초기 설정이 희석.

AIME 점수가 98.6이라도, 에이전트 워크플로우에서 "이 5개 도구 중 적절한 것을 고르고, 실패하면 재시도하고, 매 단계마다 로그 남기고..." 같은 메타 지시는 여전히 30B+ 활성 파라미터가 필요한 경우가 많습니다.

### 컨텍스트 희석

긴 컨텍스트에서 attention이 분산되면서 초반 정보 참조가 약해지는 현상은 크기에 관계없이 존재하지만, **작은 모델에서 더 심각**합니다. 1M 컨텍스트를 지원해도 실효적으로 활용 가능한 영역은 초반 100-200K에 집중되는 경우가 많습니다.

### 벤치마크 saturation

HumanEval·MMLU처럼 오래된 벤치마크는 포화 상태입니다. 모델 간 2-3% 차이가 실제 사용 경험의 2-3% 차이를 의미하지 않습니다. 최신 벤치마크(SWE-bench Pro, LiveCodeBench v6, AIME 2026)도 **훈련 데이터 누출**이 부분적으로 의심되는 경우가 있어, 실제 업무에서의 성능과 벤치마크 점수가 항상 일치하지는 않습니다.

### 멀티모달·툴 생태계

일부 멀티모달 기능(고해상도 이미지 이해, 비디오 분석)은 여전히 큰 모델에서만 잘 작동합니다. 또한 에이전트 프레임워크(Claude Code, 커스텀 에이전트)는 대부분 대형 모델 기준으로 튜닝되어 있어, 경량 모델에 동일한 프롬프트를 넣으면 성능이 크게 떨어질 수 있습니다.

---

## 실무 시사점 — 언제 경량, 언제 대형

| 상황 | 권장 크기 | 이유 |
|---|---|---|
| 단순 Q&A, 요약, 분류 | 7-14B | 작은 모델로 충분. 지연·비용 최소화. |
| 수학·과학 문제 해결 | 27-35B (reasoning 활성화) | Nemotron Cascade 2 / Qwen3.6 27B 수준이면 대부분 충분. |
| 코드 작성·리뷰 (단일 태스크) | 27-35B | Qwen3.6 27B, Qwen3-Coder-Next 80B 권장. |
| 에이전트 워크플로우 (툴 사용, 멀티스텝) | 70-120B+ | instruction following 정확도 중요. Mistral Small 4 119B, Llama 4 Scout 109B. |
| 장기 프로젝트 (긴 컨텍스트, 일관성) | 200B+ MoE | Qwen3-235B, DeepSeek V3.1 671B 수준. Claude/GPT-5 같은 클라우드도 여전히 우위. |
| 멀티모달 고급 태스크 | 대형 flagship | 경량 멀티모달은 아직 품질 한계. |

### 경량 모델의 실질적 활용 지점

- **프라이버시·오프라인 요구**: 27B 모델을 M4 Max 128GB 한 대에 올려 로컬 실행.
- **고볼륨 저지연 처리**: 경량 MoE로 tokens/sec가 높아 대량 배치 처리에 유리.
- **파인튜닝 기반 특화**: 27-35B 베이스에 LoRA로 도메인 특화 후 엔터프라이즈 배포.
- **에이전트 서브루틴**: 큰 모델이 오케스트레이션, 경량 모델이 개별 단계 실행. (Claude Sonnet이 계획, Qwen3.6 27B가 각 툴 호출 처리)

### 구매·배포 관점

- **RTX 5090 (32GB)**: Qwen3.6 27B Q4 (~17.5 GB) 또는 Gemma4 26B-A4B Q4 (~17 GB) 단일 카드 실행 가능. 이 급에서 2024년 GPT-4 수준 reasoning 가능. 2026-04 실거래 ~$3,699–3,899.
- **Mac Studio M4 Max 128GB** (데스크톱 단일 다이): Qwen3.6 35B-A3B / Mistral Small 4 Q4 (73.8 GB) 모두 수용. 1대로 "과거 flagship" 수준 달성. 24/7 워크로드 OK.
- **Mac Studio M3 Ultra 256GB** (데스크톱 듀얼 다이, **현 라인업 최대** — 512GB는 2026-03 단종): Llama 3.1 405B Q4 (~245GB) 까지도 빠듯하게 수용. 256GB 옵션은 2026-03부터 +$400 인상.
- **MacBook Pro M5 Max 128GB** (노트북 단일 다이): 사양은 M4 Max와 동급이지만 휴대성. 24/7 sustained 추론 서버로는 부적합 — 모델 개발·테스트 단말 위주.
- **데이터센터 H100/B200/B300**: 과거엔 400B dense가 필수였으나, 이제는 같은 태스크를 30-120B MoE로 처리 가능 → 동일 하드웨어에서 훨씬 많은 동시 요청 처리.

---

## 정리

2026년의 경량 모델 약진은 우연이 아닙니다. Post-training RL의 성숙, MoE 아키텍처의 정착, 하이브리드 구조의 실용화, distillation의 대중화 — 네 축이 동시에 진보한 결과입니다.

"모델 크기 = 성능"이라는 공식은 여전히 부분적으로 맞지만, **1-2년 전 플래그십을 오늘의 경량 모델이 따라잡는 속도는 빨라지고 있습니다**. 실무 설계 시 "이 작업에 정말 200B+가 필요한가, 30B로 충분한가"를 벤치마크로 확인하는 습관이 2026년의 핵심 엔지니어링 스킬이 됐습니다.

동시에 경량 모델의 한계(instruction following, long context, 에이전트 안정성)는 여전히 존재합니다. 모든 워크로드를 경량으로 대체할 수는 없고, **작업 유형에 맞는 크기 선택**이 더 중요해진 것입니다.

---

## 네비게이션

**왔던 길** — 이 부록에 도달하는 전형적인 경로

- "왜 Qwen3.6 27B가 400B급을 따라잡지?" → 여기
- "모델 라우팅에 쓸 소형 모델 근거" → [API 비용 최적화](./cost-optimization.md) → 여기
- "MoE·Hybrid Attention 아키텍처 배경" → [파라미터 착시: KV 캐시의 진실](./kv-cache.md) → 여기

**갈 길** — 이어 읽으면 자연스러운 3개

1. [API 비용 최적화](./cost-optimization.md) — 모델 라우팅으로 소형 모델을 비용 전략에 편입
2. [리즈닝 모델의 딜레마](./reasoning-models.md) — 소형 + reasoning 조합이 flagship을 대체하는 경로
3. [에이전트 프레임워크 지형도](./agent-frameworks.md) — flagship 오케스트레이션 + 소형 실행 분업의 2026 표준

---

## 연관 문서

- [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md) — 소형 모델의 하드웨어 실행 가능성을 결정하는 물리 조건
- [리즈닝 모델의 딜레마](./reasoning-models.md) — 작은 모델 + reasoning의 조합이 2026년 플래그십을 대체하는 경로
- [파라미터 착시: KV 캐시의 진실](./kv-cache.md) — MoE·Hybrid Attention이 작은 활성 파라미터로 같은 KV 압력을 처리하는 원리
- [API 비용 최적화](./cost-optimization.md) — 모델 라우팅으로 소형 모델을 비용 전략에 편입하는 패턴
- [에이전트 프레임워크 지형도](./agent-frameworks.md) — 오케스트레이션은 flagship, 단계 실행은 소형 모델로 분업하는 2026년 표준 구성
- [RAG 아키텍처](./rag-architecture.md) — 쿼리 재작성·리랭커·요약 단계에 소형 모델을 쓰는 법
