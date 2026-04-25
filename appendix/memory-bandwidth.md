# LLM 구동의 주요 스펙: 메모리 용량과 대역폭

> 연관 문서: [파라미터 착시: KV 캐시의 진실](./kv-cache.md),
> [트랜스포머 추론의 두 단계](./transformer-inference.md),
> [경량 모델의 빠른 발전](./small-model-advancement.md),
> [리즈닝 모델의 딜레마](./reasoning-models.md),
> [로컬 LLM 실행 생태계](./local-llm-platforms.md)

---

## 두 가지 질문

LLM을 로컬에서 돌리려면 두 질문에 순서대로 답해야 합니다.

1. **용량**: 모델이 메모리에 올라가는가? (부족하면 실행 자체가 불가)
2. **대역폭**: 올라간다면 얼마나 빠르게 추론하는가?

용량이 먼저, 대역폭은 그 다음입니다.

---

## 용량 — 모델이 메모리에 들어가는가

### 가중치 크기 (양자화별)

모델 파일을 메모리에 올리는 데 필요한 최소 용량은 **파라미터 수 × 정밀도(바이트)**입니다.

**양자화(quantization)**를 적용하면 정밀도를 낮춰 크기를 줄일 수 있습니다. llama.cpp / Ollama에서 가장 널리 쓰이는 방식은 **Q4_K_M** — 파라미터 하나당 약 0.6 bytes를 사용하는 4비트 양자화입니다.

> 양자화 = 모델의 수치를 덜 정밀하게 저장해 용량을 줄이는 기법. 품질은 약간 저하되지만 실사용 차이는 모델·작업에 따라 다름.

| 모델 | 파라미터 | BF16 | Q8_0 | Q4_K_M | Q3_K_M | Q2_K |
|---|---|---|---|---|---|---|
| Llama 3.2 3B | 3.2B | 6.4 GB | 3.4 GB | 2.0 GB | — | — |
| Phi-4-mini | 3.8B | ~7.6 GB | ~4.1 GB | ~2.5 GB | — | — |
| Qwen3 4B | 4.0B | 8.0 GB | 4.3 GB | 2.4 GB | — | — |
| Qwen3 8B | 8.2B | 16.4 GB | 8.7 GB | 5.0 GB | — | — |
| Qwen3.5 9B | 9.0B | ~18 GB | 9.6 GB | 5.9 GB | — | — |
| Phi-4 14B | 14.0B | ~29 GB | 15.6 GB | 9.1 GB | — | — |
| Qwen3 14B | 14.8B | 29.6 GB | 15.7 GB | 9.0 GB | — | — |
| Gemma4 26B-A4B (MoE) | 25.2B (3.8B active) | 50.5 GB | 26.9 GB | 17.0 GB | — | — |
| Qwen3.6 27B | 27.0B | 53.8 GB | 28.7 GB | 17.5 GB | — | — |
| Gemma4 31B | 30.7B | 61.4 GB | 32.6 GB | 19.6 GB | — | — |
| GLM-4.7-Flash (MoE) | 30B (3B active) | ~60 GB | ~32 GB | ~18.5 GB | — | — |
| Nemotron Cascade 2 (MoE) | 30B (3B active) | 63.2 GB | 33.6 GB | 24.7 GB | — | — |
| Nemotron 3 Nano (MoE) | 31.6B (3.2B active) | 63.2 GB | 33.5 GB | 24.7 GB | — | — |
| Qwen3 32B | 32.8B | 65.6 GB | 34.9 GB | 19.9 GB | — | — |
| Qwen3.6 35B-A3B (MoE) | 35B (3B active) | 69.4 GB | 36.9 GB | 21.4 GB | — | — |
| Llama 3.3 70B | 70.6B | 141 GB | 75 GB | 42.5 GB | 34.3 GB | 26.4 GB |
| Qwen3-Coder-Next (MoE) | ~80B (3B active) | ~160 GB | 84.8 GB | 48.7 GB | — | — |
| Llama 4 Scout (MoE) | 109B (17B active) | 216 GB | 115 GB | 65.4 GB | 51.8 GB | 39.6 GB |
| Mistral Small 4 (MoE) | 119B (6.5B active) | 238 GB | 126 GB | 73.8 GB | 54.4 GB | 40.2 GB |
| Nemotron 3 Super (MoE) | 120B (12B active) | ~240 GB | 128.5 GB | 87.0 GB | — | — |
| Qwen3.5 122B-A10B (MoE) | 122B (10B active) | ~244 GB | 129.9 GB | 75.0 GB | — | — |
| MiniMax M2.7 (MoE) | 229B (~10B active) | 457 GB | 243 GB | 139 GB | 104 GB | 80 GB |
| Qwen3-235B-A22B (MoE) | 235B (22B active) | ~470 GB | ~250 GB | ~142 GB | — | — |
| Qwen3.5 397B-A17B (MoE) | 397B (17B active) | ~794 GB | ~422 GB | ~241 GB | ~194 GB | ~166 GB |
| Llama 4 Maverick (MoE) | 402B (17B active) | 801 GB | 426 GB | 243 GB | 191 GB | 146 GB |
| Llama 3.1 405B | 405B | 810 GB | 430 GB | 245 GB | 198 GB | — |
| DeepSeek-V3.1 (MoE) | 671B (37B active) | 1,342 GB | 713 GB | 407 GB | 328 GB | 281 GB |
| DeepSeek-R1-0528 (MoE) | 671B (37B active) | 1,342 GB | 713 GB | 407 GB | 328 GB | 281 GB |
| GLM-5.1 / GLM-5.1-Instruct (MoE) | 744B (40B active) | ~1,488 GB | ~791 GB | ~451 GB | ~363 GB | ~310 GB |
| Kimi K2 / K2.5 (MoE) | ~1,000B (32B active) | ~2,050 GB | ~1,090 GB | ~580 GB | ~490 GB | ~420 GB |

MoE 모델의 파라미터 수는 총합입니다. 추론 시 활성화되는 파라미터는 훨씬 작지만(Llama 4 Scout ~17B, Mistral Small 4 ~6.5B, Llama 4 Maverick ~17B, DeepSeek ~37B), **가중치 전체를 메모리에 올려야 하므로** 총 파라미터 기준 용량이 필요합니다.

### KV 캐시

가중치를 올린 뒤, 추론 중 KV 캐시가 추가로 쌓입니다. 수식과 메커니즘은 [파라미터 착시: KV 캐시의 진실](./kv-cache.md)에서 다룹니다.

컨텍스트 크기별 KV 캐시 크기 (BF16 기준; Q8 양자화 시 약 절반). 모델이 지원하지 않는 컨텍스트는 —로 표기:

| 모델 | 32K | 64K | 128K | 256K | 최대 컨텍스트 |
|---|---|---|---|---|---|
| Llama 3.2 3B | 1.2 GB | 2.4 GB | 4.8 GB | — | 128K |
| Phi-4-mini | 4.0 GB | 8.0 GB | 16.0 GB | — | 128K |
| Qwen3 4B | 4.5 GB | 9.0 GB | 18.0 GB | — | 128K |
| Qwen3.5 9B | ~4.5 GB | ~9.0 GB | ~18.0 GB | ~36.0 GB | 262K |
| Qwen3 8B | 4.5 GB | 9.0 GB | 18.0 GB | — | 128K |
| Phi-4 14B | 6.3 GB | — | — | — | 16K (RoPE 확장 시 ~32K) |
| Qwen3 14B | 5.0 GB | 10.0 GB | 20.0 GB | — | 128K |
| Qwen3.6 27B | ~2.0 GB | ~4.0 GB | ~8.0 GB | ~16.0 GB | 262K (DeltaNet 하이브리드, 64레이어 중 16만 표준 어텐션) |
| Gemma4:31b | 5.8 GB | 10.8 GB | 20.8 GB | 40.8 GB | 256K (Hybrid attention, SWA:Global = 50:10) |
| GLM-4.7-Flash (MoE) | ~5.0 GB | ~10.0 GB | ~20.0 GB | — | 128K |
| Nemotron Cascade 2 (Mamba hybrid) | — | — | — | — | 262K (Mamba 레이어는 KV 캐시 없음) |
| Nemotron 3 Nano (Mamba hybrid) | — | — | — | — | 1M (Mamba 레이어는 KV 캐시 없음) |
| Qwen3 32B | 8.0 GB | 16.0 GB | 32.0 GB | — | 128K |
| Qwen3.6 35B-A3B (MoE) | ~8.0 GB | ~16.0 GB | ~32.0 GB | ~64.0 GB | 1M |
| Llama 3.3 70B | 10.0 GB | 20.0 GB | 40.0 GB | — | 128K |
| Llama 3.1 405B | 15.8 GB | 31.6 GB | 63.2 GB | — | 128K |
| DeepSeek-V3.1/R1-0528 | ~2.0 GB | ~4.0 GB | ~8.0 GB | — | 128K (MLA 압축) |
| GLM-5.1 (MoE, MLA) | ~3.0 GB | ~6.0 GB | ~12.0 GB | — | 203K (MLA 압축) |
| Kimi K2 / K2.5 (MoE) | ~10.0 GB | ~20.0 GB | ~40.0 GB | ~80.0 GB | 262K |

llama.cpp에서 `-ctk q8_0` 옵션으로 KV 캐시에도 Q8 양자화를 적용할 수 있습니다.

> **MLA (Multi-head Latent Attention)**: DeepSeek이 독자 개발한 구조. KV를 저차원 잠재 벡터로 압축해 캐시하므로 671B 모델임에도 KV 캐시가 표준 구조 대비 60배 이상 작습니다.

### 실제 필요 용량 = 가중치 + KV 캐시

예시 — Llama 3.3 70B Q4_K_M, 32K 컨텍스트, KV Q8:
- 가중치: 42.5 GB
- KV 캐시: 5.0 GB
- **합계: ~47.5 GB** → M4 Max 128 GB에 여유 있게 수용, RTX 5090(32 GB)은 초과

---

## 대역폭 — 얼마나 빠르게 추론하는가

Decode 단계(토큰을 한 개씩 생성하는 구간)에서 GPU는 매 스텝마다 **모델 가중치 전체**를 메모리에서 읽어야 합니다. 연산 코어(CUDA core, GPU shader)는 수조 번의 곱셈을 1초에 처리할 수 있지만, 메모리에서 데이터를 꺼내오는 속도 — **대역폭** — 가 그만큼 빠르지 않습니다. 짐을 나르는 길이 좁으면 코어는 데이터를 기다리며 놉니다.

토큰 생성 속도의 이론적 상한:

```
tokens/sec ≈ Memory Bandwidth (GB/s) / Model Size (GB)
```

Gemma4:31b (BF16, ~61 GB) 기준:

| 하드웨어 | 대역폭 | 이론 tok/s |
|---|---|---|
| 데스크톱 PC DDR5 dual-ch | ~100 GB/s | ~1.6 |
| M4 Max (546 GB/s 구성) | 546 GB/s | ~8.9 |
| NVIDIA H100 SXM (HBM3) | 3,350 GB/s | ~54 |

---

## 데스크톱 PC에 RAM을 많이 꽂으면 되지 않나

안 됩니다. **꽂는 용량과 대역폭은 별개입니다.**

소비자용 데스크톱 플랫폼은 메모리 채널이 2개입니다. DDR5-6400 듀얼 채널 기준 최대 대역폭은 약 102 GB/s입니다. 슬롯 4개를 모두 채워서 128 GB, 256 GB를 만들어도 **채널 수는 그대로**이므로 대역폭은 그대로 ~100 GB/s입니다.

```
소비자 데스크톱 플랫폼 (DDR5 dual-ch):

  DIMM x2 (64 GB each = 128 GB total)
  +-----------+    +-----------+
  |  DIMM A   |    |  DIMM B   |
  | 32 GB/s   |    | 32 GB/s   |   <-- 각 채널 절반씩
  +-----------+    +-----------+
        |                |
        +------CPU-------+
              ~100 GB/s total        <-- 채널 수가 한계
```

용량은 늘어났지만, 데이터를 나르는 "도로 수"는 늘지 않았습니다.

더 큰 문제는 GPU와의 관계입니다. CPU 메모리에 올라간 모델 가중치를 GPU가 연산하려면 **PCIe 버스**를 경유해야 합니다:

```
CPU RAM (DDR5) --> PCIe 5.0 x16 --> GPU VRAM (GDDR6X/HBM)
                  max ~64 GB/s
                  (이것이 병목)
```

PCIe 5.0 x16의 최대 양방향 대역폭은 약 64 GB/s입니다. GPU VRAM이 부족해서 CPU RAM에 일부를 offload하면, 그 부분은 PCIe 속도로 제한됩니다. 64 GB/s는 H100의 HBM 대역폭(3,350 GB/s)의 1.9% 수준입니다.

### 워크스테이션·서버 플랫폼은 다르다

AMD EPYC(8채널), Intel Xeon(8채널) 같은 서버 CPU는 채널 수가 많습니다.

| 플랫폼 | 채널 수 | DDR5-6400 기준 대역폭 |
|---|---|---|
| 소비자 데스크톱 | 2채널 | ~100 GB/s |
| 워크스테이션 (Threadripper) | 4채널 | ~200 GB/s |
| 서버 (AMD EPYC 9004) | 12채널 | ~576 GB/s |

그러나 서버 CPU의 메모리는 여전히 PCIe를 거쳐야 GPU에 도달합니다. GPU VRAM에 담기지 않는 모델을 CPU RAM에 offload하는 방식(llama.cpp의 `--n-gpu-layers` 부분 오프로딩)은 대역폭 손실이 심각합니다.

---

## Apple Silicon이 다른 이유 — 통합 메모리 아키텍처 (UMA)

Apple Silicon은 CPU, GPU, Neural Engine이 **같은 물리 메모리 풀을 공유**합니다. 별도의 GPU VRAM이 없는 대신, SoC 패키지 위에 LPDDR5/LPDDR5X를 직접 붙여 모든 코어가 동일한 고대역폭 메모리에 접근합니다.

```
기존 PC 구조:

  [CPU] <--> [DDR5 DRAM]  ~100 GB/s
               |
             PCIe ~64 GB/s
               |
  [GPU] <--> [GDDR6X VRAM]  ~1,000 GB/s


Apple Silicon UMA 구조:

  [CPU cores  ]
  [GPU cores  ] <--> [LPDDR5X 통합 메모리]  ~120~819 GB/s
  [Neural Eng ]
```

PCIe 전송이 없습니다. GPU 코어가 동일한 버스로 전체 메모리를 직접 읽습니다. 128 GB를 탑재한 M4 Max는 그 128 GB를 최대 546 GB/s 대역폭으로 GPU가 바로 사용합니다.

### Apple Silicon에서 LLM 실행: llama.cpp vs MLX

Apple Silicon에서 LLM을 로컬로 돌리는 주요 방법이 두 가지입니다.

| | llama.cpp / Ollama | MLX (Apple 공식 ML 프레임워크) |
|---|---|---|
| UMA 활용 | Metal backend로 부분 활용 | 설계 단계부터 UMA 네이티브 |
| 추론 속도 | Apple Silicon에서 양호 | Apple Silicon에서 더 빠른 경향 (체감 10-30%) |
| 파인튜닝 | 제한적 | LoRA 네이티브 지원 (양자화 모델도 가능) |
| 모델 포맷 | GGUF | HuggingFace safetensors 직접 지원 |
| 양자화 | Q4_K_M 등 GGUF K-quant | 4-bit / 8-bit / FP8 / INT8 등 네이티브 |
| 특징 | 크로스 플랫폼, 광범위한 커뮤니티 | Apple Silicon 전용, 메모리 효율 최적 |

MLX는 Apple이 직접 만든 오픈소스 프레임워크로, CPU와 GPU가 같은 메모리 버스를 공유하는 UMA 구조를 코드 레벨에서 네이티브로 활용합니다. `mlx-lm` 패키지 하나로 HuggingFace 모델 실행·양자화·LoRA 파인튜닝이 모두 가능합니다.

### MLX 사용 예시

```bash
pip install mlx-lm

# 추론 (HuggingFace에서 모델 자동 다운로드)
mlx_lm.generate --model mlx-community/Qwen3.6-27B-4bit \
    --prompt "Explain attention mechanism" --max-tokens 500

# 양자화 (safetensors → 4-bit MLX 포맷)
mlx_lm.convert --hf-path Qwen/Qwen3.6-27B -q --q-bits 4

# LoRA 파인튜닝 (양자화된 모델에도 가능)
mlx_lm.lora --model mlx-community/Qwen3.6-27B-4bit \
    --train --data ./dataset --iters 1000
```

`mlx-community/` 네임스페이스에는 주요 모델의 MLX 포맷 변환본이 사전 업로드되어 있어 별도 변환 없이 바로 실행 가능합니다.

### llama.cpp Metal 백엔드

llama.cpp는 `GGML_METAL=1` 빌드 옵션으로 Metal GPU 가속을 활성화합니다 (Homebrew/pip 빌드에서는 기본 활성화).

```bash
# 소스 빌드 시
cmake -B build -DGGML_METAL=1
cmake --build build --config Release

# 실행
./llama-cli -m qwen3.6-27b-q4_k_m.gguf -ngl 999 -p "Hello"
```

`-ngl 999` (모든 레이어를 GPU로 오프로드) 옵션으로 UMA 메모리에 올려 GPU 연산을 수행합니다. Ollama는 내부적으로 llama.cpp + Metal을 쓰므로 별도 설정 없이 자동 활용됩니다.

### 실측 성능 비교 (M4 Max 128GB 기준)

| 모델 (Q4/4bit) | llama.cpp Metal | MLX 4-bit | 비고 |
|---|---|---|---|
| Qwen3.6 27B | ~32 tok/s | ~40 tok/s | MLX가 20-25% 빠름 |
| Gemma4 31B | ~28 tok/s | ~35 tok/s | MLX 유리 |
| Llama 3.3 70B | ~12 tok/s | ~15 tok/s | 둘 다 대역폭 병목 |

> 절대값은 컨텍스트 길이·배치·프롬프트에 따라 ±20% 변동. MLX가 빠른 이유는 unified memory 접근 최적화 + Apple 자체 Metal Performance Shaders 튜닝.

### 선택 기준

- **추론만 필요, 크로스 플랫폼 공유**: llama.cpp / Ollama (GGUF 파일 하나로 Mac·Linux·Windows 공유)
- **속도 우선, Mac 전용 서빙**: MLX
- **파인튜닝 포함**: MLX (양자화 모델에 LoRA 네이티브 지원)
- **GUI 필요**: LM Studio (내부적으로 llama.cpp + MLX 둘 다 지원, 최근엔 MLX 우선)

---

## Apple Silicon M 시리즈 전체 라인업

| 칩 | 출시 | 최대 메모리 | 메모리 대역폭 |
|---|---|---|---|
| M1 | 2020 | 16 GB | 68 GB/s |
| M1 Pro | 2021 | 32 GB | 200 GB/s |
| M1 Max | 2021 | 64 GB | 400 GB/s |
| M1 Ultra (M1 Max × 2) | 2022 | 128 GB | 800 GB/s |
| M2 | 2022 | 24 GB | 100 GB/s |
| M2 Pro | 2023 | 32 GB | 200 GB/s |
| M2 Max | 2023 | 96 GB | 400 GB/s |
| M2 Ultra (M2 Max × 2) | 2023 | 192 GB | 800 GB/s |
| M3 | 2023 | 24 GB | 100 GB/s |
| M3 Pro | 2023 | 36 GB | 150 GB/s |
| M3 Max (30-core GPU) | 2023 | 128 GB | 300 GB/s |
| M3 Max (40-core GPU) | 2023 | 128 GB | 400 GB/s |
| M3 Ultra (M3 Max × 2) | 2025 | 512 GB | 819 GB/s |
| M4 | 2024 | 32 GB | 120 GB/s |
| M4 Pro | 2024 | 64 GB | 273 GB/s |
| M4 Max (32-core GPU) | 2024 | 128 GB | 410 GB/s |
| M4 Max (40-core GPU) | 2024 | 128 GB | 546 GB/s |
| M4 Ultra | — | — | — | 개발 취소 |
| M5 | 2025 | 32 GB | 153 GB/s |
| M5 Pro | 2026 | 64 GB | 307 GB/s |
| M5 Max (32-core GPU) | 2026 | 128 GB | 460 GB/s |
| M5 Max (40-core GPU) | 2026 | 128 GB | 614 GB/s |
| M5 Ultra | 출시 예정 | — | ~1,100 GB/s (예상) | Mac Studio 2026 탑재 예정 — 2026-04-19 Bloomberg: 공급 이슈로 최악 10월 슬립 가능 |

Ultra 칩은 두 개의 Max 다이를 Apple의 UltraFusion 인터커넥트로 연결한 구성입니다. M4 Ultra는 개발 취소됐으며, M5 Ultra가 Mac Studio에 탑재될 예정입니다. 출시는 2026년 중반~후반이 유력하며, 대역폭은 M3 Ultra(819 GB/s) 대비 약 +35% 수준인 1,100 GB/s 내외로 루머됩니다.

---

## NVIDIA 소비자용 GPU 라인업 (RTX 40 / 50 시리즈)

소비자 GPU의 VRAM은 **전용 GDDR 메모리**입니다. 시스템 RAM과 완전히 분리된 별도 칩으로, 용량은 작지만 대역폭은 높습니다.

### RTX 40 시리즈 (Ada Lovelace, 단종 → 재고 유통 중)

| 모델 | VRAM | 메모리 타입 | 대역폭 | 시장가 (USD) |
|---|---|---|---|---|
| RTX 4070 | 12 GB | GDDR6X | 504 GB/s | ~$600–700 |
| RTX 4070 Ti Super | 16 GB | GDDR6X | 672 GB/s | ~$800–900 |
| RTX 4080 Super | 16 GB | GDDR6X | 736 GB/s | ~$1,000–1,100 |
| RTX 4090 | 24 GB | GDDR6X | 1,008 GB/s | ~$2,500–3,000 |

### RTX 50 시리즈 (Blackwell, 2025 현행)

| 모델 | VRAM | 메모리 타입 | 대역폭 | MSRP (USD) | 실거래가 (USD, 2026-04) |
|---|---|---|---|---|---|
| RTX 5060 Ti (16 GB) | 16 GB | GDDR7 | 448 GB/s | $429 | ~$449–$599 |
| RTX 5070 | 12 GB | GDDR7 | 672 GB/s | $549 | ~$629–$699 |
| RTX 5070 Ti | 16 GB | GDDR7 | 896 GB/s | $749 | ~$999–$1,169 |
| RTX 5080 | 16 GB | GDDR7 | 960 GB/s | $999 | ~$1,200–$1,300 |
| RTX 5090 | 32 GB | GDDR7 | 1,792 GB/s | $1,999 | ~$3,500–$5,000 |

RTX 50 시리즈는 GDDR7 채용으로 대역폭이 전 세대 대비 50–80% 향상됐습니다. 전반적으로 MSRP를 크게 웃도는 실거래가가 지속되고 있습니다. RTX 5090은 GDDR7 공급 부족 영향으로 MSRP($1,999) 대비 실거래가가 2배 이상입니다.

**소비자 GPU의 결정적 한계**: VRAM 용량 상한이 32 GB(RTX 5090)입니다. BF16 기준 30B 모델(~60 GB)도 들어가지 않습니다. Q4 양자화해서 15–20 GB로 줄이거나, 모델 자체를 작게 써야 합니다.

---

## NVIDIA 전문가용 GPU (RTX PRO / 구 Quadro 라인)

소비자용(최대 32 GB)과 데이터센터(HBM, $30,000+) 사이의 공백을 채우는 라인입니다. GDDR 메모리를 쓰지만 ECC 지원·대용량 VRAM으로 워크스테이션·온프레미스 추론 서버에 적합합니다.

| 모델 | VRAM | 메모리 타입 | 대역폭 | 가격 (USD) |
|---|---|---|---|---|
| RTX 4500 Ada | 24 GB | GDDR6 ECC | 432 GB/s | ~$2,000–2,500 |
| RTX 6000 Ada | 48 GB | GDDR6 ECC | 960 GB/s | ~$7,800–10,000 |
| RTX PRO 6000 Blackwell | 96 GB | GDDR7 ECC | 1,792 GB/s | ~$8,000–9,300 |

"Quadro"는 2021년 NVIDIA RTX (Professional)로 리브랜딩됐고, 2025년부터 "RTX PRO" 이름을 사용합니다. RTX PRO 6000 Blackwell은 96 GB로 RTX 5090의 3배 VRAM을 제공하며, Blackwell 아키텍처의 NVFP4까지 지원합니다.

**LLM 관점**: RTX 6000 Ada(48 GB)는 BF16 기준 30B 모델을 단일 카드에 올릴 수 있습니다. RTX PRO 6000 Blackwell(96 GB)은 BF16 70B 모델도 단일 카드에 수용 가능합니다.

---

## NVIDIA 데이터센터 GPU (H / B 시리즈)

VRAM으로 HBM(High Bandwidth Memory)을 사용합니다. GDDR 대비 대역폭이 3–8배 높고, 용량도 80–180 GB에 달합니다. 가격도 그에 비례합니다.

| 모델 | VRAM | 메모리 타입 | 대역폭 | 단가 (USD) |
|---|---|---|---|---|
| A100 40GB (구세대) | 40 GB | HBM2 | 1,600 GB/s | ~$10,000–12,000 |
| A100 80GB (구세대) | 80 GB | HBM2e | 2,000 GB/s | ~$15,000–17,000 |
| H100 PCIe | 80 GB | HBM2e | 2,000 GB/s | ~$27,000–35,000 |
| H100 SXM | 80 GB | HBM3 | 3,350 GB/s | ~$30,000–40,000 |
| H200 SXM | 141 GB | HBM3e | 4,800 GB/s | ~$30,000–40,000 |
| B200 | 180 GB | HBM3e | 8,000 GB/s | ~$30,000–55,000 |
| B300 (Blackwell Ultra) | 288 GB | HBM3e+ | 8,000 GB/s | ~$40,000–50,000+ |

A100은 2020년 출시로 구세대지만 여전히 클라우드·사내 서버에서 광범위하게 운용됩니다. H100은 현재 클라우드 AI 서비스의 사실상 표준이고, B200은 2025년 투입 중입니다. B300(Blackwell Ultra)은 2026년 1월부터 출하 시작되었으며, B200과 대역폭은 동일하지만 메모리 용량이 288 GB로 대폭 증가해 대형 모델 추론에 특화됩니다.

SXM 폼팩터는 NVLink 연결을 통해 여러 GPU를 하나의 메모리 풀처럼 사용할 수 있어 대형 모델 서빙에 유리합니다. PCIe 폼팩터는 일반 서버에 꽂을 수 있지만 GPU 간 대역폭이 낮습니다.

### NVIDIA 추론 스택: TensorRT-LLM / vLLM / llama.cpp

NVIDIA GPU에서 LLM을 돌리는 방식은 목적에 따라 세 갈래로 나뉩니다. 각각 서로 다른 문제를 풉니다.

#### TensorRT-LLM — 저지연·프로덕션

NVIDIA의 LLM 추론 전용 **컴파일러·런타임**. 모델을 GPU별로 AOT(ahead-of-time) 컴파일해 kernel fusion·메모리 레이아웃·양자화를 극단적으로 최적화합니다. 단일 요청 지연(latency)이 가장 중요한 프로덕션 API에서 사실상 표준.

```bash
# 예: Llama 3.3 70B를 H100 2장에 맞춰 FP8 엔진으로 컴파일
trtllm-build --checkpoint_dir ./llama33_fp8_ckpt \
    --output_dir ./engines/llama33_fp8 \
    --gemm_plugin fp8 --use_paged_context_fmha enable \
    --max_batch_size 32 --max_input_len 8192

# 실행 (Triton Inference Server 또는 trtllm-serve 래퍼)
trtllm-serve --engine_dir ./engines/llama33_fp8
```

양자화 포맷별 대상 GPU:

| 포맷 | 비트 | 대상 GPU | 특징 |
|---|---|---|---|
| **INT8** | 8비트 정수 | 모든 NVIDIA GPU | 기준 양자화, 정확도·속도 균형 |
| **FP8** | 8비트 부동소수점 | H100, RTX 4090+ | 정확도 손실 최소, H100에서 FP16 대비 ~2× 처리량 |
| **INT4 AWQ** | 4비트 정수 (블록 단위) | RTX 소비자 GPU 포함 | 소형 배치에서 FP8보다 낮은 지연, VRAM 절반 |
| **NVFP4** | 4비트 부동소수점 (NVIDIA 독자) | Blackwell (B200/B300) | FP8 대비 수학 처리량 ~2×, 메모리 절반 |

**메모리 대역폭 관점**: FP4는 같은 VRAM에서 FP16 대비 4배 많은 파라미터를 유효하게 처리합니다. B300의 288 GB HBM3e + NVFP4 조합은 이전 세대 FP16 대비 모델 용량·처리량 모두 8× 이상 향상되는 이유입니다.

#### vLLM — 고처리량·멀티 요청

UC Berkeley 출신 오픈소스 엔진. **동시 다수 요청 처리량(throughput)** 최적화에 특화됐습니다. 두 가지 핵심 기법:

- **PagedAttention**: KV 캐시를 OS 가상 메모리처럼 고정 크기 페이지로 쪼개 관리. 요청별로 다른 길이의 KV를 파편화 없이 메모리에 채워 넣어, 동일 VRAM에서 2-4배 많은 동시 요청을 담을 수 있습니다.
- **Continuous Batching (연속 배칭)**: 기존엔 "배치 하나 끝날 때까지 대기" 였다면, vLLM은 요청 단위로 매 스텝마다 배치를 재구성. 짧은 요청이 먼저 끝나고 빈 슬롯에 새 요청이 바로 들어갑니다.

```bash
pip install vllm

# 실행 (OpenAI 호환 API 서버)
vllm serve Qwen/Qwen3.6-27B --dtype bfloat16 \
    --tensor-parallel-size 2 --max-num-seqs 64

# 클라이언트 (OpenAI SDK 그대로 사용)
curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "Qwen/Qwen3.6-27B", "messages": [...]}'
```

vLLM은 **같은 H100 한 장에서 TensorRT-LLM 대비 단일 요청 지연은 약간 높지만, 동시 요청 10개 이상일 때 전체 tokens/sec 처리량은 30-50% 높은** 경향이 있습니다. SaaS API 서비스, 챗봇 백엔드, 배치 추론에 적합.

#### llama.cpp / Ollama — 로컬·개발

크로스 플랫폼 C++ 추론 엔진. NVIDIA CUDA 빌드(`GGML_CUDA=1`)로 GPU 가속 사용. 위 두 스택 대비 최적화는 낮지만 **설치·실행 간편성, 모델 포맷(GGUF) 단일성**이 강점. Ollama는 llama.cpp 위에 관리 레이어를 얹은 구조.

#### 선택 기준

| 목적 | 권장 스택 | 이유 |
|---|---|---|
| 단일 요청 저지연 (프로덕션 API) | TensorRT-LLM | kernel fusion으로 가장 낮은 latency |
| 동시 다수 요청 (챗봇·SaaS) | vLLM | PagedAttention + continuous batching |
| 로컬 개발·실험 | llama.cpp / Ollama | 설치 한 줄, GGUF 하나로 이동 가능 |
| 파인튜닝 + 추론 혼합 | vLLM + PEFT (LoRA 지원) | HuggingFace 생태계 호환 |
| Blackwell NVFP4 활용 | TensorRT-LLM | NVFP4는 TRT-LLM에서만 full 지원 |

> **주의**: TensorRT-LLM은 모델·GPU 조합마다 엔진을 다시 컴파일해야 합니다. 모델 변경이 잦은 개발 단계에는 vLLM이 훨씬 편합니다. 프로덕션으로 굳어지면 TRT-LLM으로 전환해 지연을 줄이는 게 일반적인 패턴입니다.

---

## 기타 플랫폼 선택지

### AMD ROCm

AMD GPU(RDNA 3 / CDNA 시리즈)에서 LLM을 구동하는 오픈소스 스택입니다. **ROCm (Radeon Open Compute)**은 NVIDIA의 CUDA에 대응하는 AMD의 GPU 컴퓨팅 플랫폼. 가성비 면에서 강점(W7900이 RTX 6000 Ada 대비 VRAM 동급이지만 가격 절반)이지만, 생태계 성숙도는 CUDA 대비 1-2년 뒤처져 있습니다.

**AMD 전문가용 GPU 라인업 (Radeon Pro)**

| 모델 | VRAM | 메모리 타입 | 대역폭 | 가격 (USD) |
|---|---|---|---|---|
| Radeon Pro W7800 | 32 GB | GDDR6 ECC | 576 GB/s | ~$2,500 |
| Radeon Pro W7900 | 48 GB | GDDR6 ECC | 864 GB/s | ~$3,499 |
| Radeon AI PRO R9700 | 32 GB | GDDR6 | 640 GB/s | ~$1,500–2,000 |

Radeon Pro W7900은 48 GB ECC GDDR6로 RTX 6000 Ada와 동급 VRAM이지만 가격이 절반 수준입니다. Radeon AI PRO R9700은 2025년 출시된 RDNA4 기반 AI 특화 모델입니다.

**ROCm 지원 추론 스택 현황 (2026-04)**:

- **llama.cpp HIP 백엔드**: 가장 성숙. `GGML_HIPBLAS=1`로 빌드. GGUF 포맷 그대로 사용. 실측 성능은 같은 대역폭의 NVIDIA GPU 대비 70-85% 수준.
- **vLLM ROCm**: 공식 지원. RDNA3/CDNA2 이상. MI300X(192 GB HBM3) 기준 H100 대비 동급 처리량 보고 사례 있음. 단, 양자화 포맷 일부 제한 (AWQ 일부 미지원).
- **PyTorch ROCm**: 2.5+에서 안정화. HuggingFace Transformers 대부분 그대로 동작하지만 FlashAttention 같은 일부 커널은 CUDA 전용.
- **TensorRT-LLM**: 미지원 (NVIDIA 독점).

W7900 실측 (Qwen3.6 27B Q4, llama.cpp HIP): ~25-30 tok/s. 같은 모델이 RTX 4090(대역폭 1,008 GB/s)에서 ~35-40 tok/s, RTX 5090(1,792 GB/s)에서 ~60 tok/s 수준.

### Intel OpenVINO + NPU

**OpenVINO**: Intel이 만든 "CPU·iGPU·NPU 통합 추론 런타임". ONNX 또는 PyTorch 모델을 OpenVINO IR(중간 표현)로 변환해 Intel 칩에 최적화된 커널로 실행.

Intel Core Ultra (Meteor Lake / Lunar Lake / Arrow Lake) 프로세서에 내장된 **NPU(Neural Processing Unit)**는 별도 전력 라인을 쓰며, CPU/GPU를 쉬게 하면서 AI 추론을 저전력으로 수행합니다. OpenVINO 2026은 7B 이하 모델에서 attention 레이어를 NPU로 오프로드해 **GPU 단독 대비 최대 3.8× 처리량** 보고.

```bash
pip install openvino-genai

# HuggingFace 모델을 OpenVINO IR로 변환
optimum-cli export openvino --model Qwen/Qwen3-8B \
    --weight-format int4 ./qwen3-8b-ov

# 실행 (자동으로 가용한 NPU/GPU/CPU 선택)
python -c "
from openvino_genai import LLMPipeline
pipe = LLMPipeline('./qwen3-8b-ov', 'NPU')  # 또는 'GPU', 'CPU', 'AUTO'
print(pipe.generate('What is attention?', max_new_tokens=200))
"
```

대형 모델(30B+)은 NPU로 올리기엔 용량 부족. 노트북/미니 PC의 **엣지 LLM** 시나리오(개인 비서, 로컬 RAG)에 적합. 최근 Copilot+ PC의 로컬 AI 기능 대부분이 이 스택 기반.

### 모바일·엣지 NPU (Qualcomm / Samsung / MediaTek)

ARM ISA 기반 모바일 SoC들은 대부분 유사한 전략을 씁니다 — CPU + GPU + **전용 NPU**를 한 칩에 통합해 AI 추론을 저전력으로 처리. LLM 관점에서는 세 제조사가 사실상 같은 카테고리입니다.

| 제조사 | 대표 칩 | NPU 성능 | 주요 시장 |
|---|---|---|---|
| Qualcomm | Snapdragon X2 Elite (PC), 8 Gen 4 (모바일) | 75-85 TOPS | AI PC, Android 플래그십 |
| Samsung | Exynos 2500 | ~60 TOPS | 갤럭시 일부 지역 |
| MediaTek | Dimensity 9400 | ~50 TOPS | 중국·동남아 Android 주력 |

**추론 스택 현황**:

- **Qualcomm QNN SDK (AI Engine Direct)**: Qualcomm 독자 런타임. PyTorch/ONNX → QNN 변환. 가장 성숙한 모바일 LLM 스택.
- **MediaTek NeuroPilot**: MediaTek 독자 SDK. 자체 변환 툴체인.
- **Samsung Exynos NPU SDK**: 상대적으로 오픈 생태계 빈약. 갤럭시 자체 기능에 주로 쓰임.
- **llama.cpp 모바일 포팅** (LLMfarm, ChatterUI 등): 세 제조사 모두에서 CPU+GPU 경로로 실행 가능하지만, **NPU 가속은 제한적**. 대부분 CPU/GPU에서 Q4_0 기준 실행.

실측 (Q4, 7B 이하):

- Snapdragon X Elite + Llama 3.2 3B (Q4): ~15-20 tok/s, 배터리 구동 가능
- Dimensity 9400 + Qwen3 4B (Q4): ~10-15 tok/s
- Exynos 2500 + 유사 모델: ~8-12 tok/s

데이터센터·워크스테이션과는 맥락이 다르지만, **배터리 구동 AI PC·스마트폰**에서 LLM을 돌리는 현실적 선택지입니다. 2026년 기준 아이폰·갤럭시 최신 플래그십도 유사한 NPU 가속 구조로 3-7B 모델의 온디바이스 실행이 표준이 되는 중입니다.

> **서버용 ARM (NVIDIA Grace, Ampere Altra, AWS Graviton)**: ARM ISA 기반 서버 CPU. 주로 일반 컴퓨팅·클라우드 인프라 용도이며, LLM 추론은 여전히 같은 시스템에 연결된 GPU가 담당. 강의 범위(로컬 LLM 구동)에서는 우선순위 낮음.

### 플랫폼별 추천 추론 스택 요약

| 하드웨어 | 최적 스택 | 비고 |
|---|---|---|
| Apple Silicon (Mac) | MLX (속도) 또는 llama.cpp (호환성) | UMA 덕분에 128GB까지 단일 노드 가능 |
| NVIDIA H100/B200 (데이터센터) | TensorRT-LLM (지연) + vLLM (처리량) | 양자화는 FP8/NVFP4 |
| NVIDIA RTX 소비자 GPU | vLLM (소규모 서빙) / llama.cpp (개발) | RTX 5090 32GB 이하는 양자화 필수 |
| AMD Radeon Pro | llama.cpp HIP, vLLM ROCm | CUDA 전용 기능 불가 확인 필요 |
| Intel Core Ultra NPU | OpenVINO GenAI | 엣지·노트북 7B 이하 |
| Qualcomm/Samsung/MediaTek NPU | QNN SDK / NeuroPilot / llama.cpp 포팅 | 모바일·AI PC, 7B 이하 |

---

## 플랫폼 종합 비교 — 대역폭 · 용량 · 가격

| 하드웨어 | GPU 메모리 | 대역폭 | 가격 (USD) | $/GB |
|---|---|---|---|---|
| Mac mini M4 (32 GB) | 32 GB UMA | 120 GB/s | ~$1,099 | ~$34 |
| Mac Studio M4 Max (128 GB) | 128 GB UMA | 546 GB/s | ~$3,499 | ~$27 |
| MacBook Pro M5 Max (128 GB) | 128 GB UMA | 614 GB/s | ~$3,999 | ~$31 |
| Mac Studio M3 Ultra (96 GB base) | 96 GB UMA | 819 GB/s | $3,999 (28-core CPU/60-core GPU 베이스) | ~$42 |
| Mac Studio M3 Ultra (256 GB) | 256 GB UMA | 819 GB/s | base + ~$2,000 (2026-03 +$400 인상) — **현 라인업 최대** | — |
| ~~Mac Studio M3 Ultra (512 GB)~~ | ~~512 GB UMA~~ | ~~819 GB/s~~ | **2026-03 단종** (글로벌 메모리 칩 부족) | — |
| RTX 5060 Ti (16 GB) | 16 GB GDDR7 | 448 GB/s | ~$379–599 실거래 | ~$24–37 |
| RTX 5070 Ti | 16 GB GDDR7 | 896 GB/s | ~$1,080 실거래 | ~$68 |
| RTX 5080 | 16 GB GDDR7 | 960 GB/s | ~$1,250 실거래 | ~$78 |
| RTX 5090 | 32 GB GDDR7 | 1,792 GB/s | ~$3,699–3,899 실거래 (MSRP $1,999) | ~$116–122 |
| AMD Radeon Pro W7900 | 48 GB GDDR6 ECC | 864 GB/s | ~$3,499 (현 SEP) | ~$73 |
| AMD Radeon AI PRO R9700 | 32 GB GDDR6 | 640 GB/s | $1,299 출시가 | ~$41 |
| RTX 6000 Ada | 48 GB GDDR6 ECC | 960 GB/s | ~$7,800 | ~$163 |
| RTX PRO 6000 Blackwell | 96 GB GDDR7 ECC | 1,792 GB/s | MSRP $8,565, 실거래 ~$8,000–9,300 | ~$89 |
| NVIDIA H100 SXM | 80 GB HBM3 | 3,350 GB/s | ~$35,000 | ~$437 |
| NVIDIA H200 SXM/NVL | 141 GB HBM3e | 4,800 GB/s | ~$30,000–40,000 | ~$210–283 |
| NVIDIA B200 | **192 GB HBM3e** | 8,000 GB/s | ~$30,000–55,000 | ~$156–286 |
| NVIDIA B300 (Blackwell Ultra) | 288 GB HBM3e+ | 8,000 GB/s | ~$40,000–50,000+ | ~$139–174 |

읽는 법:
- **대역폭** → 같은 모델 크기 기준 토큰 생성 속도에 비례
- **용량** → 얼마나 큰 모델을 올릴 수 있는가의 상한
- **$/GB** → 메모리 용량당 비용 (낮을수록 저렴)

Apple Silicon의 강점은 **용량 대비 가격**입니다. H100 대비 GB당 비용이 10–15배 저렴하고, PCIe 병목이 없습니다. 단, 절대 대역폭은 H100의 약 1/6 수준입니다.

RTX 5090은 **$/GB 기준으로 가장 비싸지만** 32 GB에 1,792 GB/s라는 조합이 소형 모델의 빠른 추론에 최적입니다. 문제는 32 GB를 넘는 모델을 올릴 수 없다는 점입니다.

---

## 실무 시사점 — 모델 크기에 따른 하드웨어 선택

| 운용 모델 크기 | 권장 하드웨어 | 이유 |
|---|---|---|
| ~20 GB 이하 (Q4 30B급) | RTX 5090 (~$2,000 MSRP / 실거래 ~$3,700–3,900) | VRAM에 딱 들어감, 대역폭 1,792 GB/s로 빠름 |
| ~60 GB (BF16 30B급) | Mac Studio M4 Max 128GB (~$3,500) — 데스크톱 단일 다이 | VRAM에 올리면서 PCIe 병목 없음 |
| ~60 GB + 노트북 휴대성 | MacBook Pro M5 Max 128GB (~$3,999) | 같은 128GB UMA + 614 GB/s 대역폭. **24/7 sustained 워크로드에는 부적합** (열 설계·배터리) |
| ~140 GB (BF16 70B급) | Mac Studio M3 Ultra 256GB — 데스크톱 듀얼 다이 (M3 Max × 2 UltraFusion) | RTX 단일 카드로는 불가. 512GB는 2026-03 단종 |
| ~400 GB+ (대형 MoE) | H100 80GB × 다중 / B200 192GB / B300 288GB | 소비자 장비로는 수용 불가 |

**하드웨어별 포지셔닝 요약** (Apple 폼팩터 3종 분리):

- **M4 Max (Mac Studio)** — 데스크톱 단일 Max 다이. ~60 GB BF16 모델까지 합리적. 24/7 워크로드 OK.
- **M3 Ultra (Mac Studio)** — 데스크톱 듀얼 Max 다이. ~140 GB BF16 모델까지. 라인업 변경(2026-03): 512GB 옵션 단종, 256GB가 현 최대.
- **M5 Max (MacBook Pro)** — 노트북 단일 Max 다이. 사양은 M4 Max와 비슷하지만 휴대성·내장 배터리 제약. 모델 다운로드·개발 단말로는 적합, 24/7 추론 서버로는 부적합.
- **RTX 5090**: 소형 모델 빠른 추론에 최적. 32 GB 상한이 아킬레스건.
- **NVIDIA H100/H200/B200/B300**: 속도·용량 모두 압도적. 비용도 압도적. 클라우드 API 서비스 제공자 또는 연구소 수준의 선택.

**사내 온프레미스 추론 서버**를 구성한다면, 현실적 선택지는:
- 소규모: Mac Studio M4 Max (128 GB, ~$3,500) × N대 병렬
- 중규모: Mac Studio M3 Ultra (96/256 GB, ~$4,000–10,000) — 256GB가 현 최대 (512GB 단종)
- 대규모: NVIDIA H100/H200 클러스터 (예산 ×10–100)

### 소형 모델의 instruction following 한계

모델 크기는 "VRAM에 올라가는가"만의 문제가 아닙니다. **지시를 실제로 따르는가**도 선택 기준입니다.

작은 모델에서 자주 나타나는 패턴:

- **지시 무시**: 시스템 프롬프트의 제약 조건을 넘어 엉뚱한 출력을 냄
- **컨텍스트 희석**: 대화가 길어질수록 앞의 지시에 대한 attention이 약해져 초기 설정을 잊음
- **형식 민감도**: 프롬프트 형식이 학습 패턴과 조금만 달라도 지시를 일반 텍스트로 처리

원인은 pre-training 용량 부족, RLHF 보상 신호 약함, attention 희석이 겹친 결과입니다.

| 작업 유형 | 최소 권장 크기 |
|---|---|
| 단순 Q&A, 요약 | 7B 이상 |
| 역할·형식·제약이 있는 지시 | 14B 이상 |
| 도구 호출·멀티스텝 에이전트 | 30B 이상 |

> **관련 — 경량 모델의 빠른 발전**: 위 한계가 완화되는 추세도 있습니다. 2026년 현재 경량 모델이 벤치마크 점수 기준으로 과거 대형 모델을 따라잡는 현상이 뚜렷합니다. 자세한 내용은 [397B를 이긴 27B: 경량 모델이 대형 모델을 따라잡는 이유](./small-model-advancement.md) 참조.

---

## 네비게이션

**왔던 길** — 이 부록에 도달하는 전형적인 경로

- "내가 사려는 GPU/Mac에서 이 모델이 돌까?" → 여기
- "KV 캐시 수식 배경으로서 대역폭" → [파라미터 착시: KV 캐시의 진실](./kv-cache.md) → 여기
- "로컬 실행을 시작하기 전 사전 체크" → [로컬 LLM 실행 생태계](./local-llm-platforms.md) → 여기

**갈 길** — 이어 읽으면 자연스러운 3개

1. [파라미터 착시: KV 캐시의 진실](./kv-cache.md) — 대역폭이 왜 천장인지의 수식
2. [로컬 LLM 실행 생태계](./local-llm-platforms.md) — Ollama·LM Studio·MLX로 실제 구동
3. [경량 모델의 빠른 발전](./small-model-advancement.md) — 소형 모델이 이 한계를 얼마나 풀었는지

---

## 연관 문서

- [파라미터 착시: KV 캐시의 진실](./kv-cache.md) — 대역폭이 왜 천장인지의 수식. KV 크기와 초당 읽기 양의 관계
- [트랜스포머 추론의 두 단계](./transformer-inference.md) — Prefill(compute bound)과 Decode(memory bound)이 각각 요구하는 하드웨어 특성
- [경량 모델의 빠른 발전](./small-model-advancement.md) — 대역폭 한계가 완화되는 최신 흐름. 소형 모델이 같은 하드웨어에서 더 많은 동시 요청을 처리
- [리즈닝 모델의 딜레마](./reasoning-models.md) — 소형 모델의 instruction following 한계를 thinking으로 보완하는 구조
- [로컬 LLM 실행 생태계](./local-llm-platforms.md) — Ollama·LM Studio·MLX로 실제 로컬 구동할 때 이 수치를 어떻게 계산에 넣는지
