# 모델별 GGUF 파일 크기 참조 (HuggingFace 실측)

출처: 주로 [bartowski](https://huggingface.co/bartowski) 및 [unsloth](https://huggingface.co/unsloth) GGUF 레포 기준.
단위: GB. `~` 표시는 추정값.
마지막 검증: 2026-04-25

> **읽는 법**
> - BF16: 원본 정밀도. 최고 품질, 가장 큰 용량.
> - Q8_0: 8비트 정수 양자화. 품질 손실 거의 없음.
> - Q6_K / Q5_K_M: 고품질 압축. 거의 BF16 수준.
> - Q4_K_M: **일반 권장**. 품질·용량 균형.
> - Q3_K_M / Q2_K: 저용량 대안. 소형 모델에서 품질 저하 주의. 400B+ 초대형에서 실용적.

---

## 소형 모델 (≤ 20B)

| 모델 | 파라미터 | 컨텍스트 | BF16 | Q8_0 | Q6_K | Q5_K_M | Q4_K_M | Q3_K_M | Q2_K | 출처 |
|---|---|---|---|---|---|---|---|---|---|---|
| Llama 3.2 3B | 3.2B dense | 128K | 6.4 | 3.4 | 2.8 | 2.5 | 2.0 | 1.7 | 1.3 | bartowski |
| Phi-4-mini | 3.8B dense | 128K | ~7.6 | 4.1 | — | — | 2.5 | — | — | bartowski |
| Qwen3 4B | 4.0B dense | 128K | 8.0 | 4.3 | 3.5 | 3.1 | 2.4 | 2.0 | 1.6 | bartowski |
| Qwen3 8B | 8.2B dense | 128K | 16.4 | 8.7 | 7.1 | 6.3 | 5.0 | 4.1 | 3.2 | bartowski |
| Qwen3.5 9B | 9.0B dense | 262K | ~18 | 9.6 | — | — | 5.9 | — | — | bartowski |
| Phi-4 14B | 14.0B dense | 16K (ext ~32K) | ~29 | 15.6 | — | — | 9.1 | — | — | bartowski |
| Qwen3 14B | 14.8B dense | 128K | 29.6 | 15.7 | 12.8 | 11.3 | 9.0 | 7.3 | 5.7 | bartowski |

> Phi-4-mini / Phi-4 / Qwen3.5 9B: Q6·Q5 미조사 (우선순위 낮음).

---

## 중형 모델 (20B ~ 40B)

| 모델 | 파라미터 | 컨텍스트 | BF16 | Q8_0 | Q6_K | Q5_K_M | Q4_K_M | Q3_K_M | Q2_K | 비고 |
|---|---|---|---|---|---|---|---|---|---|---|
| Gemma4 26B-A4B | 25.2B MoE (3.8B active) | 256K | 50.5 | 26.9 | 22.9 | 19.3 | 17.0 | 13.0 | 11.0 | bartowski; 멀티모달, Apache 2.0 |
| Qwen3.6 27B | 27.0B dense | 262K | 53.8 | 28.7 | 23.2 | 20.5 | 17.5 | 14.4 | 11.6 | bartowski; DeltaNet 하이브리드 |
| Gemma4 31B | 30.7B dense | **256K** | 61.4 | 32.6 | 26.7 | 22.6 | 19.6 | 15.9 | 12.6 | bartowski; 멀티모달 |
| GLM-4.7-Flash | 30B MoE (3B active) | 128K | ~60 | ~32 | — | — | ~18.5 | — | — | bartowski; GGUF는 전체 가중치 포함 |
| Nemotron Cascade 2 | 30B MoE (3B active) | 262K | 63.2 | 33.6 | 33.6 | 26.2 | 24.7 | 19.5 | 18.2 | bartowski; Mamba 하이브리드, 2026-03 출시 |
| Nemotron 3 Nano | 31.6B MoE (3.2B active) | 1M | 63.2 | 33.5 | — | — | 24.7 | — | — | bartowski; Mamba-Transformer 하이브리드 |
| Qwen3 32B | 32.8B dense | 128K | 65.6 | 34.9 | 28.5 | 25.2 | 19.9 | 16.4 | 12.7 | bartowski |
| Qwen3.6 35B-A3B | 35B MoE (3B active) | 1M | 69.4 | 36.9 | 29.3 | 26.5 | 21.4 | 16.6 | 12.3 | unsloth UD; thinking 기본 활성화 |

> **GLM-4.7-Flash 주의**: 공식 소개에서 "3B active"라고 해도 GGUF에는 총 30B 가중치가 모두 포함됨. 과거 잘못된 수치(~10 GB Q4) 유포 있음 — 실제 Q4는 ~18.5 GB.

---

## 대형 모델 (40B ~ 130B)

| 모델 | 파라미터 | 컨텍스트 | BF16 | Q8_0 | Q6_K | Q5_K_M | Q4_K_M | Q3_K_M | Q2_K | 비고 |
|---|---|---|---|---|---|---|---|---|---|---|
| Llama 3.3 70B | 70.6B dense | 128K | 141 | 75 | — | 49.9 | 42.5 | 34.3 | 26.4 | unsloth |
| Qwen3-Coder-Next | ~80B MoE (3B active) | 256K | ~160 | 84.8 | — | — | 48.7 | — | — | bartowski; 코딩 에이전트 특화 |
| Llama 4 Scout | 109B MoE (17B active) | 10M | 216 | 115 | 88.4 | 76.5 | 65.4 | 51.8 | 39.6 | unsloth; 16 experts |
| Mistral Small 4 | 119B MoE (**6B active per token, 8B with embed**) | **256K** | 238 | 126 | 99.4 | 89.2 | 73.8 | 54.4 | 40.2 | unsloth UD; 128 experts, 멀티모달, Apache 2.0, 2026-03 출시 |
| Nemotron 3 Super | 120B MoE (12B active) | 1M | ~240 | 128.5 | — | — | 87.0 | — | — | bartowski |
| Qwen3.5 122B-A10B | 122B MoE (10B active) | 262K | ~244 | 129.9 | — | — | 75.0 | — | — | bartowski |

> **Llama 4 Scout 컨텍스트**: 공식 10M 토큰 지원. 실제 활용 시 KV 캐시 메모리 확인 필수.

---

## 초대형 모델 (130B+)

| 모델 | 파라미터 | 컨텍스트 | BF16 | Q8_0 | Q6_K | Q5_K_M | Q4_K_M | Q3_K_M | Q2_K | 비고 |
|---|---|---|---|---|---|---|---|---|---|---|
| MiniMax M2.7 | 229B MoE (~10B active) | 200K | 457 | 243 | 197 | 163 | 139 | 104 | 80 | bartowski |
| Qwen3-235B-A22B | 235B MoE (22B active) | 1M | ~470 | ~250 | — | — | ~142 | — | — | bartowski |
| Qwen3.5 397B-A17B | 397B MoE (17B active) | 262K | ~794 | ~422 | — | — | ~241 | ~194 | ~166 | bartowski |
| Llama 4 Maverick | 402B MoE (17B active) | 1M | 801 | 426 | 329 | 284 | 243 | 191 | 146 | unsloth; 128 experts, 2026-04 출시 |
| Llama 3.1 405B | 405B dense | 128K | 810 | 430 | — | — | 245 | 198 | — | bartowski |
| DeepSeek-V3.1 | 671B MoE (37B active) | 128K | 1,342 | 713 | — | — | 407 | 328 | 281 | bartowski |
| DeepSeek-R1-0528 | 671B MoE (37B active) | 128K | 1,342 | 713 | — | — | 407 | 328 | 281 | bartowski |
| GLM-5.1 | 744B MoE (40B active) | 203K | ~1,488 | ~791 | — | — | ~451 | ~363 | ~310 | unsloth; MLA 압축 구조 |
| Kimi K2 (이전) | 1T MoE (32B active, MLA) | **128K** | ~2,050 | ~1,090 | — | — | ~580 | ~490 | ~420 | huggingface.co/moonshotai/Kimi-K2-Instruct |
| Kimi K2.5 | ~1T MoE | 262K | 동급 | 동급 | — | — | ~580 | — | — | bartowski |
| Kimi K2.6 | 1T MoE (32B active, 384 experts: 8 routed + 1 shared, MLA) | **256K** | **2,050** | **595** (UD-Q8_K_XL) | — | — | **584** (UD-Q4_K_XL) | — | **340** (UD-Q2_K_XL) | unsloth UD; **Q8과 Q4 차이 10 GB뿐** (lossless 권장), Modified MIT |

---

## 특수 포맷 메모

| 포맷 | 설명 | 지원 하드웨어 | 참고 크기 (Qwen3.6 27B) |
|---|---|---|---|
| FP8 (E4M3) | 8비트 부동소수점. Q8_0과 유사 크기, 더 나은 수치 정밀도 | H100, RTX 4090+ | ~28 GB (Qwen/Qwen3.6-27B-FP8 참조) |
| MXFP8 | Block-scaled FP8. Ollama에서 지원 시작 | RTX 4090+ | ~31 GB (Ollama 실측) |
| NVFP4 | NVIDIA Blackwell 전용 4비트. Q4보다 빠름 | B200, RTX 5090 | ~20 GB (Ollama 실측) |
| MXFP4_MOE | MoE 전용 MXFP4 (unsloth UD 포맷) | RTX 5090+ | ~22 GB (35B-A3B 기준) |

> NVFP4·MXFP8은 Ollama 또는 TensorRT-LLM을 통해 사용. llama.cpp 표준 GGUF와 별도.
