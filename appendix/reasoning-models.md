# 리즈닝 모델의 딜레마

> 연관 문서: [파라미터 착시: KV 캐시의 진실](./kv-cache.md),
> [경량 모델의 빠른 발전](./small-model-advancement.md),
> [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md),
> [API 비용 최적화](./cost-optimization.md),
> [LLM 평가](./llm-evaluation.md),
> [안전한 AI는 더 약한 AI인가?](./safety-alignment.md)

**추론 모델(Reasoning Model)** — OpenAI GPT-5.5 (2026-04-23 출시), Anthropic Claude Opus 4.7 (Adaptive Thinking), Google Gemini 3.1 Pro (Deep Think, 2026-04-01 GA), DeepSeek R1-0528 — 은 일반 모델과 다르게 작동합니다.

강력한 추론 능력을 얻는 대신 **컨텍스트 소비·비용·지연 시간** 세 가지가 함께 증가합니다. 이것이 추론 모델의 딜레마입니다.

---

## 외부 하네스 vs 내재화된 하네스

일반 모델은 피드백 루프가 **외부**에 있습니다:

```
[LLM] --> [tool call] --> [result] --> [LLM] --> ...
```

추론 모델은 그 루프를 **내부로 흡수**했습니다:

```
[LLM <thinking>
  ... hypothesis -> verify -> counterexample -> retry ...
</thinking>] --> [final answer]
```

하네스의 피드백 루프가 뇌 안으로 들어간 구조입니다. 외부에서 보면 한 번의 호출처럼 보이지만, 내부에서는 수십~수백 스텝의 자기 검토가 일어나고 있습니다.

여기서 중요한 질문: **단순히 루프가 내부로 들어간 것뿐인가, 아니면 추론 수준 자체가 달라지는가?**

답은 **학습 방식이 다르기 때문에 수준 자체가 달라진다**입니다.

외부 하네스는 LLM이 루프 구조 설계에 관여하지 않습니다. 사람이 미리 짜둔 tool schema(`read_file`, `run_code` 등)에 LLM이 끼워넣어지는 것입니다. 어떤 중간 단계를 밟을지는 하드코딩되어 있습니다.

추론 모델은 **RLVR(Reinforcement Learning with Verifiable Rewards)** 방식으로 학습됩니다:

1. 모델이 `<thinking>...</thinking>` → 최종 답을 자유롭게 생성
2. 최종 답만 reward signal로 평가 (수학 문제면 정답 여부)
3. 올바른 답으로 이어진 thinking 체인이 강화됨

schema 없이 임의의 중간 추론을 생성하면서, **어떤 intermediate thought가 올바른 답으로 이어지는지를 RL로 스스로 학습**합니다. "백트래킹", "반례 탐색", "단계적 검증" 같은 전략은 사람이 설계한 게 아니라 모델이 reward를 보며 발견한 것입니다.

---

## 왜 내부 Chain-of-Thought가 효과적인가

"생각하는 척" 출력이 왜 실제로 성능을 높일까요?

핵심은 **추론을 위한 연산 예산(compute budget)을 늘린다**는 것입니다. 트랜스포머는 레이어 수가 고정되어 있어, 단일 forward pass에서 쓸 수 있는 연산량에 물리적 상한이 있습니다. `<thinking>` 토큰을 생성하면:

1. 각 토큰 생성이 하나의 forward pass
2. N개의 thinking 토큰 = N번의 forward pass
3. 단계마다 이전 토큰들을 KV 캐시로 참조하며 "생각"을 이어감

결과적으로 **test-time compute를 수직 확장**합니다. 학습 시 고정된 파라미터 수에 관계없이, 추론 단계에서 더 많은 계산을 투입할 수 있습니다. AlphaGo가 MCTS로 탐색 깊이를 늘린 것과 같은 원리입니다.

---

## 숨겨진 비용: thinking은 output 가격으로 과금된다

일반 모델에 `<thinking>` 흉내를 내게 시키면 어떻게 될까요? 속마음이 전부 텍스트로 출력되어 **KV 캐시를 빠르게 잠식합니다.**

추론 모델 자체도 마찬가지입니다. `<thinking>` 토큰은 사용자 눈에는 안 보이지만 **output 토큰 단가**로 과금되고 컨텍스트 윈도우를 빠르게 소진합니다.

| 상황 | 선택 |
|---|---|
| 수학 증명, 논리 퍼즐, 복잡한 아키텍처 설계 | 추론 모델 |
| 반복적 파일 수정, 코드 검색, 정형화된 작업 | 일반 모델 + 강한 하네스 |
| 긴 작업 파이프라인 자동화 | 일반 모델 + Hook + Skill |

추론 모델은 "단발성 깊은 분석"에 씁니다. 루틴 작업을 추론 모델에 맡기는 것은 스캘펠로 나무를 베는 격입니다.

---

## 2026년 공급자별 thinking 조절 파라미터

세 주요 공급자 모두 thinking을 외부에서 조절할 수 있는 파라미터를 제공하지만, **이름·단위·기본값**이 전부 다릅니다.

| 공급자 | 파라미터 | 레벨 | 과금 단가 (thinking) |
|---|---|---|---|
| OpenAI GPT-5.4 | `reasoning.effort` | none / low / medium / high / **xhigh** | output 단가 ($15/MTok) |
| Anthropic Claude Opus 4.7 | `thinking.effort` + `task_budget` | low / medium / high / max (**adaptive, 기본 off**) | output 단가 ($25/MTok) |
| Google Gemini 3.1 Pro | `thinking_level` | LOW / MEDIUM / **HIGH** (Deep Think Mini) | output 단가 ($12/MTok, 200K 초과 시 $18) |

> **Opus 4.7 특이 사항**: `thinking: {type: "enabled", budget_tokens: ...}` 형식의 manual thinking 블록은 Opus 4.7에서 **400 에러**를 반환합니다. Opus 4.7은 adaptive thinking(`thinking: {"effort": "low|medium|high|max"}`)만 받아들입니다. manual 블록이 필요하면 Sonnet 4.6 이하 또는 Haiku 4.5 이하를 써야 합니다.

### Claude 4.7의 변화: budget_tokens 제거

Claude 4.6까지는 `thinking.budget_tokens`로 thinking 토큰 상한을 정수로 지정했지만, 4.7에서 **제거**되고 **adaptive thinking** (effort level + task budget) 방식으로 전환됐습니다.

```python
import anthropic

client = anthropic.Anthropic()

# Claude 4.6 방식 (4.7부터 실패)
# thinking={"type": "enabled", "budget_tokens": 10000}

# Claude 4.7 방식 (adaptive thinking, 기본 비활성 → opt-in)
response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=16000,
    thinking={
        "type": "adaptive",        # 4.7은 adaptive 만 허용 (type:"enabled" 는 400 에러)
        "effort": "high",          # low / medium / high / max
        "display": "summarized"    # 4.7 기본은 "omitted" — summarized 받으려면 명시 필요
    },
    task_budget=30000,             # 전체 agentic loop (thinking + tool + output) 상한
    messages=[{"role": "user", "content": "Prove that sqrt(2) is irrational."}]
)

for block in response.content:
    if block.type == "thinking":
        print("[thinking]", block.thinking[:200], "...")
    else:
        print("[answer]", block.text)
```

`task_budget`은 기존 `budget_tokens`와 개념이 다릅니다 — thinking만이 아니라 **전체 에이전트 루프** (thinking + tool call + tool result + final output) 의 토큰 총량을 상한으로 지정합니다. 개별 thinking 단계를 제어하기보다 **전체 비용 상한을 걸어두는** 방식입니다.

### OpenAI GPT-5.4 reasoning.effort

```python
from openai import OpenAI
client = OpenAI()

response = client.responses.create(
    model="gpt-5.4",
    input="Prove that sqrt(2) is irrational.",
    reasoning={"effort": "high"}   # none / low / medium / high / xhigh
)
```

`xhigh`로 올리면 긴 프롬프트에서 thinking 토큰이 **단발 요청당 20K 토큰**까지 쉽게 쌓입니다. 토큰 단가는 바뀌지 않지만 요청당 비용이 수직 상승합니다.

### Google Gemini 3.1 Pro thinking_level

```python
from google import genai
client = genai.Client()

response = client.models.generate_content(
    model="gemini-3.1-pro",
    contents="Prove that sqrt(2) is irrational.",
    config={"thinking_level": "HIGH"}   # LOW / MEDIUM / HIGH
)
```

HIGH에서 "Deep Think Mini" 가 자동 활성됩니다. 과거의 `thinking_budget` (정수) 도 backward compat로 지원되지만 `thinking_level` 사용이 권장됩니다.

---

## 에이전트 루프 속 Interleaved Thinking

Claude Code 같은 에이전트 환경에서 Adaptive Thinking을 쓰면, thinking 블록이 **도구 호출 사이사이에도** 끼어들 수 있습니다(Interleaved Thinking). **Opus 4.7은 adaptive 모드에서 자동 활성화되며 별도 beta header 가 필요 없습니다** (Sonnet 4.6 manual thinking은 `extended-thinking-beta` header 필요, Opus 4.6 manual은 미지원).

```
[thinking: 파일 구조 파악 계획] --> [read_file] --> [thinking: 결과 해석, 다음 전략] --> [edit_file] --> ...
```

각 도구 호출 직전에 thinking이 들어가므로, "다음 어떤 도구를 써야 하는가"를 한 번 더 추론하게 됩니다. 복잡한 리팩터링이나 다단계 디버깅에서 유효하지만, 도구 호출마다 thinking 비용이 붙으므로 단순 루틴 작업에는 비효율적입니다.

Claude 4.7의 `task_budget`은 이런 누적을 겨냥한 장치입니다 — thinking + tool + output 토큰의 **합계 상한**을 지정해 에이전트 루프 전체가 예산 내에서 끝나도록 강제합니다.

---

## 2026년 실사용 비용 감각

thinking 레벨별 평균 토큰 소비와 요청당 비용 (2026-04 조사):

| 설정 | 평균 thinking 토큰/req | thinking 비용/req | 체감 |
|---|---|---|---|
| 일반 모델 (thinking 없음) | 0 | 0 | 기준 |
| Gemini 3.1 Pro LOW | ~300 | ~$0.004 | 빠른 단순 질의 |
| Gemini 3.1 Pro MEDIUM | ~2,000 | ~$0.024 | 일반 추론 |
| Claude Opus 4.7 high | ~5,000~10,000 | ~$0.13~$0.25 | 중간-복잡 설계 |
| Gemini 3.1 Pro HIGH (Deep Think Mini) | ~8,000 | ~$0.096 | 고난도 추론 |
| GPT-5.4 xhigh (긴 프롬프트) | ~20,000 | ~$0.30 | 올림피아드·증명 |
| Claude Opus 4.7 max + interleaved (에이전트 루프) | 50,000+ (루프 전체) | $1+ | 단일 에이전트 세션 |

**같은 프롬프트**를 thinking 없음 → HIGH로 올리면 **요청당 비용이 3-10배**, 에이전트 루프에서 interleaved로 누적되면 **세션당 비용이 수십 배**가 됩니다. `task_budget` 또는 effort 하향은 단순히 품질 조절이 아니라 **비용 통제 장치**입니다.

### 토크나이저 이슈 (Claude 4.7)

Claude Opus 4.7은 새 토크나이저를 채택해, 같은 텍스트가 4.6 대비 **1x~1.35x 토큰**으로 카운트됩니다. 단가가 바뀌지 않았더라도 **실사용 비용은 최대 35% 상승** 가능합니다. 대규모 배치 작업에서는 실제 청구서가 예상보다 높게 나올 수 있으니 PoC 단계에서 실측하세요.

### 컨텍스트 구간 단가

| 공급자 | 구간 전환점 | 초과 시 변화 |
|---|---|---|
| GPT-5.4 | 272K | input 2배 ($2.50 → $5) |
| Gemini 3.1 Pro | 200K | input $2 → $4, output $12 → $18 |
| Claude Opus 4.7 | 없음 | 고정 ($5 / $25) |

긴 컨텍스트 + 깊은 thinking 조합은 두 축에서 동시에 비용이 튑니다. Claude가 고정 단가라 예산 예측이 쉽고, GPT/Gemini는 단가 변곡점을 넘기지 않도록 프롬프트 길이를 관리해야 합니다.

---

## 실무 판단 기준

추론 모델 투입 여부를 빠르게 판단하는 기준:

```
단 한 번의 호출로 끝나는가?
       |
      YES --> 추론 모델 후보
       |
      NO
       |
       v
반복·루틴 실행이 필요한가? --> YES --> 일반 모델 + 하네스
```

추론 모델은 API 호출 비용이 일반 모델 대비 3~10배 비쌉니다. "정말 어려운 단발성 문제인가"를 먼저 확인하세요. 프롬프트를 잘 짜면 일반 모델로 해결되는 문제가 대부분입니다.

---

## 네비게이션

**왔던 길** — 이 부록에 도달하는 전형적인 경로

- "GPT-5.4/Claude 4.7/Gemini 3.1 Pro thinking이 뭐가 다르지?" → 여기
- "API 비용 튀는 원인 중 thinking 토큰" → [API 비용 최적화](./cost-optimization.md) → 여기
- "소형 모델 + reasoning 조합이 왜 강해졌지?" → [경량 모델의 빠른 발전](./small-model-advancement.md) → 여기

**갈 길** — 이어 읽으면 자연스러운 3개

1. [API 비용 최적화](./cost-optimization.md) — thinking 비용을 줄이는 4개 축 (캐싱·배치·라우팅·컨텍스트 관리)
2. [파라미터 착시: KV 캐시의 진실](./kv-cache.md) — thinking이 KV 캐시를 잠식하는 방식
3. [LLM 평가](./llm-evaluation.md) — thinking 출력을 평가하는 pass^k·step efficiency

---

## 연관 문서

- [파라미터 착시: KV 캐시의 진실](./kv-cache.md) — thinking 토큰이 KV 캐시를 어떻게 잠식하는지, 그리고 2026년 KV 압축 아키텍처들
- [경량 모델의 빠른 발전](./small-model-advancement.md) — Reasoning + 경량 모델로 대형 flagship을 대체하는 시나리오 (Nemotron Cascade 2가 IMO 금메달 수준 달성한 원리)
- [LLM 구동 스펙: 메모리 용량과 대역폭](./memory-bandwidth.md) — 소형 모델의 instruction following 한계를 reasoning으로 보완하는 방식
- [API 비용 최적화](./cost-optimization.md) — thinking 비용을 줄이는 4개 축 (캐싱·배치·라우팅·컨텍스트 관리) 전체 지도
- [LLM 평가](./llm-evaluation.md) — thinking 토큰을 포함한 reasoning 출력 평가의 특수성. pass@k·pass^k, step efficiency, sandbagging 감지
- [안전한 AI는 더 약한 AI인가?](./safety-alignment.md) — reasoning 모델에서 더 선명해지는 alignment faking 패턴과 thinking 내부 감시 전략
