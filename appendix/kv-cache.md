# 파라미터 착시: KV 캐시의 진실

> 이것이 이 강의에서 가장 중요한 슬라이드입니다.

> 연관 문서: [트랜스포머 추론의 두 단계](./transformer-inference.md),
> [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md),
> [리즈닝 모델의 딜레마](./reasoning-models.md),
> [경량 모델의 빠른 발전](./small-model-advancement.md),
> [API 비용 최적화](./cost-optimization.md),
> [RAG 아키텍처](./rag-architecture.md),
> [에이전트 프레임워크 지형도](./agent-frameworks.md)

---

## "모델 크기 = KV 캐시 크기"는 틀렸다

두 모델을 나란히 놓겠습니다.

| 항목 | Gemma4:31b (Dense) | GLM-5.1 (MoE 744B) |
|---|---|---|
| 모델 가중치 (BF16) | ~61.4 GB | ~1,488 GB (1.49 TB) |
| 모델 가중치 (Q4 양자화) | ~19.6 GB | ~451 GB |
| **KV 캐시 (256K / 203K 컨텍스트)** | **~240 GB** | **~990 GB** |
| **KV / 모델 비율** | **4배 이상** | **0.7배 (캐시 < 모델)** |

모델 크기가 30배 차이 나는 두 모델의 KV 캐시 비율이 반전되어 있습니다.

**"19.6GB 모델인데 KV 캐시가 240GB"** — 모델 가중치보다 단기 기억이 12배 무겁습니다. 파라미터 크기는 착시입니다.

> **KV 캐시**란: 트랜스포머의 어텐션(attention) 계산 시, 이전 토큰들의 Key·Value 벡터를 저장해두는 메모리. "이미 읽은 것을 다시 읽지 않겠다"는 캐시인데, 컨텍스트가 길어질수록 저장할 과거 토큰이 폭발적으로 늘어납니다.

---

## KV 캐시 폭발의 수학

KV 캐시 크기는 다음 수식으로 결정됩니다:

```
KV cache (bytes) = T x L x 2 x H_kv x d x 2
```

| 기호 | 의미 | GLM-5.1 스펙 |
|---|---|---|
| T | 시퀀스 길이 (토큰 수) | 200,000 |
| L | 레이어(layer) 수 | 78 |
| 2 | Key + Value 두 벡터 | — |
| H_kv | KV 헤드(head) 수 | 64 |
| d | 헤드 차원(head_dim) | 256 |
| 2 bytes | bf16 정밀도 | — |

**핵심은 T에 선형 비례**한다는 것입니다. 컨텍스트를 2배 늘리면 KV 캐시도 2배. 4K → 200K라면 50배.

모델 가중치는 추론 중 **고정**입니다. KV 캐시만이 세션마다, 토큰마다 쌓입니다.

---

## GPU는 왜 90% 놀고 있을까

GPU가 느린 이유는 **연산이 부족해서**가 아닙니다. **데이터를 꺼내오는 속도(대역폭)**가 병목입니다.

```
[GPU Memory (HBM)]
        |
        |  <-- bandwidth: how fast data moves (GB/s)
        |
[GPU Compute Cores]   <-- waiting for data
```

KV 캐시가 클수록, 어텐션 계산마다 더 많은 데이터를 메모리에서 불러와야 합니다. 대역폭이 부족하면 연산 코어가 데이터를 기다리며 놉니다. 극단적으로는 GPU 활용률의 90%가 메모리 이동 대기에 잡아먹힙니다.

연산 코어를 아무리 추가해도 대역폭이 병목이면 의미가 없습니다. 코어가 아니라 대역폭을 넓혀야 합니다.

**산술 강도(Arithmetic Intensity)**: FLOPs ÷ Bytes(메모리 이동량). 어텐션은 산술 강도가 낮은 연산입니다 — 계산 자체보다 메모리 읽기/쓰기가 훨씬 많습니다. 이것이 HBM 대역폭이 AI 추론 성능의 핵심 지표가 되는 이유입니다.

---

## MoE: 파라미터가 많아도 KV는 왜 작은가

GLM-5.1은 744B 파라미터인데 왜 KV 캐시가 모델보다 작을까요?

KV 캐시 수식을 다시 봅니다:

```
KV cache = T x L x 2 x H_kv x d x 2
```

이 수식에 **FFN(Feed-Forward Network) 파라미터가 없습니다.** KV 캐시는 오직 attention 헤드 구조에만 비례합니다.

MoE(Mixture of Experts) 모델은 파라미터의 대부분이 **FFN expert**에 집중됩니다. 추론 시 토큰당 일부 expert만 활성화되어 FFN 계산에 쓰이고, KV 벡터로 캐시되지는 않습니다.

**GLM-5.1 744B의 파라미터 배분 (추정):**

```
GLM-5.1 744B
+-- Attention:  ~84B  (11%)  <-- KV 캐시는 이것만 반영
+-- FFN expert: ~660B (89%)  <-- KV 캐시에 기여 없음
```

MoE로 모델 지능(파라미터 수)을 키우는 건 KV 캐시에 거의 청구서가 오지 않습니다. 반면 컨텍스트를 늘리는 건 T에 선형 비례하여 **직접 청구서**가 날아옵니다.

---

## Hybrid Attention: 256K를 지원해도 전부 캐시하지 않는다

Gemma4:31b는 256K 컨텍스트를 지원하지만 실제 KV 캐시는 naive 계산의 20% 수준입니다. 이유는 **Hybrid Attention** 구조 때문입니다.

Gemma4 시리즈는 두 종류의 어텐션 레이어를 혼합합니다:

| 레이어 타입 | 설명 | KV 캐시 |
|---|---|---|
| Sliding Window Attention (SWA) | 최근 1024 토큰만 봄 | 컨텍스트 길이와 무관, 고정 크기 |
| Global Attention | 전체 컨텍스트를 봄 | T에 선형 비례 |

**Gemma4:31b (60층):** SWA 50층 + Global 10층 (5:1 패턴 × 10회 반복)

```
KV cache (256K):
  SWA 50층:    50 x 1024 x 2 x 16 x 256 x 2 =  0.78 GB  (고정)
  Global 10층: 10 x 256K x 2 x 16 x 256 x 2 = 39.06 GB  (T에 비례)
  실제 합계:                                   39.84 GB

  naive 계산 (60층 x 256K):                  240 GB
  절감:                                       83%
```

**Gemma4:26b A4B (30층):** SWA 26층 + Global 4층, MoE (Active 4B)

```
KV cache (256K):
  SWA 26층:    26 x 1024 x 2 x 16 x 256 x 2 =  0.41 GB
  Global 4층:   4 x 256K x 2 x  4 x 512 x 2 =  7.81 GB
  실제 합계:                                    8.22 GB
```

Gemma4:26b A4B는 MoE(Global 레이어 KV 헤드가 4개뿐) + Hybrid Attention(Global 레이어가 4층뿐)의 이중 효과로 256K 컨텍스트에서 KV 캐시가 **8GB** 수준입니다. 31B Dense 대비 6배 작습니다.

---

## 어텐션 구조 혁신: 2026년의 KV 압축 계보

Gemma4의 SWA+Global은 **레이어별 attention 범위**를 차별화해 KV 캐시를 줄이는 방식이었습니다. 2024-2026년에는 이와 다른 축 — **어텐션 헤드 구조 자체를 압축**하거나 **어텐션을 다른 연산으로 대체**하는 — 의 혁신이 본격화됐습니다. 세 갈래를 순서대로 따라갑니다.

### 갈래 1. 헤드 구조 압축: MHA → MQA → GQA → MLA

모두 표준 softmax 어텐션은 유지하되, **KV 헤드 저장 형태**를 단계적으로 줄여온 계보입니다.

| 방식 | KV 저장 | 수식의 `H_kv` | 품질 | 대표 모델 |
|---|---|---|---|---|
| MHA (Multi-Head) | 쿼리 헤드마다 K·V 한 쌍씩 | = H_q (모든 헤드 수) | 기준 | Llama 2, GPT-3 |
| MQA (Multi-Query) | 모든 헤드가 K·V **1쌍** 공유 | 1 | 품질 소폭 손실 | PaLM |
| GQA (Grouped-Query) | 헤드를 G개 그룹으로 묶어 그룹당 K·V 1쌍 | G (보통 4~8) | MHA와 거의 동일 | Llama 3, Mistral, Gemma 4 |
| **MLA (Multi-head Latent)** | **저차원 잠재 벡터 1개** (필요 시 복원) | 잠재 차원 `d_c` | MHA 이상 | **DeepSeek V2/V3/R1, GLM-5.1** |

GQA까지는 KV 수식의 `H_kv` 숫자를 줄여서 캐시를 선형 압축하는 방식입니다 (Gemma4:31b의 `H_kv=16`, 쿼리 헤드 32개도 GQA 사례). **MLA는 접근 자체가 다릅니다** — K·V를 따로 저장하지 않고, 두 값을 복원할 수 있는 **공통 잠재 벡터 하나**만 캐시합니다.

#### MLA의 동작

```
c^KV_t = W^DKV · h_t          # 입력을 d_c 차원으로 '압축' (캐시에 저장)
K_t    = W^UK · c^KV_t        # attention 계산 시 잠재벡터에서 '복원'
V_t    = W^UV · c^KV_t        # 동일
```

캐시에는 `c^KV` (d_c 차원) 하나만 쌓이고, attention 계산 시점에 up-projection 행렬로 K와 V를 복원합니다. 저장량 관점에서 일반 MHA의 `2 · H_kv · d_h` 대비 `d_c` 하나로 줄어듭니다.

**압축비**: `r = (d_h × H) / d_c`

DeepSeek-V3는 `d_h=128`, `H=128`, `d_c=4·d_h=512` → **r = 32배**. DeepSeek-V2 기준 KV 캐시 **93.3% 감소**, 처리량 **5.76배**.

> **Decoupled RoPE 각주**: RoPE(위치 인코딩)는 원래 K에 직접 적용되는데, K를 잠재 공간으로 압축하면 RoPE를 미리 박을 수 없습니다(저장 시점과 복원 시점의 위치가 다름). 해결책은 K를 두 부분으로 나누는 것 — 대부분은 잠재 공간으로 압축하고, RoPE 전용 작은 부분(d_h/2 차원)만 별도로 유지합니다.

### 갈래 2. 어텐션 자체 대체: Linear Attention 하이브리드

Softmax 어텐션은 토큰 T개에 대해 **O(T²)** 계산과 T에 선형 비례하는 KV 캐시를 요구합니다. **Linear attention** 계열은 이 둘을 동시에 끊습니다 — 어텐션을 **고정 크기 hidden state를 업데이트하는 RNN 형태**로 바꿔, 계산은 O(T), 캐시는 레이어당 고정 크기로 만듭니다.

대표 구현이 **Gated DeltaNet** (Mamba-2의 state 업데이트 구조 + Delta rule + 게이팅). 단독으로는 long-range reasoning 품질이 softmax에 뒤처지므로, **표준 어텐션 레이어와 교차 배치**하는 하이브리드가 실용 해법이 됐습니다.

| 모델 | 레이어 구성 | 컨텍스트 | KV 캐시 |
|---|---|---|---|
| Qwen3-Next 80B-A3B | Gated DeltaNet 3 : Full Attention 1 | 256K (최대 1M) | 1M에서 ~25 GB (순수 Transformer 대비 4배 작음) |
| Qwen3.6 27B | 64 레이어 중 표준 attention 16개, 나머지 DeltaNet | 262K (최대 1M) | 동급 dense 대비 약 1/4 |
| Qwen3.6 35B-A3B | Gated DeltaNet + Gated Attention + MoE | 1M | 1M 기본 지원 |

구조적 의미: **대부분 레이어가 KV 캐시에 거의 기여하지 않는다**. 표준 어텐션 레이어 몇 개에서만 T에 선형 비례하는 캐시가 발생하고, 나머지는 레이어당 수백 KB 수준의 고정 state만 유지합니다.

### 갈래 3. SSM 교차: Mamba-Transformer 하이브리드

**SSM(State Space Model)** 은 어텐션과 별개의 계보입니다 — 과거 전체를 고정 차원 hidden state에 **압축 요약**하는 시퀀스 모델. **Mamba-2** 는 이 hidden state 업데이트 규칙을 입력 의존적으로 만들어 "무엇을 기억하고 무엇을 잊을지"를 동적으로 고릅니다.

```
h_t = A · h_{t-1} + B(x_t) · x_t       # hidden state 업데이트
y_t = C(x_t) · h_t                     # 출력
```

`B`, `C`, 업데이트 스텝 `Δ`가 입력에 따라 달라지는 것이 **selective scan**. 시퀀스 길이에 대해 **O(T)** 복잡도, 토큰 수에 무관한 **고정 크기 state**.

NVIDIA Nemotron 3 계열은 이 Mamba-2 레이어를 대부분으로 쓰고, 일부만 GQA로 유지하는 **Mamba-Transformer 하이브리드**입니다.

| 모델 | 파라미터 (active) | 컨텍스트 | 비고 |
|---|---|---|---|
| Nemotron 3 Nano | 31.6B (3.2B) | 1M | Mamba-2 + GQA + MoE, KV/compute 4배 효율 |
| Nemotron 3 Super | 120B (12B) | 1M | 동일 구조, 더 큰 규모 |
| Nemotron Cascade 2 | 30B (3B) | 262K | Nano 베이스 + Cascade RL, IMO 금메달 수준 |

Mamba 레이어는 **KV 캐시 자체가 없습니다** — 레이어당 state 차원 하나만 유지하므로, 1M 컨텍스트에서도 메모리 압력이 GQA 레이어 몇 개에서만 발생합니다.

### 세 갈래 한 표로

| 방향 | 핵심 | KV 감소 메커니즘 | 대표 |
|---|---|---|---|
| 헤드 구조 압축 | K·V를 잠재 차원으로 묶음 | `H_kv` 자체를 축소 (MLA는 `d_c` 1개로) | DeepSeek, GLM-5.1 |
| Linear Attention 교차 | 일부 레이어를 RNN 유사 구조로 대체 | 해당 레이어는 KV 없이 고정 state | Qwen3-Next, Qwen3.6 |
| Mamba 하이브리드 | SSM 레이어가 대부분, 일부만 어텐션 | Mamba 레이어는 KV 없음 | Nemotron 3 |
| (Gemma4 방식) | 어텐션은 유지, 범위만 차별화 | SWA 레이어는 고정 창 + Shared KV | Gemma 3/4 |

세 갈래 모두 **"컨텍스트는 길어져도 KV 캐시는 선형 비례하지 않게"** 만들겠다는 목표가 같습니다. 어디서 타협을 두느냐가 다를 뿐입니다.

> 상호 참조: 경량 모델이 아키텍처 혁신을 어떻게 활용했는지는 [경량 모델의 빠른 발전](./small-model-advancement.md#3-아키텍처-혁신) 참조.

---

## 남은 소프트웨어 레벨 최적화

위 세 갈래가 **모델 구조 자체를 바꾸는** 접근이라면, 구조는 그대로 두고 런타임에서 캐시 사용을 줄이는 소프트웨어 기법도 함께 씁니다.

### Flash Attention
어텐션 계산을 타일(tile) 단위로 쪼개 HBM↔SRAM 왕복 횟수를 최소화합니다. KV 캐시 크기 자체는 줄지 않지만, 읽기 패턴을 최적화해 대역폭 낭비를 줄입니다.

### Prefix Caching (Prompt Caching)
시스템 프롬프트처럼 **반복되는 입력**의 KV 벡터를 서버가 재사용합니다. Claude, OpenAI, Gemini API 모두 지원. 대화마다 같은 컨텍스트를 다시 계산하지 않아 비용과 지연 시간을 줄입니다.

> 공급자별 할인율(일반적으로 read 90%), 최소 토큰, 자동/수동 구분, 배치 API와의 스택 조건은 [API 비용 최적화 — 프롬프트 캐싱](./cost-optimization.md#2-프롬프트-캐싱--반복되는-입력의-90-할인) 참조.

**한계**: 소프트웨어 최적화는 결국 "이미 계산된 KV를 낭비하지 않기" 수준입니다. 컨텍스트와 배치가 함께 커지면 결국 물리적 메모리 용량·대역폭의 벽에 부딪힙니다 — 여기서부터는 아키텍처 혁신이 담당합니다.

---

## 지능의 속도는 HBM 대역폭이 결정한다

이것이 소프트웨어 엔지니어가 메모리 엔지니어에게 보내는 러브레터의 핵심입니다. 컨텍스트가 더 길어질수록, 더 많은 에이전트가 병렬로 돌수록, 단기 기억을 퍼 나르는 하드웨어의 속도가 지능의 실제 천장이 됩니다.

---

## 네비게이션

**왔던 길** — 이 부록에 도달하는 전형적인 경로

- "왜 긴 컨텍스트를 쓰면 비용이 이렇게 튀지?" → [API 비용 최적화](./cost-optimization.md) → 여기
- "이 모델이 내 하드웨어에 올라갈까?" → [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md) → 여기
- "Prefill/Decode가 왜 나뉘어 있지?" → [트랜스포머 추론의 두 단계](./transformer-inference.md) → 여기

**갈 길** — 이어 읽으면 자연스러운 3개

1. [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md) — KV 압력을 실제로 받는 하드웨어의 모습
2. [API 비용 최적화](./cost-optimization.md) — Prompt Caching이 KV를 공급자 서버에 재활용해 90% 할인시키는 구조
3. [RAG 아키텍처](./rag-architecture.md) — KV 누적 비용을 회피하는 아키텍처 선택

---

## 연관 문서

- [트랜스포머 추론의 두 단계: Prefill과 Decode](./transformer-inference.md) — KV 캐시가 생기는 메커니즘. prefill·decode 단계가 이 문서 수식의 전제
- [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md) — KV 캐시가 실제로 부딪히는 하드웨어 병목. VRAM·HBM 수치와 직결
- [리즈닝 모델의 딜레마](./reasoning-models.md) — thinking 토큰이 KV 캐시를 잠식하는 방식
- [경량 모델의 빠른 발전](./small-model-advancement.md) — MLA·Hybrid Attention·MoE가 KV 압력을 완화하는 원리
- [API 비용 최적화](./cost-optimization.md) — Prompt Caching이 KV를 공급자 서버에 재활용해 90% 할인시키는 구조
- [RAG 아키텍처](./rag-architecture.md) — long-context vs RAG 선택의 물리적 근거. KV 누적 비용이 RAG를 더 싸게 만드는 지점
- [에이전트 프레임워크 지형도](./agent-frameworks.md) — 멀티턴 에이전트에서 KV 누적이 처리량 한계를 결정하는 이유
- [notes/kv-compression-architectures.md](../notes/kv-compression-architectures.md) — MLA·DeltaNet·Mamba·SWA 수식과 비교표
