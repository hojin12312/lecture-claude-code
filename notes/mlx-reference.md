# MLX 참조 메모 (로컬 전용, gitignore)

마지막 조사: 2026-04-25
출처: github.com/ml-explore/mlx, github.com/ml-explore/mlx-lm

## 최신 버전 (2026-04-22)

- **MLX 코어**: 0.31.2
- **mlx-lm**: 0.31.3
- M5 Neural Accelerators 최적 활용 시 **macOS 26.2+** 필요

## Ollama MLX 통합 (2026-03-31)

Ollama가 Apple Silicon 백엔드로 MLX 채택. 프롬프트 처리 1.6×, 응답 생성 ~2× 가속. 즉, **MLX를 직접 쓰지 않더라도 Ollama 사용자가 자동 수혜**. 독립 MLX 설치는 LoRA 파인튜닝이나 커스텀 모델 실행에서 여전히 필요.

---

## MLX란

Apple이 만든 오픈소스 머신러닝 프레임워크. Apple Silicon 전용.
PyTorch와 유사한 API를 갖지만 내부 동작은 Apple Silicon UMA에 최적화됨.

---

## 핵심 특징

| 특징 | 내용 |
|---|---|
| 통합 메모리 네이티브 | CPU↔GPU 데이터 전송 없음 — 모든 코어가 동일 메모리 풀 직접 접근 |
| Lazy evaluation | 필요할 때만 연산 실행 — 불필요한 메모리 할당 최소화 |
| 동적 그래프 | 입력 shape 변경 시 재컴파일 불필요 |
| 자동 미분 + 벡터화 | 추론뿐 아니라 학습·파인튜닝까지 지원 |
| 분산 처리 | `mx.distributed`로 다중 디바이스 확장 가능 |

---

## mlx-lm (LLM 전용 패키지)

```bash
pip install mlx-lm

# 모델 실행
mlx_lm.generate --model mlx-community/Qwen3-8B-4bit --prompt "Hello"

# 양자화
mlx_lm.convert --hf-path Qwen/Qwen3-8B --mlx-path ./qwen3-8b-4bit -q

# LoRA 파인튜닝
mlx_lm.lora --model ./qwen3-8b-4bit --train --data ./my_data
```

| 기능 | 내용 |
|---|---|
| 모델 지원 | HuggingFace Hub 수천 개 모델 직접 실행 |
| 양자화 | 4비트 포함 다양한 양자화 지원 |
| LoRA | 양자화된 모델도 파인튜닝 가능 |
| 프롬프트 캐싱 | 긴 컨텍스트 재사용 시 성능 최적화 |
| Rotary Cache | 고정 KV 캐시 크기로 메모리 사용량 제한 |

---

## llama.cpp vs MLX (Apple Silicon 기준)

| 항목 | llama.cpp / Ollama | MLX |
|---|---|---|
| 플랫폼 | 크로스 플랫폼 (CPU, CUDA, Metal) | Apple Silicon 전용 |
| UMA 활용 | Metal backend로 부분 활용 | 설계 단계부터 UMA 네이티브 |
| 추론 속도 | Apple Silicon에서 양호 | Apple Silicon에서 더 빠른 경향 |
| 파인튜닝 | 지원 제한적 | LoRA 파인튜닝 네이티브 지원 |
| 생태계 | 광범위한 커뮤니티, 모델 포맷(GGUF) | HuggingFace 모델 직접 지원 |
| 모델 포맷 | GGUF | HF safetensors / MLX 변환 포맷 |

---

## memory-bandwidth.md 반영 위치

- Apple Silicon 섹션 끝 또는 "실무 시사점" 앞에 추가
- 핵심 메시지: Apple Silicon에서 LLM 실행 시 MLX가 llama.cpp보다 UMA를 더 잘 활용

