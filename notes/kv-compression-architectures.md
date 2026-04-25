# KV 캐시 압축 아키텍처 참조

2024-2026년 등장한 KV 캐시 압축·대체 아키텍처 정리. `appendix/kv-cache.md` 업데이트 시 이 문서 먼저 참조.

마지막 조사: 2026-04-25

---

## 1. MLA (Multi-head Latent Attention) — DeepSeek

**도입**: DeepSeek-V2 (2024-05), 이후 V3·R1·V3.1·V3.2에 지속.

### 핵심 아이디어

표준 MHA는 각 토큰마다 Key·Value 벡터를 그대로 캐시한다. MLA는 **저차원 잠재 벡터 c^KV**로 압축해 캐시하고, 실제 attention 계산 시점에 up-projection으로 복원한다.

### 수식 개요

```
c^KV_t = W^DKV · h_t         # down-projection (저장용, d_c 차원)
K_t    = W^UK · c^KV_t       # up-projection (attention 계산용)
V_t    = W^UV · c^KV_t       # up-projection
```

- `W^DKV`: 입력 h_t를 d_c 차원으로 내리는 가중치
- `W^UK`, `W^UV`: 잠재 공간에서 K·V를 복원
- 캐시에는 오직 `c^KV` (d_c 차원) 만 저장 → K·V 두 개를 d_h×H 차원으로 저장하는 표준보다 훨씬 작음

### 압축비 공식

```
r = (d_h × H) / d_c
```

- `d_h`: head 차원
- `H`: KV head 수
- `d_c`: 잠재 차원

**DeepSeek-V3 예시**: `d_h = 128`, `H = 128`, `d_c = 4·d_h = 512`
→ 압축비 `r = 128·128 / 512 = 32배`

r > 1.5 이상이어야 실질 이득 발생.

### Decoupled RoPE

RoPE(위치 인코딩)는 원래 K에 직접 적용되는데, K를 잠재 공간으로 압축하면 RoPE를 저장된 c^KV에 미리 적용할 수 없다 (K로 복원되는 시점의 위치가 아니라 저장 시점의 위치가 박히기 때문).

해결: K를 두 부분으로 분리
- **NoPE 부분 (non-positional)**: 잠재 공간으로 압축
- **RoPE 부분 (positional, dR_h = d_h / 2 차원)**: 별도로 유지, RoPE 직접 적용

### KV 캐시 감소 효과

- DeepSeek-V2 67B Dense 대비 KV 93.3% 감소
- 처리량(throughput) 최대 5.76배
- 대형 MoE 기준 표준 MHA 대비 KV 약 4% 수준

### 채택 모델
- DeepSeek-V2 / V2-Lite / V3 / R1 / V3.1 / V3.2
- GLM-5.1 (ZhipuAI, 2026) — MLA 압축 채택, 744B MoE

---

## 2. Gated DeltaNet 하이브리드 — Qwen3-Next 계열

**도입**: Qwen3-Next 80B-A3B (2025-09), Qwen3.6-27B (2026-04-23), Qwen3.6-35B-A3B (2026-04-14).

### 핵심 아이디어

Transformer 블록 일부를 **linear attention (Gated DeltaNet)** 으로 대체. Linear attention은 softmax attention의 O(T²) 복잡도를 O(T)로 낮추는 대신, 고정 크기 hidden state에 과거 정보를 요약.

### Gated DeltaNet

- 기반: Mamba-2 + **Delta rule** (error-correcting update)
- RNN 계열 memory + 게이팅(gate)으로 망각률 조절
- Qwen3-Next: head당 1개 scalar gate (memory decay 속도 제어)
- Kimi Linear: 같은 계열이지만 feature dimension별 channel-wise gating

### 레이어 배치 비율

- **Qwen3-Next 80B-A3B**: 3:1 (DeltaNet 3개당 Full Attention 1개)
- **Qwen3.6-27B**: 64 레이어 중 16개만 표준 (full) attention, 나머지는 DeltaNet계 → 약 1:3 (full:linear)
- **Qwen3.6-35B-A3B**: Gated DeltaNet + Gated Attention + MoE 조합

### KV 캐시 효과

- Qwen3-Next 80B-A3B: 1M 컨텍스트에서 전체 KV 약 25 GB
  - 순수 Transformer 동급 모델 대비 4배 작음
- Apple Silicon 128 GB에서 Qwen3-Next 80B-A3B가 92 GB wired로 실행 가능

### 채택 모델
- Qwen3-Next 80B-A3B
- Qwen3.6-27B (dense, DeltaNet 하이브리드)
- Qwen3.6-35B-A3B (MoE + DeltaNet)
- Kimi Linear (참고: 같은 linear attention 계열이지만 게이팅 방식 다름)

---

## 3. Mamba-Transformer 하이브리드 — NVIDIA Nemotron

**도입**: Nemotron Nano 2 (2025-08), Nemotron 3 Nano/Super/Cascade 2 (2025-12 ~ 2026-03).

### 핵심 아이디어

SSM(State Space Model) 계열의 **Mamba-2** 레이어를 대부분으로 쓰고, 일부 레이어만 **GQA**(Grouped Query Attention) 로 유지. SSM은 고정 크기 hidden state에 과거 정보를 압축 저장 → 시퀀스 길이에 대해 선형 복잡도.

### Mamba-2 동작

```
h_t = A · h_{t-1} + B · x_t    # hidden state 업데이트
y_t = C · h_t                  # 출력
```

- `h_t`: N차원 hidden state (N은 state expansion factor; Mamba-2에서 64~256)
- `A`: scalar × identity 구조 (Mamba-2 제약)
- `B`, `C`, `Δ`(step size): **입력 의존적** → 선택적 스캔 (selective scan)
  - 토큰마다 B·C·Δ가 달라져 "무엇을 기억/망각할지" 동적 결정
- 복잡도: O(T) (Transformer의 O(T²) 대비 유리)

### 아키텍처 구성

- Nemotron 3 Nano 31.6B (3.2B active): Mamba-2 + GQA + MoE (FFN을 MoE로 대체)
- Nemotron 3 Super 120B (12B active): 동일 패턴, 더 큰 규모
- Nemotron Cascade 2 30B (3B active): Nano 베이스에 Cascade RL post-training

### KV 캐시 효과

- Mamba-2 레이어는 KV 캐시 없이 고정 크기 state만 유지 → 컨텍스트 길이 무관
- 일부 GQA 레이어만 KV 캐시 필요 → 표준 Transformer 대비 **4배 메모리·연산 효율**
- 1M 컨텍스트 기본 지원 (Nemotron 3 Nano, Super)

### 채택 모델
- NVIDIA Nemotron Nano 2
- NVIDIA Nemotron 3 Nano / Super / Cascade 2
- (참고: Jamba, Zamba 등 다른 하이브리드 SSM 계열도 동일 아이디어)

---

## 4. Sliding Window + Global Attention + Shared KV — Gemma 4

**도입**: Gemma 3 (2025), Gemma 4 (2026-04) 이어감.

### 핵심 아이디어

표준 attention을 유지하되, 대부분 레이어는 **로컬 윈도우** (예: 최근 1024 토큰)만, 일부 레이어만 **글로벌 attention**. 뒷 레이어는 앞 레이어의 KV를 **재사용** (Shared KV).

### 레이어 구성 (Gemma 4 26B-A4B 기준)

- 총 30 레이어
- SWA 26층 (sliding window = 1024 토큰)
- Global 4층 (전체 컨텍스트)
- 128 experts + 1 shared expert, 토큰당 8+1 active
- 마지막 N 레이어: 앞 레이어의 KV 재사용 (Shared KV Cache)

### KV 캐시 효과 (256K 컨텍스트)

```
SWA 26층:    26 × 1024 × 2 × 16 × 256 × 2 =  0.41 GB (고정)
Global 4층:   4 × 256K × 2 × 4  × 512 × 2 =  7.81 GB (T 비례)
합계:                                        8.22 GB
```

표준 30층 × 256K 대비 약 95% 감소.

### 채택 모델
- Gemma 3 / 4 전 라인업
- Gemma 4 26B-A4B (MoE + SWA + Shared KV 삼중 효과)

---

## 5. Mamba-3 (ICLR 2026) — State Space Model 고도화

**도입**: ICLR 2026 발표 (openreview.net/forum?id=HwCvaJOiCj). 아직 프론티어 상용 모델에 반영된 건 없지만, 다음 세대 Mamba 기반 아키텍처 설계에 바로 영향을 줄 가능성이 큼.

### 핵심 개선 3가지

1. **SSM discretization 기반 더 표현력 있는 recurrence** — Mamba-2 대비 더 풍부한 memory update 표현
2. **Complex-valued state update** — 복소수 hidden state로 state tracking (예: 카운팅·위상 추적) 능력 강화
3. **MIMO formulation** — multi-input, multi-output 확장 (이전엔 SISO)

### 성능 (1.5B 규모)

- Mamba-3 > Gated DeltaNet: average downstream accuracy +0.6 pp
- Mamba-3 MIMO: 추가 +1.2 pp

### 의미

- Gated DeltaNet(ICLR 2025)은 Qwen3.6·Kimi Linear에 이미 반영됨
- Mamba-3는 "Gated DeltaNet 다음 세대" — 2026 하반기 프론티어 오픈소스 모델(Qwen 4·Nemotron 4 등)이 채택할 가능성
- 수업에서는 "linear attention 계보는 계속 빠르게 진화 중"의 근거로 언급

---

## 6. 계보 비교 — 한눈에

| 방식 | 접근 | KV 저장 대상 | 대표 모델 |
|---|---|---|---|
| MHA | 표준 | 각 head별 K·V 전체 | Llama 2 |
| MQA | head 공유 | 모든 head가 K·V 1쌍 공유 | PaLM |
| GQA | 그룹 공유 | G개 그룹 × K·V | Llama 3, Mistral, Gemma |
| **MLA** | 잠재 압축 | 저차원 c^KV 1개 | DeepSeek V2/V3, GLM-5.1 |
| **Linear Attention 교차** | attention 자체 대체 | 일부 레이어만 KV, 나머지는 RNN state | Qwen3-Next, Qwen3.6 |
| **Mamba 하이브리드** | SSM 교차 | 일부 GQA 레이어만 KV, 대부분 Mamba state | Nemotron 3 |
| **SWA + Global + Shared KV** | 레이어별 attention 범위 차별화 | 로컬 KV (고정) + 글로벌 KV (적은 수) | Gemma 3/4 |

---

## 주요 출처

- DeepSeek-V2 논문: https://arxiv.org/pdf/2405.04434
- DeepSeek-V3 기술 보고서: https://arxiv.org/pdf/2412.19437
- MLA 해설 (Sebastian Raschka): https://sebastianraschka.com/llms-from-scratch/ch04/05_mla/
- MLA 시각 해설: https://planetbanatt.net/articles/mla.html
- Qwen3-Next vLLM 블로그: https://blog.vllm.ai/2025/09/11/qwen3-next.html
- Qwen3.6-27B 블로그: https://qwen.ai/blog?id=qwen3.6-27b
- Mamba-2 (Tri Dao): https://tridao.me/blog/2024/mamba2-part1-model/
- Mamba 원 논문: https://arxiv.org/abs/2312.00752
- Nemotron Nano 2: https://arxiv.org/abs/2508.14444
- Nemotron 3 Nano: https://arxiv.org/html/2512.20848v1
- Nemotron 3 Super: https://developer.nvidia.com/blog/introducing-nemotron-3-super-an-open-hybrid-mamba-transformer-moe-for-agentic-reasoning/
- Mamba-3 (ICLR 2026): https://openreview.net/forum?id=HwCvaJOiCj
- Gated Delta Networks (ICLR 2025): https://arxiv.org/abs/2412.06464
