# 모델 컨텍스트 창 참조 (로컬 전용, gitignore)

memory-bandwidth.md KV 캐시 표의 "-" 마킹 기준.
마지막 조사: 2026-04-25

---

## 컨텍스트 크기 일람

| 모델 | 최대 컨텍스트 | 256K 지원 여부 | 비고 |
|---|---|---|---|
| Claude Opus 4.7 (API) | 1M | ✓ | 2026-04-16 GA |
| Claude Sonnet 4.6 (API) | 1M (beta) | ✓ | 2026-02-17 GA, 1M 옵션 beta |
| Claude Haiku 4.5 (API) | 200K | ✗ | Extended Thinking 미지원 |
| Gemini 3.1 Pro (API) | 1M | ✓ | 2026-04-01 GA, 200K 초과 시 단가 2배 구간 |
| GPT-5.5 (API) | 1M | ✓ | 2026-04-23 출시 |
| GPT-5.4 (API) | 1M | ✓ | 272K 초과 시 input 2배 |
| Llama 3.2 3B | 128K | ✗ | |
| Phi-4-mini 3.8B | 128K | ✗ | |
| Qwen3 4B/8B/14B/32B | 128K | ✗ | |
| Qwen3.5 9B | 262K | ✓ | |
| Qwen3.5 27B | 262K | ✓ | |
| Phi-4 14B | 16K | ✗ | RoPE 확장 시 ~32K 가능 |
| Gemma4:31b | **256K** | ✓ | Hybrid attention (60 layers, SWA + global, 마지막 레이어 global) |
| GLM-4.7-Flash | 128K | ✗ | |
| Nemotron 3 Nano | 1M | ✓ | Mamba-Transformer 하이브리드; Mamba 레이어는 KV 캐시 없음 |
| Nemotron 3 Super | 1M | ✓ | |
| Qwen3.5 35B-A3B | 262K | ✓ | |
| Qwen3.5 122B-A10B | 262K | ✓ | Ollama 표기 "256K" = 262,144 토큰 |
| Qwen3.6 35B-A3B | 1M | ✓ | native 262K, extendable 1M |
| Llama 3.3 70B | 128K | ✗ | |
| Llama 4 Scout | 10M | ✓ | 멀티모달 MoE |
| Qwen3-235B-A22B | 128K | ✗ | |
| Qwen3.5 397B-A17B | 262K | ✓ | |
| Llama 3.1 405B | 128K | ✗ | |
| DeepSeek-V3.1 | 128K | ✗ | MLA 압축 |
| DeepSeek-R1-0528 | 128K | ✗ | MLA 압축 |
| GLM-4.6 | 200K | ✗ (200K < 256K) | 4.5의 128K에서 확장 |
| GLM-5.1 / GLM-5.1-Instruct | 203K | ✗ (203K < 256K) | MLA·DSA 압축 (754B MoE) |
| Kimi K2 (이전) | **128K** | ✗ | 1T MoE, 384 experts (8+1 active), MLA, 61 layers |
| Kimi K2.5 | 262K | ✓ | K2 후속 |
| Kimi K2.6 | **256K** | ✓ | 멀티모달, Modified MIT, HF 가중치 공개 |
| MiniMax M2 | **128K** | ✗ | 230B MoE (10B active) |
| MiniMax M2.5 | 198K | ✗ (198K < 256K) | |
| MiniMax M2.7 | 205K | ✗ | 229B MoE (~10B active) |
| Mistral Small 4 (2603) | **256K** | ✓ | 119B MoE (6B active per token, 8B with embed) |
| DeepSeek V4-Pro / V4-Flash | **1M** | ✓ | Hybrid CSA+HCA, 2026-04-24 출시 |

---

## KV 캐시 크기 계산 규칙

KV 캐시는 컨텍스트 길이에 **선형 비례**합니다:

```
KV(64K) = KV(32K) × 2
KV(128K) = KV(32K) × 4
KV(256K) = KV(32K) × 8
```

Q8 양자화 시 BF16의 약 절반.

MLA(Multi-head Latent Attention) 모델 — DeepSeek, GLM-5.1 — 은 표준 GQA 대비 KV 캐시가 60배 이상 작습니다.

---

## 확인 완료 항목

- Nemotron 3 Nano: 1M (Ollama 공식, NVIDIA 공식)
- Qwen3.5 122B-A10B: 262K (Ollama library, "256K" 표기 = 262,144 토큰)
- MiniMax M2.5: 198K (Ollama library)
