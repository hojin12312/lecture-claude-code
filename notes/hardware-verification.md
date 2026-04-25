# 하드웨어 스펙 검증 메모 (로컬 전용, gitignore)

memory-bandwidth.md 업데이트 전 웹 서치로 사전 검증하는 문서.
마지막 검증: 2026-04-25

---

## 검증 결과

### Apple Silicon

| 항목 | 내용 | 출처 |
|---|---|---|
| M4 Ultra | **개발 취소** — 공식 발표 없이 조용히 취소됨 | 검색 결과 |
| M5 | 2025-10 출시 (MacBook Pro), 32 GB, **153 GB/s** | apple.com/macbook-pro/specs |
| M5 Pro | 2026-03 출시 (MacBook Pro), 64 GB, **307 GB/s** | apple.com/macbook-pro/specs |
| M5 Max 32-core GPU | 2026-03 출시 (MacBook Pro), 128 GB, **460 GB/s** (노트북 전용, Mac Studio 미탑재) | apple.com/macbook-pro/specs |
| M5 Max 40-core GPU | 2026-03 출시 (MacBook Pro), 128 GB, **614 GB/s** (노트북 전용) | apple.com/macbook-pro/specs |
| M5 Ultra | **미출시** — 2026 H1~10월 슬립 가능 (Bloomberg/Gurman supply chain 보도). M5 Mac Studio는 base 가격 +$200 인상 예상 (512GB SSD → 1TB 베이스화). 대역폭 예상치 ~1,100 GB/s (M3 Ultra 819 GB/s 대비 +35%) | Macworld, Bloomberg |
| **Mac Studio M3 Ultra 512GB 옵션** | **2026-03 단종** — 현재 최대 256 GB | tomshardware.com/tech-industry/apple-pulls-512-mac-studio-upgrade-option |
| **Mac Studio M3 Ultra 256GB 업그레이드** | $1,600 → **$2,000** (+$400, 2026-03 동시 인상) | 동일 |
| 128GB / 256GB 구성 배송 | 4-5개월 지연 (글로벌 메모리 칩 부족, AI 수요 압박) | macworld.com 기사 |
| **M3 Ultra base 가격** | **$3,999** (96GB, 1TB SSD, 28-core CPU/60-core GPU) | macrumors.com/2025/03/05/maxed-out-m3-ultra-mac-studio-14099 |
| M3 Ultra maxed (단종 전) | **$14,099** (32-core CPU + 80-core GPU + 512GB + 16TB SSD) | 동일 |
| Mac Studio 현 라인업 | M4 Max + M3 Ultra 혼재 판매 중. 128GB/256GB 구성은 2026-04 기준 공급 부족 | Apple Store |
| Mac Studio M4 Max 128GB 가격 | ~$3,499 추정 (1차 출처 차단, 보류) | Apple Store |

### NVIDIA 소비자 GPU 실거래가 (2026-04)

| 모델 | MSRP | 실거래가 | 비고 |
|---|---|---|---|
| RTX 5090 | $1,999 | **~$3,999** | 공급난 지속, GDDR7 병목 |
| RTX 5080 | $999 | **~$1,249** | 피크($1,900) 대비 하락 중 |
| RTX 5070 Ti | $749 | **~$999** | 피크($1,220) 대비 하락 추세 |

### NVIDIA 데이터센터 GPU

| 모델 | VRAM | 대역폭 | 단가 | 비고 |
|---|---|---|---|---|
| H200 SXM / NVL | 141 GB HBM3e | 4,800 GB/s | **~$30,000–40,000** | SXM·NVL 메모리 동일 |
| B200 | **192 GB HBM3e** | 8,000 GB/s | **~$30,000–55,000** | 2 chiplet × 4 HBM3e × 24 GB/stack = 192 GB |
| **B300 (Blackwell Ultra)** | **288 GB HBM3e+** | **8,000 GB/s** | **~$40,000–50,000+** | 2026-04 "Shipping Now", DGX B300 8-GPU = 2.1 TB |

- 출처: nvidia.com/en-us/data-center/h200, nvidia.com/en-us/data-center/dgx-b300
- B200 메모리: 과거 180 GB로 알려졌으나 실제 192 GB. snel.com / technical.city / slyd.com 일치
- B300 vs B200: 대역폭 동일, 메모리 용량 288 GB로 대폭 증가 (추론 inference 특화)

---

## memory-bandwidth.md 반영 완료 항목

- [x] Apple Silicon 표: M4 Ultra 취소 표시 + M5 시리즈 추가
- [x] NVIDIA 데이터센터 표: B300 추가, H200/B200 가격 수정
- [x] RTX 50 시리즈 실거래가 업데이트
- [x] 플랫폼 비교 표: Mac Studio 가격, M5 Max 행 추가

---

---

## 2026-04-21 추가 조사 결과

### RTX 50 시리즈 실거래가 업데이트 (2026-04, bestvaluegpu/videocardz 1차 출처)

| 모델 | MSRP | 실거래가 (2026-04) | 변화 |
|---|---|---|---|
| RTX 5060 Ti (16 GB) | $429 | **~$379–$599** | bestvaluegpu/videocardz 직접 |
| RTX 5070 | $549 | ~$629–$699 | 이전 ~$650–800에서 하락 |
| RTX 5070 Ti | $749 | ~$999–$1,169 | 이전 ~$900–$1,000에서 상승 |
| RTX 5080 | $999 | ~$1,200–$1,300 | 유지 |
| RTX 5090 | $1,999 | **~$3,699–$3,899** (Newegg $3,699 / Amazon $3,899 신품) | 종전 추정 ~$3,500–5,000 → 1차 출처에서는 $3,699–3,899 범위 |

- RTX 5060 Ti: 16 GB GDDR7, 448 GB/s, 8 GB 변종도 있으나 VRAM 부족으로 비추천
- RTX 5090 GDDR7 공급 부족 지속 → 가격 고공행진

### NVIDIA 전문가용 GPU (RTX PRO / 구 Quadro)

| 모델 | VRAM | 대역폭 | 가격 | 비고 |
|---|---|---|---|---|
| RTX 4500 Ada | 24 GB GDDR6 ECC | 432 GB/s | ~$2,000–2,500 | |
| RTX 6000 Ada | 48 GB GDDR6 ECC | 960 GB/s | ~$7,800–10,000 | 현재도 유통 |
| **RTX PRO 6000 Blackwell** | **96 GB GDDR7 ECC** | **1,792 GB/s** | **MSRP $8,565** (2025-03 출시), 2026-04 ~$8,000–9,300 | 24,064 CUDA, NVFP4 지원, **NVLink 미지원** |

- "Quadro" → 2021년 NVIDIA RTX Professional로 리브랜딩 → 2025년 "RTX PRO"
- RTX PRO 6000 Blackwell: 96 GB = RTX 5090(32 GB)의 3배. Workstation 단일 GPU 최대 VRAM
- 출처: nvidia.com 워크스테이션 페이지, thundercompute.com, videocardz.com

### AMD 전문가용 GPU (Radeon Pro)

| 모델 | VRAM | 대역폭 | 가격 | 아키텍처 |
|---|---|---|---|---|
| Radeon Pro W7800 | 32 GB GDDR6 ECC | 576 GB/s | **MSRP $2,499** (2023-04-13 출시) | RDNA 3 |
| Radeon Pro W7900 | 48 GB GDDR6 ECC | 864 GB/s | 출시 MSRP $3,999 → **현 SEP $3,499** (384-bit 18 Gbps) | RDNA 3 |
| **Radeon AI PRO R9700** | 32 GB GDDR6 | 640 GB/s | **출시가 $1,299** (Microcenter/XFX/Amazon 일치) | RDNA 4 (2025) |

- W7900: 48 GB ECC, RTX 6000 Ada 동급 VRAM 절반 가격 → LLM 온프레미스 가성비
- W9700: 별도 제품 없음 — AMD가 기존 Radeon Pro W 브랜드를 Radeon AI PRO R 브랜드로 교체. R9700이 사실상 W9700 포지션을 대체함 (폐기)
- 출처: phoronix.com 리뷰 (amd-radeon-ai-pro-r9700), AMD 공식, tomshardware, cgchannel.com

### vLLM ROCm 진척 (2026-04 기준)

- **2025-12-29 dedicated CI 출범** → 2026-04 기준 **AMD CI 테스트 93% 통과** (2025-11 37% 대비 급상승)
- MI300X / MI325X / MI350X / MI355X 모두 vLLM Docker 공식 지원
- **AITER 백엔드**: `ROCM_AITER_FA` 가 legacy `ROCM_ATTN` 대비 **TPOT 2.8–4.6× 향상**
- 출처: rocm.docs.amd.com, blog.vllm.ai/2026/02/27/rocm-attention-backend.html
- 이전 기록 "vLLM ROCm 지원 미성숙" 은 더 이상 유효하지 않음 (2025-11 → 2026-04 사이에 큰 진전)

### 기타 플랫폼 (AMD ROCm / Intel / Qualcomm) — 2026-04 정확화

#### Intel OpenVINO 릴리즈 (github.com/openvinotoolkit/openvino/releases)

| 버전 | 출시일 | 핵심 기능 |
|---|---|---|
| **2026.1.0** | **2026-04-07** | Qwen3-VL / GPT-OSS-120B (CPU), **OpenVINO backend for llama.cpp** (Intel CPU/GPU/NPU), TaylorSeer Lite caching (diffusion-transformer 가속), Dynamic LoRA (Qwen3-VL 등 vision-language), Intel Core Series 3 + Arc Pro B70 신규 지원, `openvino.runtime` deprecated namespace 제거 |
| **2026.0.0** | **2026-02-23** | GPT-OSS-20B / MiniCPM-V-4.5-8B / MiniCPM-o-2.6, NPU 신규 모델, **NPU speculative decoding (EAGLE-3)**, INT4 data-aware weight compression for 3D MatMuls (MoE LLM 효율 배포) |

#### Qualcomm Snapdragon X2 Elite/Extreme (qualcomm.com/news/releases/2025/09/)

| 항목 | 정확 값 |
|---|---|
| 발표 / 노트북 출시 | **2025-09-25 Snapdragon Summit / 2026 H1** |
| 공정 / CPU | **3nm / 18-core Oryon Prime** (12 Prime + 6 Performance, 5.0 GHz boost) |
| 캐시 / 메모리 | 53 MB / 192-bit / 228 GB/s |
| **Hexagon NPU** | **80 TOPS INT8** (X1 Elite 45 TOPS의 1.78배), **FP8 / BF16 신규 지원** |
| LLM 추론 능력 | **30B+ 파라미터** 추론 가능 (QNN SDK + ONNX Runtime QNN EP) |
| Copilot+ PC 최저 / 로컬 LLM 권장 | 40 TOPS / 45+ TOPS + 32GB RAM |

#### 기타

- **AMD ROCm**: llama.cpp + vLLM 지원 진척, RX 7900 XTX 24 GB GDDR6 960 GB/s. (vLLM ROCm 진척 별도 섹션 참조)
- 출처: github.com/openvinotoolkit/openvino/releases, qualcomm.com/news/releases/2025/09/

### Ollama 최신 릴리즈 (1차 출처 직접 검증 후 정정)

- **v0.21.2** (2026-04-23): OpenClaw 온보딩 안정성, 모델 추천 순서 정렬, 웹 검색 플러그인 번들링
- **v0.21.1** (2026-04-22): Kimi CLI 통합 (`ollama launch kimi`), MLX logprobs/top-P/top-K, GLM4 MoE Lite 라우터 최적화
- **v0.21.0** (2026-04-16): **Gemma 4 MLX (Apple Silicon), Hermes Agent, GitHub Copilot CLI 통합** (`ollama launch`), MLX 백엔드 mixed-precision quantization, OpenClaw `--yes` 버그픽스
- **v0.20.5** (2026-04-09): **OpenClaw 채널 설정 (WhatsApp/Telegram/Discord) — Ollama-OpenClaw 통합 시작점**, Gemma 4 Flash Attention
- **v0.7.0** (2025-05-13): 멀티모달 신엔진 시작 (Llama 4 / Gemma 3 / Qwen 2.5 VL / Mistral Small 3.1)
- **v0.6.2** (2025-03-18): Gemma 3 멀티 이미지, AMD Strix Halo GPU 지원, /save 버그픽스
- 출처: github.com/ollama/ollama/releases

> **이전 잘못된 기록**: "v0.21.0 (2026-04-17): Kimi K2.5/GLM-5/MiniMax 지원, OpenClaw 통합" / "v0.6.2 (2026-03): Llama 4, Flash Attention 2.7, M4 Metal 3" — 1차 출처 대조 시 두 항목 모두 시점·내용 오류 (피드백 §2.1 참조).

### NVIDIA TensorRT-LLM 양자화 (공식 문서 직접 정리)

| 포맷 | 비트 | 대상 GPU | 비고 |
|---|---|---|---|
| **NVFP4** | 4-bit FP | Blackwell (SM120/SM100/SM103) 전용 | ModelOpt 오프라인 양자화. FP16 대비 메모리 3.5×, FP8 대비 1.8× 절감 |
| **FP8 Per Tensor** | 8-bit FP | Blackwell + Hopper + **Ada Lovelace (RTX 4090 포함)** | 가장 광범위한 FP8 |
| **FP8 Block Scaling** | 8-bit FP | Blackwell + Hopper | SM100/103 MXFP8 (E4M3) 레시피 |
| **FP8 Rowwise** | 8-bit FP | Hopper 전용 | |
| **FP8 KV Cache** | 8-bit | 모든 지원 아키텍처 | 가중치와 별개로 활성화 |
| **W4A8 / W4A16 AWQ** | 4-bit weight | Blackwell + Hopper + Ada + Ampere | |
| **INT4 GPTQ** | 4-bit weight | Blackwell + Hopper + Ada + Ampere | |

- 출처: nvidia.github.io/TensorRT-LLM/features/quantization.html
- **이전 오류**: standalone INT8 항목은 공식 문서에 미언급. FP8은 H100/RTX 4090 한정이 아니라 Ada Lovelace 전체 지원.

---

## 다음 검증 필요 항목

- M5 Ultra 출시 시 스펙 업데이트 (Mac Studio 2026)
- RTX 50 시리즈 가격 정상화 여부 (수급 안정화 시)
- B300 개별 GPU 공식 단가 확정 시
- AMD Radeon AI PRO R9700 대역폭: 640 GB/s 확인 완료 (AMD 공식 데이터시트)
- Radeon Pro W9700: 폐기 — AMD가 W 브랜드 → AI PRO R 브랜드로 교체, R9700이 해당 포지션 대체 (2026-04-21 확인)

