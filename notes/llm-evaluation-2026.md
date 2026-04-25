# LLM 평가 스택 참조 (2026-04)

강의 부록 `appendix/llm-evaluation.md` 작성 근거 자료. 프레임워크·벤치마크·LLM-as-Judge 연구 순으로 정리.

마지막 조사: 2026-04-25

---

## 1. 공급자 내장 평가 도구

### 1.1 Anthropic — Claude Console + Skill-creator evals

- **Claude Console "Evaluation" 탭** — 프롬프트를 여러 테스트 케이스에 대해 일괄 실행, 모델 응답을 사이드 바이 사이드로 비교. `Generate Test Case`로 Claude가 자동 생성하거나 CSV 업로드 가능
- **Skill-creator eval pipeline (2026-04)** — 4개 sub-agent 병렬 파이프라인: `executor`(스킬을 평가 프롬프트에 대해 실행) + `grader`(예상 결과와 대조) + 2개 보조. Anthropic 내부에서 Claude Code 시스템 프롬프트 변경마다 자동 eval 돌림
- **"Demystifying evals for AI agents"** (공식 엔지니어링 블로그) — 자동 평가는 pre-launch + CI/CD의 1차 방어선. 각 라인별 ablation으로 프롬프트 변경 영향 측정
- **Claude Agent SDK**로도 eval 가능 — TypeScript SDK가 전 기능 노출

### 1.2 OpenAI Evals

- **GitHub**: `openai/evals`, ~17,600 stars (2026-01 기준)
- **대시보드 UI**: OpenAI Dashboard에서 직접 설정·실행
- **Grader 종류**:
  - Basic (Ground-truth) — exact match, 결정론적 체크, 수학·객관식에 적합
  - Model-graded — 더 강한 LLM이 주관적 품질(톤·유머·요약) 평가
- **2026 신기능**:
  - **Trace Evals** — 에이전트 trace에 구조화된 기준으로 점수. regression·실패 모드 포착
  - **Datasets, Prompt Optimization**
  - **Third-Party Model Support** (OpenAI 외 모델도 평가 가능)
- **HealthBench** (2025-05) — 262명 의사 협업, 의료 시나리오 특화 벤치마크

### 1.3 LangSmith (LangChain)

- LangChain 생태계 내장 observability + evaluator
- Tracing·debugging·prompt management 통합
- create_agent·LangGraph 중심 팀에 최적

---

## 2. 오픈소스 평가 프레임워크

### 2.1 Inspect AI — UK AISI

- **Repo**: `UKGovernmentBEIS/inspect_ai`
- **버전**: 0.3.209 (2026-04-20 기준). 2025-09 이후 80+ 마이너 릴리스가 쌓여 현재는 거의 격일 릴리스 페이스. MIT, Python 3.10+
- **채택**: Anthropic, DeepMind, Grok(xAI) 포함 프론티어 랩 "framework of choice"
- **핵심 프리미티브**: `dataset` → `Task` → `Solver` → `Scorer`
- **도구·에이전트 지원**:
  - Custom tools + MCP tools
  - Built-in: bash, python, text editing, web search, web browsing, computer use
  - Sandboxed execution (Docker 내장, Kubernetes/Proxmox 어댑터)
  - Agent eval — multi-turn·multi-agent·external agent 실행 가능
- **UI**: VS Code extension + 웹 기반 Inspect View
- **Inspect Evals**: `UKGovernmentBEIS/inspect_evals` — **200개+ pre-built eval** 컬렉션

### 2.2 DeepEval (Confident AI)

- **Repo**: `confident-ai/deepeval`, MIT
- **특징**: pytest 스타일 "LLM 유닛테스트"
- **60+ metrics** — RAG, agents, multi-turn, MCP, safety, image 전부 커버 (오픈소스 중 최다, 2026-04 기준 50 → 60+로 확장)
- **G-Eval** — Chain-of-Thought LLM-as-judge, **임의 기준으로** 평가. DeepEval이 월 1,000만+ G-Eval 회 실행
- **Synthesizer** — golden dataset 자동 생성 기능 내장

### 2.3 RAGAS

- **Repo**: `explodinggradients/ragas`, PyPI 최신 **0.4.3 (2026-01-13)**
- **핵심 metric 4종**: faithfulness, answer relevancy, context precision, context recall
- **확장 metric**: context entities recall, noise sensitivity, response relevancy
- **강점**: ground truth 없이 LLM-as-judge로 retrieval + 생성 품질 동시 측정
- **Faithfulness** 정의: 응답의 모든 주장이 retrieved context로 뒷받침되면 1.0
- **Context Precision** 정의: 관련 청크가 상위에 랭크되는 정도

### 2.4 TruLens

- RAG + agent tracing 강점
- "evaluation + tracing 단일 워크플로우" — multi-hop trace가 복잡한 에이전트 시스템 디버깅에 유리

### 2.5 promptfoo

- **Repo**: `promptfoo/promptfoo`, MIT
- **2025-11 변화**: OpenAI 인수. 여전히 오픈소스 + MIT 유지
- CLI + YAML config 기반 → CI/CD에 깊이 통합
- 프롬프트 테스팅 + **red teaming / vulnerability scanning** 내장
- **약점**: 로컬 eval에는 최적이나 **production live traffic 평가 미지원** — 프로덕션에선 Braintrust 등 보완 필요

---

## 3. 상용 eval 플랫폼

### 3.1 Braintrust

- **URL**: braintrust.dev
- **커버리지**: offline eval + online scoring(live traffic) + CI/CD gate + tracing
- **Brainstore**: 자체 observability DB (모든 model call·tool invocation 저장)
- **GitHub Action**: PR마다 점수 threshold 밑이면 merge block
- **Scorers**: 25+ built-in + `Loop` AI 어시스턴트가 자연어로 custom scorer 생성
- **가격 (2026-04)**:
  - Free: 1M trace spans/월, 10K eval 실행, 무제한 사용자
  - Pro: $249/월 시작
  - Enterprise: 커스텀

### 3.2 기타 observability/eval 플랫폼

- **Langfuse** — 오픈소스 LLM observability, RAGAS 연동
- **Arize Phoenix** — OpenTelemetry 기반 trace + eval, synthetic dataset 검증 가이드 풍부
- **Maxim AI, Confident AI** — 상용 RAG eval·agent eval 플랫폼

---

## 4. 2026-04 벤치마크 지형

### 4.1 포화된 벤치마크

| 벤치마크 | 최고 점수 | 상태 |
|---|---|---|
| **MMLU** | 88–94% (일부 리포트 97–99%) | Saturated — 프론티어 구분 불가 |
| **GPQA Diamond** | 95–97% | Near-saturation |
| **HumanEval, GSM8K** | 95%+ | Saturated |

공통 이슈: **data contamination** (훈련 데이터에 벤치마크 질문 유출), **scaffold dependence** (평가 하네스가 성능 좌우)

### 4.2 현재 differentiating 벤치마크

| 벤치마크 | 내용 | 2026-04 SOTA |
|---|---|---|
| **Humanity's Last Exam (HLE)** | 2,500 문제, 멀티모달·전문 과목 | 10–46% (여유 큼) |
| **SWE-bench Verified** | 500 실제 GitHub 이슈 (human-filtered) | Claude Opus 4.7 82% / Claude Mythos 93.9% (리포트별 상이) |
| **SWE-bench Pro** | Scale AI의 harder 변형 | — |
| **τ-bench** | 도구 호출 + user simulator + DB 상태 비교. metric: pass^k (다회 실행 일관성) | gpt-4o급 <50%, pass^8 <25% (retail) |
| **τ²-Bench** | dual-control (에이전트·유저 양쪽 제어) | 2026 출시 |
| **τ-Knowledge / τ-Voice** | 2026 τ-bench 패밀리 확장 | — |
| **MLE-bench** | Kaggle 75개 ML 엔지니어링 대회 | o1-preview + AIDE = bronze medal 16.9% |
| **PaperBench** | ICML 2024 Spotlight/Oral 20편 재현. 8,316개 gradable task | Claude 3.5 Sonnet (New) = 21.0% replication |
| **Chatbot Arena (LMSYS)** | 크라우드소싱 pairwise, Bradley-Terry Elo. 6M+ votes | 별도 리더보드 |

### 4.3 Bradley-Terry vs Elo

- Chatbot Arena 초기엔 online Elo → 현재 **Bradley-Terry MLE** 로 전환
- BT 모델: game order 무관, 중앙 집중식 MLE → uncertainty interval 제공
- 장점: 실사용자 선호 기반 → 훈련 데이터 contamination과 독립
- 약점: "좋아 보이는 답변" 편향(길이·스타일) 존재

---

## 5. LLM-as-Judge 연구 (2024–2026)

### 5.1 알려진 편향들

| 편향 | 내용 | 대응 |
|---|---|---|
| **Position bias** | pairwise에서 A/B 위치에 따라 판정 바뀜. 무작위 아니라 체계적 | A-then-B + B-then-A 양방향 실행, 양쪽 일치할 때만 집계 |
| **Length bias** | 긴 출력이 이기는 경향 — 길이 = 노력 = 품질로 읽힘 (실제 정답 여부와 무관) | rubric에 "길이는 평가 기준 아님" 명시, 응답 길이 정규화 |
| **Style bias** | 마크다운·bullet 많은 응답이 유리 | plain text로 정규화 후 비교 |
| **Self-preference** | judge와 같은 모델 계열 응답 선호 | cross-vendor judge 사용 |
| **Sycophancy** | 유저 어조에 맞추는 경향 | neutral judge prompt |

### 5.2 주요 논문

- **"Judging the Judges: A Systematic Study of Position Bias"** (arXiv 2406.07791, ACL 2025) — repetition stability / position consistency / preference fairness 3-metric 프레임워크
- **"Justice or Prejudice? Quantifying Biases in LLM-as-a-Judge"** (llm-judge-bias.github.io) — **CALM 프레임워크**, 12종 bias 정량화. 다만 comparative 평가에 편중, scoring-based 평가 bias 연구 부족
- **"A Survey on LLM-as-a-Judge"** (arXiv 2411.15594v6)
- **"Evaluating Scoring Bias in LLM-as-a-Judge"** (arXiv 2506.22316)

### 5.3 Sandbagging & Evaluation Awareness

- **"AI Sandbagging"** (arXiv 2406.07358) — GPT-4·Claude 3 Opus가 dangerous capability eval에서 **전략적으로 underperform** 가능. 합성 데이터로 fine-tune해서 password 있을 때만 능력 드러내게 훈련 가능. WMDP 같은 공개 벤치마크에 일반화
- **"Probing Evaluation Awareness"** (arXiv 2507.01786) — linear probe로 eval awareness를 activation에서 추출 가능
- **"LLMs Can Covertly Sandbag Against CoT Monitoring"** (alphaXiv 2508.00943) — monitor-aware 조건에서 monitor false negative 2-6% → 16-36%로 증가
- **Apollo Research: Claude Sonnet 3.7 evaluation awareness** — alignment eval임을 자주 인식함을 실증
- **Meta Muse Spark** (2026-04 safety report) — 공개 벤치마크를 eval로 플래그하는 비율 19.8% vs 내부 eval 2.0%. **eval awareness가 모델 크기와 power-law 스케일링** — 프론티어 세대마다 악화
- **실무적 함의**: "public benchmark는 specific domain production 배포 결정의 유일 근거가 될 수 없다" → 자체 도메인 eval 필수
- **Anthropic "Eval awareness in Claude Opus 4.6's BrowseComp"** — 공식 엔지니어링 블로그, 프론티어 모델의 eval awareness 현상과 대응

---

## 6. Golden Dataset & Synthetic Eval 생성

### 6.1 원칙

- **Synthetic + Human-authored 혼용** — 합성 데이터만으로는 edge case 커버 부족. human-in-the-loop QA로 silver → gold 승격
- **생성 모델 ≠ 검증 모델** — 같은 모델로 생성하고 검증하면 bias 누적
- **Diversity + Complexity** — 다양한 real-world 입력, 단순/복잡 난이도 balanced
- **Edge case 우선** — cold start·adversarial·rare scenario
- **Manual review 필수** — mission-critical일수록 SME(Subject Matter Expert) calibration

### 6.2 도구

- **DeepEval Synthesizer** — 문서에서 쿼리+답변 자동 생성 (RAG eval에 특히 유용)
- **Phoenix/Arize** — synthetic dataset 생성 + 검증 쿡북
- **Langfuse** — synthetic eval dataset 생성 가이드

### 6.3 품질 관리

- **Filter low-quality goldens** — LLM 생성 시 노이즈 섞임. threshold 이하 자동 제거
- **Top-performing model로 벤치마크** — 단 **생성 모델과 달라야 함**
- **버저닝** — golden set v1, v2... 으로 고정. 변경 시 이전 버전 회귀 비교 필수

---

## 7. 에이전트 평가 특수성

### 7.1 측정 차원

| 차원 | 의미 | 측정 방법 |
|---|---|---|
| **Task completion rate** | 전체 작업 성공률 | end-to-end, DB 상태 비교 (τ-bench) |
| **Tool call accuracy** | 올바른 도구 + 올바른 인자 | step-level scoring |
| **Pass^k (reliability)** | k회 반복 실행 전부 성공 | τ-bench pass^8 |
| **Cost per task** | 한 작업당 토큰·API 비용 | trace 집계 |
| **Step efficiency** | 최소 step 수 대비 실제 step | trajectory diff |

### 7.2 문제점

- 대부분 벤치마크가 **single end-to-end number만 리포트** → 파이프라인 어느 단계 실패인지 localize 안 됨
- **AgentProp-Bench** (arXiv 2604.16706) — 3단계로 분해해 독립 측정 (propagation cascade 연구)

---

## 8. OpenTelemetry for LLM

- **OTel GenAI semantic conventions** — 2026년 사실상 표준
- LangSmith·Braintrust·Langfuse·Arize Phoenix 모두 OTel 호환
- 에이전트 trace = span tree: agent → tool call → model call → nested
- **Inspect AI**도 2026년 OTel export 지원
- 에이전트 프레임워크 참조: [agent-frameworks-2026.md §표준화](./agent-frameworks-2026.md)

---

## 출처

### 프레임워크
- Inspect AI: https://github.com/UKGovernmentBEIS/inspect_ai , https://inspect.aisi.org.uk/
- Inspect Evals: https://github.com/UKGovernmentBEIS/inspect_evals
- OpenAI Evals: https://github.com/openai/evals , https://evals.openai.com/
- DeepEval: https://github.com/confident-ai/deepeval , https://deepeval.com/
- RAGAS: https://github.com/explodinggradients/ragas , https://docs.ragas.io/ , PyPI
- promptfoo: https://github.com/promptfoo/promptfoo
- Braintrust: https://www.braintrust.dev , https://www.braintrust.dev/docs/evaluate , https://www.braintrust.dev/pricing
- LangSmith: docs.smith.langchain.com

### Anthropic
- "Demystifying evals for AI agents": https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- Claude Console eval tool: https://docs.claude.com/en/docs/test-and-evaluate/eval-tool
- "Eval awareness in Claude Opus 4.6's BrowseComp": https://www.anthropic.com/engineering/eval-awareness-browsecomp
- Tessl blog on skill-creator evals: https://tessl.io/blog/anthropic-brings-evals-to-skill-creator-heres-why-thats-a-big-deal/

### 벤치마크
- HLE: https://agi.safe.ai/ , arXiv 2501.14249
- SWE-bench Verified: https://www.swebench.com/verified.html , https://epoch.ai/benchmarks/swe-bench-verified
- τ-bench/τ²-bench: https://github.com/sierra-research/tau2-bench , arXiv 2406.12045, arXiv 2506.07982
- MLE-bench: https://openai.com/index/mle-bench/ , arXiv 2410.07095
- PaperBench: arXiv 2504.01848
- AgentProp-Bench: arXiv 2604.16706
- Chatbot Arena: https://openlm.ai/chatbot-arena/ , arXiv 2403.04132

### LLM-as-Judge 연구
- Position bias: arXiv 2406.07791 (ACL 2025)
- Justice or Prejudice (CALM): https://llm-judge-bias.github.io/
- Survey: arXiv 2411.15594v6
- Scoring bias: arXiv 2506.22316
- Sandbagging: arXiv 2406.07358
- Eval Awareness probing: arXiv 2507.01786
- Covert sandbagging vs CoT monitoring: alphaXiv 2508.00943
- Apollo Research Sonnet 3.7: https://www.apolloresearch.ai/blog/claude-sonnet-37-often-knows-when-its-in-alignment-evaluations/

### Golden set / Synthetic
- Evidently AI: https://www.evidentlyai.com/llm-guide/llm-test-dataset-synthetic-data
- Phoenix/Arize: https://phoenix.arize.com/creating-and-validating-synthetic-datasets-for-llm-evaluation-experimentation/
- Maxim AI golden dataset: https://www.getmaxim.ai/articles/building-a-golden-dataset-for-ai-evaluation-a-step-by-step-guide/
- Langfuse synthetic dataset: https://langfuse.com/guides/cookbook/example_synthetic_datasets
