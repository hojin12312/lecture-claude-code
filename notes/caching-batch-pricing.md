# 프롬프트 캐싱 · 배치 API 가격 참조 (2026-04)

`appendix/cost-optimization.md` 작성·갱신용 참조 노트.
각 공급자의 캐싱·배치 할인 구조·최소 토큰·만료 시간만 정리. 모델별 base 단가는 `model-versions.md` 참조.

마지막 조사: 2026-04-25

---

## 프롬프트 캐싱 비교

| 공급자 | 방식 | cache write | cache read | 최소 토큰 | 만료 (기본) |
|---|---|---|---|---|---|
| Anthropic Claude | **수동** (`cache_control`) | 1.25x (5분) / 2x (1시간) | **0.1x (90% 할인)** | 1024 (Sonnet/Opus) · 2048 (Haiku) | 5분 / 1시간 옵션 |
| OpenAI GPT-5.x | **자동** (1024 이상 prefix 매칭) | 불변 (1x) | **0.1x (90% 할인)** | 1024 (128 증가) | 5-10분 비활성, 최대 1시간 |
| Gemini 2.5+ Implicit | **자동** | 불변 | 할인 **보장 없음** (best-effort) | Flash 1024 · Pro 2048 | 무응답 |
| Gemini 2.5+ Explicit | **수동** (CachedContent) | 불변 + 저장 비용 | **0.1x (90% 할인)** | 32,768 | TTL 사용자 지정 |
| Gemini 3 Pro Explicit | 수동 | 불변 + 저장 비용 | **0.25x (75% 할인)** | 32,768 | TTL 사용자 지정 |

### Gemini Explicit 저장 비용

- Pro: **$4.50/MTok/hr** 저장
- Flash: **$1.00/MTok/hr** 저장

### Anthropic 스택

- 캐싱(90% read 할인) + 배치(50% 추가) = **최대 95% 절감**
- 2026-02-05부터 **workspace-level isolation** 적용 (이전 organization-level)

### OpenAI 자동 캐싱 특징

- API 통합 코드 변경 불필요 — 1024 토큰 이상 동일 prefix면 자동 적용
- GPT-5.4: $2.50/MTok → **$0.25/MTok** (cached input)
- GPT-5.4 mini: $0.75 → $0.075
- 128 토큰 단위로 prefix 매칭 범위 확장

---

## 배치 API (비동기 대량 처리)

| 공급자 | 할인율 | 실제 turnaround | 비고 |
|---|---|---|---|
| Anthropic Message Batches | **50% (input·output 둘 다)** | ~24시간 보장 | 캐싱과 스택 가능 → 최대 95% 절감 |
| OpenAI Batch API | **50% (input·output 둘 다)** | 1-6시간 실제, 24시간 보장 | GPT-5/5.4/5.5 전부 지원 |
| Google Gemini Batch | **50% (input·output 둘 다)** | 24시간 목표 | Flex pricing과 동등 |

### Anthropic 배치 적용 예 (2026-04)

| 모델 | 실시간 단가 | 배치 단가 |
|---|---|---|
| Claude Opus 4.7 | $5 / $25 | **$2.50 / $12.50** |
| Claude Sonnet 4.6 | $3 / $15 | **$1.50 / $7.50** |
| Claude Haiku 4.5 | $1 / $5 | **$0.50 / $2.50** |

### OpenAI 배치 적용 예

| 모델 | 실시간 단가 | 배치 단가 |
|---|---|---|
| GPT-5.4 | $2.50 / $15 | **$1.25 / $7.50** |
| GPT-5.5 | $5 / $30 | **$2.50 / $15** (= GPT-5.4 실시간과 동일) |

### 적합 워크로드

- 문서 처리 파이프라인 (사내 보고서 일괄 요약)
- 데이터 엔리치먼트 (CSV 대량 분류·라벨링)
- 야간 분석 잡, 오프라인 평가
- 콘텐츠 생성 큐
- 임베딩 대량 생성

**부적합**: 사용자 대기 중인 실시간 응답, 대화형 에이전트, 실시간 의사결정.

---

## 캐싱 + 배치 결합 효과

캐싱과 배치를 동시에 적용할 경우 총 비용 = `base × 배치 0.5 × cached read 0.1 ≈ base × 0.05`.

실질적으로 **입력 토큰 기준 최대 95% 할인**. RAG·서사적으로 긴 시스템 프롬프트가 반복되는 파이프라인에서 효과가 가장 큼.

단, 캐싱은 입력 토큰에만 적용 (output에는 적용되지 않음). 출력 부분은 배치 50%만 남음.

---

## 주요 출처

- Anthropic Prompt Caching: https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- Anthropic Pricing: https://platform.claude.com/docs/en/about-claude/pricing
- OpenAI Automatic Prompt Caching: https://openai.com/index/api-prompt-caching/
- OpenAI Pricing: https://openai.com/api/pricing/
- Gemini Context Caching: https://ai.google.dev/gemini-api/docs/caching
- Gemini Batch API: https://ai.google.dev/gemini-api/docs/batch-api
- Gemini Pricing: https://ai.google.dev/gemini-api/docs/pricing
- Finout Anthropic 2026 가이드: https://www.finout.io/blog/anthropic-api-pricing
