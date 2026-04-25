# 트랜스포머 추론의 두 단계: Prefill과 Decode

> 연관 문서: [파라미터 착시: KV 캐시의 진실](./kv-cache.md),
> [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md),
> [API 비용 최적화](./cost-optimization.md),
> [RAG 아키텍처](./rag-architecture.md),
> [Claude Code 3대 설정 축](./claude-code.md)

---

## 토큰을 하나씩 출력하는 이유

트랜스포머는 자동회귀(autoregressive) 방식으로 동작합니다. 토큰 t+1을 예측하려면 [1, 2, ..., t]까지의 정보가 필요하기 때문에, 출력은 본질적으로 순차적입니다.

그렇다고 매번 모든 토큰을 처음부터 다시 계산하지는 않습니다. 여기서 Prefill과 Decode의 구분이 중요해집니다.

---

## Prefill 단계 — 입력을 한 번에 소화한다

사용자가 프롬프트를 보내면, 그 안의 토큰들은 **한 번에 병렬로** 처리됩니다. 이 단계에서:

- 각 토큰의 Key · Value 벡터를 계산
- 계산 결과를 **KV 캐시**에 저장
- 출력 토큰의 첫 번째를 생성

Prefill은 처리할 토큰이 많을수록 시간이 오래 걸립니다. Attention 연산이 시퀀스 길이의 제곱에 비례하기 때문입니다.

---

## Decode 단계 — 한 번에 토큰 하나씩

첫 번째 출력 토큰이 나온 이후부터는 Decode 단계입니다:

1. 새로 생성된 토큰의 **Query · Key · Value만** 계산
2. 이전 토큰들의 K/V는 캐시에서 그대로 읽어옴
3. 새 토큰의 K/V를 캐시에 추가
4. 다음 토큰 생성 반복

이전 토큰들을 다시 forward pass할 필요가 없습니다. KV 캐시 덕분에 "이미 읽은 것을 다시 읽지 않는다."

```
Prefill :  [T1, T2, T3, T4] --> K/V 캐시 저장, 출력: T5
Decode  :  T5 query + 캐시된 [K1..K4] --> 출력: T6
Decode  :  T6 query + 캐시된 [K1..K5] --> 출력: T7
           ...
```

---

## 세션 재개 시 무슨 일이 일어나는가

세션 진행 중에는 KV 캐시가 서버 메모리에 살아있습니다. 새 메시지는 Decode 단계부터 시작할 수 있어 빠릅니다.

세션이 종료되고 재개되면:

- 이전 대화 전체를 컨텍스트로 다시 전송
- 서버는 이 전체를 **다시 Prefill**
- 컨텍스트가 길수록 첫 응답 지연이 늘어남

**Prompt Caching**을 활성화하면 이 비용을 일부 회수할 수 있습니다. 이전에 전송했던 동일한 prefix에 대해 서버가 KV 결과를 기본 5분(ephemeral)간 보관합니다. `cache_control`의 `ttl="1h"` 옵션을 주면 1시간까지 확장 가능합니다. 캐시 히트가 나면 Prefill을 재실행하지 않고 저장된 KV를 그대로 씁니다.

---

## 컨텍스트 오염과의 연결

컨텍스트 오염 — 세션에 무관한 내용이 뒤섞이는 현상 — 은 단순히 "기억이 뒤죽박죽"되는 문제가 아닙니다. 트랜스포머 구조 수준에서 실질적인 비용이 세 가지로 발생합니다.

**1. Attention 분산**

Decode 단계에서 모델은 새 토큰의 Query를 캐시된 **모든** K/V에 대해 attend합니다. 컨텍스트에 불필요한 정보가 많을수록 Attention Score가 분산되어, 모델이 정작 중요한 정보에 집중하지 못합니다. "집중력이 떨어진다"는 표현이 비유가 아니라 수치적 사실입니다.

**2. Prefill 비용 증가**

세션을 재개할 때 오염된 긴 컨텍스트를 다시 Prefill하면, 첫 응답이 나오기까지 더 오래 걸립니다. Attention이 O(n²)이므로, 컨텍스트 길이가 2배가 되면 Prefill 비용은 4배가 됩니다.

**3. KV 캐시 압박**

컨텍스트가 길수록 KV 캐시 메모리를 더 많이 차지합니다. 서버가 메모리 한도에 가까워지면 캐시를 일찍 evict하고, 다음 요청에서 Prefill을 다시 해야 합니다.

"1세션 = 1작업"과 "의존성 경량화" 원칙의 근거가 여기 있습니다. 컨텍스트를 짧고 선명하게 유지하는 것은 실용적 습관이기 전에, 트랜스포머 추론 구조가 요구하는 사용법입니다.

---

## 요약

| 단계 | 언제 발생 | 비용 특성 |
|---|---|---|
| Prefill | 세션 시작, 컨텍스트 재전송 | O(n²) — 컨텍스트 길이에 민감 |
| Decode | 토큰 생성 매 스텝 | O(n) attention — 상대적으로 가벼움 |
| KV 캐시 히트 | 동일 prefix 재전송 (Prompt Caching) | Prefill 생략 가능 |

컨텍스트가 짧을수록 Prefill이 빠르고, Attention이 집중되고, 캐시가 오래 유지됩니다.

> KV 캐시 수치와 메모리 구조의 상세 내용은 [파라미터 착시: KV 캐시의 진실](./kv-cache.md)에서 다룹니다.

---

## 네비게이션

**왔던 길** — 이 부록에 도달하는 전형적인 경로

- "왜 첫 응답은 느리고 그 뒤는 빠르지?" → 여기
- "KV 캐시가 어디서 만들어지는지부터" → [파라미터 착시: KV 캐시의 진실](./kv-cache.md) → 여기
- "컨텍스트 오염이 왜 비용이야?" → [Claude Code 3대 설정 축 §8](./claude-code.md) → 여기

**갈 길** — 이어 읽으면 자연스러운 3개

1. [파라미터 착시: KV 캐시의 진실](./kv-cache.md) — Prefill에서 만들어지는 KV의 수식
2. [API 비용 최적화](./cost-optimization.md) — Prompt Caching이 Prefill을 스킵시키는 원리
3. [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md) — Prefill·Decode가 요구하는 하드웨어 특성

---

## 연관 문서

- [파라미터 착시: KV 캐시의 진실](./kv-cache.md) — Prefill에서 만들어지는 KV가 어떤 메모리 압박을 만드는지의 수식
- [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md) — Prefill은 compute bound, Decode는 memory bound. 하드웨어 한계와 직결
- [API 비용 최적화](./cost-optimization.md) — Prompt Caching이 Prefill을 서버에서 스킵시키는 메커니즘
- [RAG 아키텍처](./rag-architecture.md) — long-context의 O(n²) prefill이 RAG 선택의 물리적 근거
- [Claude Code 3대 설정 축](./claude-code.md) — 컨텍스트 오염을 prefill·attention 수준에서 관리해야 하는 이유
