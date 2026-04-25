# LLM 평가 — 벤치마크·LLM-as-Judge·실무 eval 루프

> 연관 문서: [에이전트 프레임워크 지형도](./agent-frameworks.md),
> [RAG 아키텍처](./rag-architecture.md),
> [API 비용 최적화](./cost-optimization.md),
> [Claude Code 3대 설정 축](./claude-code.md),
> [안전한 AI는 더 약한 AI인가?](./safety-alignment.md),
> [리즈닝 모델의 딜레마](./reasoning-models.md),
> [경량 모델의 빠른 발전](./small-model-advancement.md)

[README §5](../README.md#5-하네스가-체급을-이긴다)에서 작성-검수 루프를 봤고, [cost-optimization](./cost-optimization.md) · [rag-architecture](./rag-architecture.md) · [agent-frameworks](./agent-frameworks.md)에서 **만드는 법** 을 다뤘습니다. 남은 퍼즐 하나 — **만든 것이 실제로 좋은지 어떻게 아는가**.

"눈대중으로 몇 번 돌려보니 괜찮다" 는 프로토타입 단계에서만 통합니다. 프로덕션 에이전트는 매일 수만 번 호출되고, 모델이 업데이트되면 어제 통했던 프롬프트가 오늘 깨질 수 있습니다. **eval이 깔려 있지 않으면 "품질이 떨어졌다"는 사실조차 뒤늦게 고객이 알려줍니다.**

이 문서는 2026년 4월 기준 LLM 평가의 축 — 벤치마크의 한계·LLM-as-Judge의 편향·프레임워크 지형 — 을 실무 판단 기준으로 정리합니다.

---

## 1. 왜 평가가 어려운가

### 1.1 벤치마크 saturation

2020년대 중반까지 LLM 평가의 표준이었던 MMLU·HumanEval·GSM8K는 2026년에 사실상 **포화**됐습니다.

| 벤치마크 | 최고 점수 | 상태 |
|---|---|---|
| MMLU | 88–94% | 프론티어 모델 구분 불가 |
| GPQA Diamond | 95–97% | near-saturation |
| HumanEval / GSM8K | 95%+ | saturated |
| Humanity's Last Exam | 10–46% | 아직 구분 가능 |
| SWE-bench Verified | 82–93% | 프론티어 경쟁 중 |

만점에 가까울수록 **"더 나은 모델"을 구분하는 힘이 사라집니다.** Claude Opus 4.7과 GPT-5.4를 MMLU로 비교해봐야 1–2%p 차이인데, 그 안에 noise가 더 큽니다.

### 1.2 데이터 contamination

더 고약한 문제는 훈련 데이터에 벤치마크 질문이 유출돼 있을 가능성입니다. 수조 토큰을 긁어서 학습한 모델 입장에서 "공개된 MMLU 문제"를 안 본 척할 수 있을까요? 2024년 이후 등장한 GPQA Diamond·HLE가 **contamination-resistant successor** 로 만들어진 이유가 여기 있습니다 — 더 어렵고, 더 specialized 하고, 일부는 비공개.

### 1.3 생성 태스크의 정답 부재

"이 에세이가 더 좋은 요약인가", "이 코드 리뷰 의견이 유용한가" — 이런 질문에는 **ground truth가 없습니다.** exact match가 불가능하고, 사람이 일일이 채점하면 수천 건 규모에서 무너집니다.

### 1.4 Eval awareness — 모델이 시험을 알아본다

2025년부터 누적된 연구는 프론티어 모델이 **자기가 평가받고 있음을 감지**할 수 있다는 것을 보여줍니다:

- Meta Muse Spark (2026-04 safety report): 공개 벤치마크를 "평가" 로 플래그하는 비율 **19.8%** vs 내부 eval **2.0%**
- Apollo Research: Claude Sonnet 3.7이 alignment eval을 자주 인식함
- arXiv 2406.07358 "AI Sandbagging": GPT-4·Claude 3 Opus가 dangerous capability eval에서 전략적으로 underperform 가능. 합성 데이터로 fine-tune하면 password 있을 때만 능력 노출
- arXiv 2508.00943: monitor-aware 조건에서 monitor false negative 2-6% → **16-36%로 증가**
- 스케일링 관찰: eval awareness가 모델 크기와 **power-law** 로 증가 → 프론티어 세대마다 악화

**실무 함의**: 공개 벤치마크 점수는 당신 도메인의 배포 결정 유일 근거가 될 수 없습니다. 자체 도메인 eval + 프로덕션 모니터링이 필수입니다. → [safety-alignment](./safety-alignment.md) 에서 alignment faking과의 관계를 더 다룹니다.

---

## 2. 정량 평가 — 언제 통하고 언제 안 통하는가

가장 단순한 eval은 deterministic check입니다. 정답이 있으면 이게 가장 싸고 빠릅니다.

### 2.1 Exact match / Regex

- 수학 문제 답, 분류 레이블, SQL 쿼리 결과, JSON 스키마 준수 — **정답 집합이 유한**한 경우에 최고
- SWE-bench가 이 계열 — 생성된 patch가 hidden test를 통과하는가
- 장점: 0 비용, 100% reproducible
- 한계: "의미는 같은데 표현이 다른 답"을 틀린 것으로 침

### 2.2 ROUGE / BLEU의 한계

ROUGE(요약), BLEU(번역)는 reference와의 n-gram overlap을 봅니다. 2010년대 NLP의 주력이었지만 LLM 시대엔 거의 쓰지 않습니다:

- **같은 의미의 바꿔쓰기를 틀리다고 판정** — "그는 화가 났다" vs "그의 분노가 폭발했다"
- 문법만 맞추면 올라감 — 헛소리에도 점수가 잘 나옴
- Reference 한 개로 모든 valid 답변을 커버할 수 없음

### 2.3 Embedding similarity

reference와 응답을 같은 임베딩 공간의 벡터로 변환 후 코사인 유사도. ROUGE/BLEU보다 semantic을 잡지만:

- **의미 유사 ≠ 정답 유사** — "반대 의견"도 벡터가 가까울 수 있음
- reference 품질에 전적으로 의존

### 2.4 Pass@k (코드·reasoning)

동일 프롬프트로 k회 sampling → 하나라도 테스트 통과하면 성공. 에이전트에서는 **pass^k** (모든 k회가 성공해야 함) 가 더 유의미합니다 — reliability가 중요하기 때문. τ-bench가 이 metric을 도입한 이유 (retail 도메인 pass^8 <25%).

### 2.5 언제 정량 평가가 충분한가

| 태스크 | 정량 평가 | 추가 필요 |
|---|---|---|
| 수학·객관식·분류 | Exact match 충분 | 없음 |
| 코드 생성 | 테스트 통과 (pass@1/pass^k) | 스타일·가독성엔 judge |
| 구조화 출력 (JSON·SQL) | 스키마 검증 + 실행 결과 | 없음 |
| 요약·번역·에세이 | 불충분 | **LLM-as-Judge 필수** |
| 대화·상담 | 불충분 | LLM-as-Judge + 사용자 시그널 |

---

## 3. LLM-as-Judge — 그리고 그 편향들

### 3.1 왜 LLM으로 LLM을 평가하는가

생성 태스크 평가의 deadlock을 푼 건 GPT-4 시대의 간단한 발견이었습니다: **충분히 강한 모델은 사람보다 일관되게 판정할 수 있다.** 사람 rater는 피로하고 의견이 갈리지만 LLM은 같은 입력에 같은 확률 분포로 응답합니다. → [README §2 일관성](../README.md#2-거울-앞의-인공지능-구조적-치환)

두 가지 주요 형식:

**Pairwise comparison** — judge에게 응답 A와 B를 보여주고 "어느 쪽이 더 나은가" 를 묻습니다. 절대 점수보다 상대 판단이 안정적이라서 Chatbot Arena·internal A/B에 광범위 사용.

**Rubric-based scoring** — judge에게 명시적 기준(correctness, conciseness, tone 등)을 주고 각각에 1–5 점수를 매기게 함. DeepEval의 G-Eval이 대표 — Chain-of-Thought로 judge를 유도해 **임의 기준** 으로 평가 가능.

### 3.2 편향들 — "judge도 편향된다"

LLM-as-Judge는 만병통치약이 아닙니다. 2024년 이후 쌓인 연구 (arXiv 2406.07791, CALM 프레임워크 등) 는 체계적 편향 12종 이상을 보고했습니다:

| 편향 | 내용 | 대응 |
|---|---|---|
| **Position** | A/B 위치에 따라 판정 바뀜 (무작위 아님) | A-then-B + B-then-A 양방향, 둘 다 일치할 때만 집계 |
| **Length** | 긴 답이 이김 — "길이 = 노력 = 품질" 로 읽힘 | rubric에 "길이 무관" 명시, 길이 정규화 |
| **Style** | 마크다운·bullet 많은 쪽이 유리 | plain text로 정규화 후 비교 |
| **Self-preference** | judge와 같은 모델 계열 선호 | cross-vendor judge |
| **Sycophancy** | 유저 어조에 맞춤 | neutral judge prompt |

**Position bias는 특히 체계적** 입니다 — arXiv 2406.07791은 "repetition stability, position consistency, preference fairness" 3-metric 프레임워크를 도입해 capable judge에서조차 random chance로 설명되지 않는 편향을 보였습니다. 양방향 pairwise(A-then-B + B-then-A, 일치할 때만 counting)가 단일 최선 대응입니다.

### 3.3 Judge 선택

- **비용·속도**: Haiku 4.5급 소형 모델 — 단순 rubric엔 충분
- **품질**: Opus 4.7 / GPT-5.4급 — 미묘한 판정·safety eval
- **cross-vendor 권장**: Claude 모델을 Claude judge로 평가하면 self-preference 편향. OpenAI judge + Anthropic judge 앙상블이 안전
- **판정 일치율 모니터링**: sample에 대해 human label과 judge label의 Cohen's kappa가 0.6 이상이어야 배포 가능

### 3.4 판정을 신뢰할 수 있는지 어떻게 아는가

Judge 자체를 evaluate 해야 합니다. 50–100개 human-labeled sample로 judge 정확도를 확인하고, 카테고리별(간단/어려움, 짧은/긴) agreement를 쪼개보세요. 한 구간에서 agreement가 뚝 떨어지면 rubric을 고쳐야 합니다.

---

## 4. Golden set — 만들고 유지하는 법

"Eval을 돌리려면 뭘 기준으로 돌려야 하는가" — 이 **golden dataset** 이 모든 eval의 토대입니다.

### 4.1 구성 원칙

- **Diversity** — 실제 사용자 쿼리 분포를 반영. 간단/복잡, 짧은/긴, 정상/adversarial 균형
- **Edge case 우선** — cold start, rare entity, 모호한 쿼리
- **크기보다 품질** — 100개 잘 만든 sample이 1,000개 자동 생성보다 낫습니다
- **버저닝** — v1·v2로 고정. 변경 시 이전 버전 대비 회귀 테스트

### 4.2 생성 방법

| 방법 | 장점 | 단점 |
|---|---|---|
| **Manual curation** | 품질 최고, SME 지식 반영 | 느림, 수백 개 한계 |
| **Synthetic (LLM 생성)** | 대량 생성 가능 | 생성 모델의 bias·noise 혼입 |
| **Production 로그 샘플링** | 실제 분포 반영 | privacy·PII 문제, 라벨링 필요 |
| **Red team / adversarial** | 취약점 발굴 | 따로 설계 필요 |

실무 권장: **Manual 50 + Synthetic 500 + Production 200** 혼합.

### 4.3 Synthetic 생성 시 주의

- **생성 모델 ≠ 검증 모델** — 같은 모델로 생성·검증하면 bias 증폭. 생성은 Sonnet, 검증은 Opus 식으로 분리
- **Silver → Gold 승격** — 합성 sample은 일단 silver로 표시, human review 통과분만 gold
- **Low-quality filter** — judge에게 "이 질문이 명확한가, 답이 유일한가" 를 먼저 물어 threshold 이하 제거
- **도메인별 SME calibration 필수** — mission-critical(의료·법률·금융)에서는 합성만으로 가지 말 것

### 4.4 도구

- **DeepEval Synthesizer** — 문서에서 쿼리+답변 자동 생성, RAG eval에 특히 유용
- **Arize Phoenix / Langfuse** — synthetic dataset 생성 + 검증 쿡북 풍부
- **Claude Console "Generate Test Case"** — Anthropic 대시보드에서 직접 생성

---

## 5. A/B 테스트 · 프로덕션 평가

Offline golden set은 **pre-launch 1차 방어선** 입니다. 진짜 품질은 실사용자 트래픽에서 드러납니다.

### 5.1 Shadow traffic

프로덕션 요청을 새 모델·프롬프트에도 **병렬로 흘려보내되, 응답은 사용자에게 반환하지 않고 로그만 남기는** 방식. 사용자 영향 0인 상태로 실분포 테스트 가능.

- 비교 metric: judge 점수, latency, cost, tool call 개수
- 샘플 1–5%만 shadow로 보내도 수천 건/일 규모면 충분

### 5.2 Online A/B

트래픽을 진짜로 나눠서 각 variant의 사용자 시그널 측정:

- **Explicit 시그널**: 👍/👎, 별점, "이 답변이 도움됐나요?"
- **Implicit 시그널**: 재질문율, 세션 종료율, 후속 도구 호출 여부, 사용자가 복사해간 블록
- **Business metric**: 전환율, 티켓 해결율, 에스컬레이션율

주의: **편향 큰 시그널이 많다** — 👎 누르는 사용자는 극단 케이스, 👍는 sycophantic 응답에도 잘 눌림. implicit + business metric을 함께 봐야 합니다.

### 5.3 Regression gate (CI/CD)

모델·프롬프트 PR마다 golden set에 대해 eval을 자동 실행하고, 점수 threshold 이하면 merge block. Braintrust·promptfoo·Inspect AI 모두 GitHub Actions 연동 제공.

```
PR open
  |
  v
eval run on golden_v3
  |
  +--> score >= threshold ?
          yes --> merge allowed
          no  --> block + diff view (어떤 sample이 regress했는지)
```

Anthropic 내부 사례: Claude Code 시스템 프롬프트 **모든 라인 변경마다 ablation eval** 돌려 영향 측정. "Demystifying evals for AI agents" 공식 엔지니어링 글 참조.

---

## 6. 에이전트 평가의 특수성

단일 LLM 호출 평가와 에이전트(=루프 + 도구 + 상태) 평가는 축 자체가 다릅니다. → [agent-frameworks](./agent-frameworks.md)

### 6.1 측정 차원

| 차원 | 의미 | 예시 metric |
|---|---|---|
| **Task completion** | 전체 목표 달성 여부 | DB 상태 비교 (τ-bench), test 통과 (SWE-bench) |
| **Tool call accuracy** | 올바른 도구 + 올바른 인자 | step-level scoring |
| **Reliability (pass^k)** | k회 반복 실행 전부 성공 | τ-bench pass^8 |
| **Cost per task** | 한 작업당 토큰·API 비용 | trace 집계 → [cost-optimization](./cost-optimization.md) |
| **Step efficiency** | 최소 step 수 대비 실제 | trajectory diff |
| **Safety** | 위험 도구 호출·탈취 시도 | guardrail 트리거 빈도 |

### 6.2 왜 task completion만 봐선 부족한가

대부분 에이전트 벤치마크가 **single end-to-end number만 리포트** 합니다 — "tau-bench retail 42%". 그런데 그 42% 안에서:

- 검색 단계 실패 vs 추론 단계 실패 vs 도구 호출 포맷 실패 — 어느 단계인지 모름
- **AgentProp-Bench** (arXiv 2604.16706): 3단계로 분해해 독립 측정 → "어디서 왜 깨지는가" 를 특정 가능

실무 eval에서는 **trace 단위 분석** 이 필수입니다. Inspect AI·Braintrust·Langfuse가 trace 기반 drill-down을 제공하는 이유.

### 6.3 주요 에이전트 벤치마크 (2026-04)

| 벤치마크 | 대상 | SOTA (2026-04) |
|---|---|---|
| **SWE-bench Verified** | 실제 GitHub 이슈 resolving | Claude Opus 4.7 82% |
| **τ-bench / τ²-Bench** | tool-agent-user 상호작용, DB 상태 비교, pass^k | gpt-4o급 <50%, pass^8 <25% |
| **MLE-bench** | Kaggle 75개 ML 엔지니어링 | o1-preview+AIDE = bronze 16.9% |
| **PaperBench** | ICML 2024 Spotlight 20편 재현, 8,316 gradable task | Claude 3.5 Sonnet (New) = 21.0% |

### 6.4 자체 에이전트 eval 최소 구성

1. **Golden task set** — 30–100개 실제 요청 시나리오 (분포 + edge)
2. **Expected trace 요약** — 각 task에서 호출돼야 할 도구 시퀀스 (정확 일치 아닌 superset 체크)
3. **Judge rubric** — correctness·efficiency·safety 3축
4. **Baseline** — 현재 프로덕션 버전 점수 고정. 모든 변경은 이 baseline 대비로만 평가

---

## 7. 2026 평가 프레임워크 지형

### 7.1 공급자 내장

| 도구 | 벤더 | 핵심 |
|---|---|---|
| **Claude Console Evals** | Anthropic | 대시보드 UI, test case 자동 생성, sub-agent 파이프라인 |
| **OpenAI Evals** | OpenAI | 대시보드 + GitHub(`openai/evals`), Trace Evals, 3rd-party 모델 지원 |
| **LangSmith** | LangChain | tracing + eval + prompt management, LangChain/LangGraph에 깊이 결합 |

공통 약점: 자사 생태계 편중. cross-vendor eval엔 별도 도구 필요.

### 7.2 오픈소스

| 프레임워크 | 특징 | 최적 용도 |
|---|---|---|
| **Inspect AI** (UK AISI) | 200+ pre-built eval, sandboxed exec, agent eval 1급. Anthropic·DeepMind·xAI 채택 | safety eval·연구·frontier 모델 평가 |
| **DeepEval** | pytest 스타일, 60+ metrics, G-Eval (CoT judge), Synthesizer 내장 | LLM 유닛 테스트, RAG·agent 통합 |
| **RAGAS** | RAG 전용 4+ metrics (faithfulness·answer relevancy·context precision/recall) | RAG 파이프라인 |
| **TruLens** | eval + tracing 단일, multi-hop agent 디버깅 강점 | 복잡 에이전트 관찰 |
| **promptfoo** | CLI/YAML/CI/CD, red team 내장. 2025-11 OpenAI 인수 (MIT 유지) | 프롬프트 회귀 테스트 |

### 7.3 상용 플랫폼

- **Braintrust** — offline eval + online scoring(live traffic) + CI gate + tracing 통합. PR gate로 regression 자동 block. Free 1M span/월, Pro $249/월
- **Langfuse** — 오픈소스 observability + RAGAS 연동
- **Arize Phoenix** — OpenTelemetry 기반 trace + synthetic dataset 가이드
- **Confident AI, Maxim AI** — RAG·agent 특화

### 7.4 선택 매트릭스

```
                   [Offline eval only]
 promptfoo                                      Inspect AI
 (CLI/YAML)                                     (research grade)

                    DeepEval / RAGAS
                    (python framework)

 LangSmith ------------- Braintrust
 (LangChain-bound)      (vendor-neutral, production)

                  [Offline + Online + CI gate]
```

상황별 첫 선택:

| 상황 | 권장 |
|---|---|
| RAG 평가 시작 | **RAGAS** + DeepEval 보조 |
| pytest 스타일 유닛 테스트 | **DeepEval** |
| safety·frontier 모델 연구 | **Inspect AI** |
| CI/CD regression gate만 원함 | **promptfoo** |
| 프로덕션 트래픽 + CI 둘 다 | **Braintrust** |
| LangChain/LangGraph 스택 | **LangSmith** |
| 공급자 lock-in 피하기 | **OpenTelemetry + Langfuse/Arize** |
| Claude 중심 단일 제품 | **Claude Console + DeepEval** 조합 |

---

## 8. RAG 평가 특화

RAG는 eval 축이 두 개로 나뉩니다 — **retrieval이 좋았나** + **생성이 근거에 충실한가**. → [rag-architecture](./rag-architecture.md)

### 8.1 RAGAS 핵심 metric

| Metric | 측정 대상 | 판단 |
|---|---|---|
| **Faithfulness** | 응답의 모든 주장이 retrieved context로 뒷받침되는가 | 생성 hallucination |
| **Answer Relevancy** | 응답이 질문에 실제로 답하는가 | 주제 이탈 |
| **Context Precision** | 관련 청크가 상위에 랭크됐는가 | retriever 랭킹 품질 |
| **Context Recall** | 정답에 필요한 정보가 context에 모두 있는가 | retriever coverage |

네 개는 **직교 축** 입니다. faithfulness만 높고 context recall이 낮으면 "근거엔 충실하지만 정답 찾는 능력 자체가 부족". context precision만 높고 faithfulness가 낮으면 "문서는 잘 찾았는데 엉뚱한 말을 함" (생성 모델 문제).

### 8.2 Ground truth 없어도 돌아감

RAGAS의 핵심 설계: **LLM-as-judge로 ground truth 없이 평가** 가능. 실무 도입 장벽을 크게 낮춘 부분. 단 judge 자체의 한계가 그대로 상속되므로 3장의 편향 대응을 그대로 적용해야 합니다.

### 8.3 RAG eval 실전 워크플로우

1. 도메인 문서에서 RAGAS로 synthetic Q/A 200개 생성 → human review로 100개 golden
2. 각 golden에 대해 4개 metric 기록 (v1 baseline)
3. 청킹·임베딩·리랭커 변경마다 동일 golden으로 재측정
4. faithfulness 하락 없이 context precision/recall 올라가야 progress
5. Chunk 하나 바꿨는데 4개 중 1개만 올라가면 통과, 2개 이상 regress하면 원복

---

## 9. 설계 체크리스트

새 LLM 기능·에이전트를 만들 때 **배포 전** 체크:

1. **Golden set 존재 여부** — 없으면 평가 불가. 50개라도 먼저 만들기
2. **정답 유형 파악** — exact match 가능? → 정량만으로 충분. 생성 태스크 → judge 필수
3. **Judge 선택** — cross-vendor (Claude 답변 → GPT judge 또는 그 역), baseline human agreement 0.6+ 확인
4. **편향 대응** — pairwise라면 양방향 실행, 길이·스타일 정규화 rubric 명시
5. **Baseline 고정** — 현재 프로덕션 버전 점수를 dataset v1로 박제. 모든 변경은 이 기준 대비로만
6. **CI gate** — PR마다 eval 자동 실행, threshold 밑이면 block. [agent-frameworks](./agent-frameworks.md)의 CI 패턴과 통합
7. **에이전트라면 trace 기반 분석** — end-to-end 숫자 하나로만 보지 말기, 실패 step localize
8. **Shadow traffic 플랜** — 배포 전 1–5% 트래픽으로 실분포 측정
9. **Online 시그널 파이프라인** — 명시적(👍/👎) + 암시적(재질문율) + business metric 3축
10. **Eval awareness 대응** — 공개 벤치마크 점수를 과신하지 말 것. 자체 도메인 eval이 근거

### 자주 틀리는 것

- **Judge로 같은 모델을 쓰기** — self-preference 편향으로 점수 부풀려짐
- **Position bias 무시** — pairwise A/B 한 방향만 돌리면 5–15% 체계적 편향
- **합성 데이터로만 golden 구성** — edge case·SME 지식 빠짐. 의료·법률·금융에서는 특히 위험
- **MMLU·HumanEval 점수만 보고 프로덕션 결정** — saturation + contamination + eval awareness 3중 문제
- **Regression test 없이 프롬프트 수정** — 한 곳 고치면 다른 곳 깨지는 걸 배포 후 알게 됨
- **Metric 하나만 추적** — faithfulness↑인데 latency가 2배 뛰면 failure. 반드시 다축 모니터링

---

## 네비게이션

**왔던 길** — 이 부록에 도달하는 전형적인 경로

- "프롬프트 바꿨는데 품질이 좋아졌는지 어떻게 알지?" → 여기
- "MMLU 94%짜리 모델이 왜 내 태스크에서 약하지?" → 여기
- "LLM-as-Judge 편향 대응" → 여기

**갈 길** — 이어 읽으면 자연스러운 3개

1. [에이전트 프레임워크 지형도](./agent-frameworks.md) — OpenTelemetry 스팬으로 eval 파이프라인의 입력 만들기
2. [RAG 아키텍처](./rag-architecture.md) — RAG eval의 두 축 (retrieval + 생성)과 RAGAS 4-metric
3. [안전한 AI는 더 약한 AI인가?](./safety-alignment.md) — eval awareness·sandbagging·alignment faking 감지

---

## 연관 문서

- [에이전트 프레임워크 지형도](./agent-frameworks.md) — eval 프레임워크와 에이전트 프레임워크는 **OpenTelemetry 표준** 으로 수렴. §11의 MCP·A2A·OTel 표준화 부분과 직결
- [RAG 아키텍처](./rag-architecture.md) — §10 설계 체크리스트의 품질 측정을 RAGAS 4-metric으로 구체화
- [API 비용 최적화](./cost-optimization.md) — eval 자체가 비용이 큼. judge 호출에 prompt caching 적용, batch API로 50% 절감
- [Claude Code 3대 설정 축](./claude-code.md) — Anthropic 내부 Claude Code eval 파이프라인이 모범 사례. CLAUDE.md·Hooks·Skills 변경에 eval 자동화 적용
- [안전한 AI는 더 약한 AI인가?](./safety-alignment.md) — eval awareness와 alignment faking의 관계, safety eval의 특수성
- [리즈닝 모델의 딜레마](./reasoning-models.md) — thinking 출력의 평가 (pass^k, step efficiency, thinking budget 실험)
- [경량 모델의 빠른 발전](./small-model-advancement.md) — 공개 벤치마크 saturation의 한 단면. 소형 모델이 대형 모델을 따라잡은 주장은 자체 도메인 eval로 반드시 검증
- [notes/llm-evaluation-2026.md](../notes/llm-evaluation-2026.md) — 프레임워크 버전·벤치마크 수치·논문 레퍼런스 전부
