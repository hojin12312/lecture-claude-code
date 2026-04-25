# RAG 아키텍처 — 검색·청킹·리랭킹·장문컨텍스트와의 경계

> 연관 문서: [API 비용 최적화](./cost-optimization.md),
> [파라미터 착시: KV 캐시의 진실](./kv-cache.md),
> [Claude Code 3대 설정 축](./claude-code.md),
> [에이전트 프레임워크 지형도](./agent-frameworks.md),
> [LLM 평가](./llm-evaluation.md),
> [경량 모델의 빠른 발전](./small-model-advancement.md),
> [리즈닝 모델의 딜레마](./reasoning-models.md),
> [트랜스포머 추론의 두 단계](./transformer-inference.md)

LLM은 학습 시점 이후의 세계를 모릅니다. 사내 위키·어제자 로그·새로 올라온 PDF — 이 중 어느 것도 파라미터에 들어 있지 않습니다. **RAG(Retrieval-Augmented Generation)** 는 이 공백을 "프롬프트 시점에 필요한 문서를 찾아서 끼워 넣는" 방식으로 메웁니다.

컨텍스트 윈도우가 1M 토큰까지 열린 2026년에도 RAG가 사라지지 않은 이유는 간단합니다. **멀티 팩트 검색에서 long context의 실제 recall은 ~60%, RAG는 ~85%입니다.** 비용은 1,250배 차이입니다. 이 문서는 2026년 4월 기준 RAG 설계의 축 — 검색·청킹·임베딩·리랭킹·장문컨텍스트 경계 — 을 실무 판단 기준으로 정리합니다.

---

## 1. RAG의 구조

RAG는 두 단계로 나뉩니다. **인덱싱(Indexing)** 은 오프라인에서 문서를 벡터로 변환해 저장하고, **검색(Retrieval)** 은 런타임에 쿼리에 맞는 문서를 꺼내 LLM에 전달합니다.

```
[Indexing (offline)]
   Documents -> Chunking -> Embedding -> Vector DB

[Retrieval (runtime)]
   Query -> Embedding -> Vector DB Search -> Reranker -> LLM -> Answer
```

인덱싱은 한 번만 하지만 품질이 파이프라인 전체를 결정합니다 — 청크가 잘못 나뉘면 아무리 좋은 임베딩 모델도 복구할 수 없습니다. 검색은 매 쿼리마다 돌지만 각 단계의 지연이 누적되므로 (BM25 + Vector + Rerank + LLM) 지연 예산을 어떻게 배분할지가 설계 문제가 됩니다.

### 한 번 더 간결하게

| 단계 | 입력 | 출력 | 지연 (일반) |
|---|---|---|---|
| Chunking | 원문 | 청크 리스트 | 오프라인 |
| Embedding | 청크 | 벡터 | 오프라인 (런타임 쿼리는 <50ms) |
| Vector DB | 쿼리 벡터 | top-K 후보 | 2-10ms |
| Reranker | 쿼리 + top-K | top-M (M≪K) | 100-300ms |
| LLM | 쿼리 + top-M | 답변 | 500ms-20s |

리랭커와 LLM이 지연의 대부분을 차지합니다. 검색 단계 최적화보다 **리랭커 호출을 줄이는 구조 설계**(top-K를 작게, 리랭커 생략할 쿼리 조기 분기)가 체감 지연에 더 영향을 줍니다.

---

## 2. 검색 단계 — BM25·Dense·Hybrid

### 2.1 Sparse (BM25): 키워드가 정확히 일치해야 할 때

**BM25(Best Matching 25)** 는 1990년대 검색 엔진의 표준 스코어링 함수입니다. 쿼리 단어가 문서에 몇 번 등장하는지(TF), 전체 말뭉치에서 얼마나 희귀한 단어인지(IDF), 문서 길이를 어떻게 보정할지를 조합합니다.

BM25는 **정확 매칭**에서 강합니다:
- 제품 SKU, 에러 코드, IP 주소, 계정 ID 같은 식별자
- 전문 용어·약어가 많은 법률·의학·재무 도메인
- 쿼리에 고유명사가 섞인 경우

반대로 "업무 스트레스 해소법"처럼 의미는 같지만 표현이 다른 쿼리에서는 약합니다.

### 2.2 Dense (Vector): 의미가 비슷하면 찾는다

**Dense retrieval**은 문서와 쿼리를 같은 임베딩 공간의 벡터로 변환하고, 코사인 유사도로 가까운 문서를 찾습니다. "하늘이 우중충하네"와 "오늘 비 올 것 같다"가 가까이 오는 것이 dense 검색의 본질입니다.

Dense는 **의미 검색**에서 강합니다:
- 자연어 질의 ("어떻게 번아웃을 극복하나요")
- 바꿔쓰기·동의어가 많은 문의
- 크로스링구얼 (한국어 쿼리 → 영어 문서)

반대로 희귀 고유명사·숫자·약어에서는 "근처 벡터"가 오히려 독입니다 — 비슷한 패턴의 **다른** 엔티티를 함께 데려옵니다.

### 2.3 Hybrid: 둘 다 돌리고 RRF로 섞는다

2026년 프로덕션 RAG의 **최소 기준선**은 BM25 + Dense 병렬 실행입니다. 두 결과를 합치는 방법이 **RRF(Reciprocal Rank Fusion)**:

```
score(doc) = sum( 1 / (k + rank_i) )   # k = 60 기본값
```

각 retriever가 내놓은 **순위**만 사용해 합산합니다. 점수 스케일이 달라도 무관합니다. k=60은 상위 순위를 강하게 반영하되 하위 순위도 어느 정도 기여하게 하는 균형점입니다.

실험 결과 (공개 벤치마크 평균):

| 방식 | recall@10 |
|---|---|
| BM25 단독 | 65-70% |
| Dense 단독 | 70-78% |
| **BM25 + Dense + RRF** | **91%** |

추가 지연은 ~6ms. LLM 생성이 500ms~20초를 먹는 파이프라인에서 무시 가능한 수준입니다. Qdrant · Milvus · Weaviate · OpenSearch는 이 구조를 네이티브로 지원합니다.

---

## 3. 청킹 전략 — 인덱스 품질의 80%

청킹은 문서를 조각 내는 작업입니다. 잘못 자르면 **한 문장이 두 청크에 걸쳐 의미가 끊기거나**, 한 청크에 **관련 없는 두 주제가 섞여** 임베딩이 흐려집니다. 프로덕션 RAG의 실패 원인을 파고들면 절반 이상이 청킹 문제입니다.

### 3.1 기본기: Recursive splitter + overlap

가장 흔한 베이스라인:
- **크기**: 512 tokens (Sonnet/GPT 기준 약 350 단어)
- **오버랩**: 10% (앞 청크의 끝 50토큰을 다음 청크 앞에 중복)
- **분할 우선순위**: 문단 → 문장 → 토큰

LangChain `RecursiveCharacterTextSplitter`, LlamaIndex `SentenceSplitter` 가 표준. 여기서 "문장을 자르지 않으려는" 휴리스틱이 품질을 좌우합니다.

### 3.2 Semantic chunking: 의미가 바뀌는 지점에서 자른다

문장마다 임베딩을 구한 뒤, **인접 문장 유사도가 급락하는 지점**을 경계로 삼습니다. 주제가 바뀌는 자연스러운 경계를 찾아 자르므로 한 청크 안의 의미가 일관됩니다.

단점: 인덱싱 시 임베딩 호출이 2배 (청크 임베딩 + 경계 탐지용 문장 임베딩).

### 3.3 Late chunking: 문서 맥락을 청크에 녹인다

일반 임베딩은 청크를 **독립적으로** 벡터화합니다. "그(He)는 회의에 늦었다"라는 문장이 "스티브 잡스는 애플을 창업했다" 다음 청크에 있을 때, 일반 임베딩은 "그"가 누구인지 잃어버립니다.

**Late chunking**(Jina 2024, Voyage 2026 확장)은 문서 전체를 **long-context 임베딩 모델에 통째로** 넣은 뒤, 각 청크 범위의 토큰 임베딩을 평균 풀링해 청크 벡터를 만듭니다. 대명사·참조가 앞뒤 청크의 정보로부터 **잠재적으로 해결된** 상태로 벡터화됩니다.

효과: anaphoric reference(대명사·지시어) 쿼리에서 recall **+10~12%**.

### 3.4 Contextual Retrieval (Anthropic 2024)

청킹 문제를 LLM에게 직접 푸는 접근입니다. 각 청크를 저장하기 전에 **"이 청크가 문서 전체에서 어떤 맥락인지"** 를 요약한 50-100토큰을 앞에 붙입니다:

```
[원본 청크]
"매출이 15% 증가했다."

[Contextual 청크]
"SK AI 사업부의 2024년 4분기 재무보고서 중 클라우드 매출 부분이다. 매출이 15% 증가했다."
```

이 context 문자열은 Haiku급 작은 LLM이 생성합니다. 문서당 LLM 호출이 1회 필요하지만, **프롬프트 캐싱**과 결합하면 같은 문서를 청크별로 수십 번 돌려도 캐시 히트 90%+ — 문서당 실질 비용이 1/10로 떨어집니다. → [API 비용 최적화 §2 프롬프트 캐싱](./cost-optimization.md#2-프롬프트-캐싱--반복되는-입력의-90-할인)

효과: 단독 사용 시 retrieval failure **-49%**, 리랭커와 스택 시 **-67%**. Anthropic이 공식 블로그에서 공개한 수치로, 2026년 프로덕션 RAG의 고급 디폴트입니다.

### 실무 적용 순서

1. Recursive + 512 tok + 10% overlap으로 베이스라인
2. 품질이 부족하면 → Semantic chunking
3. 대명사·참조가 많은 문서(논문·계약서·대화록) → Late chunking
4. 여전히 retrieval failure가 보이면 → Contextual Retrieval (Haiku + 캐싱)

---

## 4. 임베딩 모델 선택 (2026-04)

전체 비교표는 [notes/rag-stack-2026.md](../notes/rag-stack-2026.md)에 있습니다. 실무 선택 축만 요약합니다.

| 우선순위 | 권장 모델 | 근거 |
|---|---|---|
| **비용 최우선** | Google Gemini Embedding ($0.006/MTok) | 2026-04 MTEB retrieval 1위(67.71), 멀티모달, 최저가 |
| **영어 + 기술문서** | Voyage-4-large ($0.18/MTok) | **MoE 임베딩 아키텍처 최초**. RTEB NDCG@10 기준 OpenAI 3-large +14%, Cohere v4 +8.2% |
| **다국어 (한국어 포함)** | Cohere embed-v4 ($0.10/MTok) | 100+ 언어, MTEB 65.2, 한국어 품질 검증됨 |
| **멀티모달 (이미지·PDF)** | Google Gemini Embedding | 텍스트·이미지·비디오·오디오·PDF를 한 벡터 공간으로 |
| **오픈소스·온프레미스** | BGE-M3 (BAAI) · NV-Embed-v2 (NVIDIA) | GPU 1장으로 자체 호스팅 가능 |

### 차원 압축 (Matryoshka)

**Matryoshka Representation Learning**은 한 임베딩 모델에서 차원을 잘라내도 품질 손실이 적도록 학습시키는 기법입니다. OpenAI `text-embedding-3-large`(3072차원)와 Google Gemini Embedding이 지원합니다.

3072차원 → 256차원으로 잘라도 MTEB 점수가 2-3% 수준만 떨어집니다. 저장 비용과 검색 지연이 동시에 줄어들므로, **수억 벡터 규모에선 Matryoshka가 경제성 게임체인저**입니다.

### 쿼리-문서 비대칭

일부 모델은 문서와 쿼리에 **다른 프롬프트 접두어**를 붙이도록 권장합니다:

```
passage: "Qdrant is a vector database written in Rust..."
query: "What language is Qdrant written in?"
```

Cohere·BGE·E5 계열이 해당됩니다. 이걸 빠뜨리면 같은 모델이라도 MTEB 점수가 5-10% 떨어집니다 — 문서에 명시된 prefix를 반드시 확인하세요.

---

## 5. 벡터 DB 선택 (2026-04)

```
                 [Scale vs Ops Overhead]

   Billion+     |                       Milvus
                |
   100M-1B      |            Qdrant (cluster)     Milvus
                |
   10M-100M     |   Qdrant         pgvectorscale
                |
   1M-10M       |   pgvector       Qdrant         Weaviate
                |
   < 1M         |   pgvector       SQLite-VSS     Chroma
                +-----------------------------------------
                  Self-hosted        Managed        Full-SaaS
```

### 규모별 선택 근거

- **< 1M 벡터**: 기존 PostgreSQL이 있다면 `pgvector`로 끝. 별도 DB 운영 비용이 벡터 검색 이득보다 큼
- **1M ~ 50M**: Qdrant(자체) 또는 pgvectorscale(TimescaleDB 팀의 StreamingDiskANN). 후자는 **50M에서 Qdrant 41 QPS → 471 QPS**(벤치마크) 로 10배 차이
- **50M ~ 1B**: Milvus 또는 Qdrant 클러스터. 샤딩·파티셔닝 성숙도가 갈림
- **1B+**: Milvus. billion 스케일에서 안정적 운영 레퍼런스 가장 많음
- **PoC·제로 오프스**: Pinecone · Weaviate Cloud. 1M 벡터까진 저렴, 10M+부터 비용 급증

### 2026년 주요 변화

- **Qdrant ACORN** — 기존 filtered HNSW의 recall drop 문제를 해결. `user_id=X AND date>Y` 같은 조건부 벡터 검색에서 경쟁력 최상
- **Milvus 2.6** (2026-Q1 Zilliz Cloud GA) — 2.5의 하이브리드 30x 가속에 더해 **hot/cold tiering** 추가. 오래된 벡터를 저렴한 스토리지로 계층화해 장기 보관 단가 절감
- **Voyage 4 shared embedding space** — nano/lite/standard/large가 같은 벡터 공간을 공유. 문서는 large로 인덱싱하고 쿼리는 lite로 날려도 재인덱싱 없이 호환 (2026-01 발표)
- **Pinecone Dedicated Read Nodes (DRN)** — 2025-12 프리뷰. 예측 가능한 지연·비용을 원하는 대형 워크로드용 전용 모드
- **Turbopuffer·LanceDB** — 오브젝트 스토리지(S3) 기반 서버리스 벡터 DB. 쓰기 지연은 높지만 저장 단가가 수십분의 1

### HNSW vs 기타 인덱스

대부분의 벡터 DB가 기본으로 쓰는 **HNSW(Hierarchical Navigable Small World)** 는 다층 근사 그래프로 정확도 99%+를 수 ms에 달성합니다. 변형:

- **IVF-PQ**: 메모리 절약 (8x 압축). 정확도 소폭 손실
- **DiskANN**: 메모리 부족 시 디스크 기반. pgvectorscale의 무기
- **ScaNN**: Google, 병렬 처리 최적화

현재 99%의 프로덕션은 HNSW로 충분합니다. DiskANN을 검토할 시점은 벡터가 1억 개를 넘어 메모리가 TB 단위로 필요해질 때입니다.

---

## 6. 리랭커 (Reranker)

### 왜 리랭커가 필요한가

Retrieval의 최적화 목표는 "정답을 놓치지 않기" 입니다 — 즉 **recall**. top-50까지는 넉넉히 잡습니다. 반면 LLM은 top-5~10만 받아야 합니다(많을수록 컨텍스트 오염). 이 **압축 단계**가 리랭커의 역할입니다.

Dense embedding은 "문서 하나 → 벡터 하나" 구조라 정보가 잃어집니다. **Cross-encoder reranker**는 쿼리와 문서를 **함께** 트랜스포머에 넣어 직접 관련도 점수를 냅니다. 계산 비용이 N배 크지만 N=50 정도는 감당 가능합니다.

### 선택지

| 유형 | 대표 모델 | 비용 | 특징 |
|---|---|---|---|
| **API 리랭커** | Cohere Rerank 3.5 | $2 / 1K searches (1 search = query + 100 docs) | 100+ 언어, 금융·법률 SOTA |
| | Voyage rerank-2 | $0.05 / 1M tokens | Voyage 임베딩과 스택 시 일관된 품질 |
| | Jina Reranker v2 | $0.02 / 1M tokens | 저비용 멀티링구얼 |
| **오픈소스** | bge-reranker-v2-m3 | GPU 운영비 | Cohere 3급 품질, GPU 1장으로 초당 수백 query |
| | bge-reranker-v2-gemma | GPU 운영비 (2B 모델) | 더 높은 품질, 더 무거움 |
| **LLM-as-reranker** | Claude Haiku · GPT-5 nano | $1~5 / 1M tok | "이 문서가 쿼리에 얼마나 관련 있나 1-5점" 프롬프트. 품질 최상이지만 비용 누적 |

### 실무 기본값

- **시작**: Cohere Rerank 3.5. 100개 문서당 $0.002로 사실상 공짜
- **Cohere 못 쓰는 환경** (폐쇄망·개인정보): bge-reranker-v2-m3 자체 호스팅
- **최고 품질 필요**: 상위 10개에만 LLM-as-reranker 적용. 비용은 수배 들지만 recall 오차가 비즈니스 손실로 직결되는 경우(법률 서치·의료)

### ColBERT: 한 단계 고급

**ColBERT(Contextualized Late Interaction over BERT)** 는 dense retrieval과 리랭커의 중간입니다. 문서를 벡터 1개로 요약하지 않고 **토큰마다 벡터**를 보관하고, 쿼리 토큰마다 best match를 찾아 MaxSim 집계합니다.

- 품질: cross-encoder 리랭커에 근접
- 지연: dense보다 느리지만 cross-encoder보단 빠름
- 스토리지: 토큰 수만큼 벡터 → 10-100x
- 구현: Vespa, Qdrant multivector, RAGatouille

일반 RAG에서는 필요 없습니다. **수천만~수억 문서 · 초당 수천 쿼리** 규모에서 Cohere 비용이 부담이 되는 시점에 검토합니다.

---

## 7. RAG vs Long Context vs Fine-tuning

세 기법은 "외부 지식을 LLM에 주입"하는 서로 다른 축입니다. 어떤 것이 맞는지는 **지식의 성격**이 결정합니다.

| 기준 | RAG | Long Context | Fine-tuning |
|---|---|---|---|
| 지식 주입 시점 | 런타임 검색 | 런타임 프롬프트에 통째 | 학습으로 가중치에 인코딩 |
| 업데이트 주기 | 수 분 단위 (벡터 DB re-index) | 즉시 (다음 프롬프트) | 학습 재실행 (시간·비용 큼) |
| 지식 용량 | 수 TB | ~1M tokens (수 MB) | 모델 크기 비례 |
| 비용/쿼리 (예: 1MB) | $0.002 | ~$2 (Claude 4.7 기준) | 학습비 분할상환 |
| 투명성 | 근거 문서 제시 가능 | 통째 전달이라 근거 추적 약함 | 불가 (파라미터에 녹음) |
| 정확도 (멀티 팩트) | ~85% | ~60% | 도메인 튜닝 시 상위 |
| 행동 양식 변경 | 약함 (문서 추가로 해결 안 됨) | 약함 | 강함 (어조·형식 고정) |

### 판단 기준

- **지식이 자주 바뀐다 / 용량이 크다 / 출처를 제시해야 한다** → RAG
- **단발 질의 + 전체 문서 통독 필요 + 고가 모델 감수** → Long Context
- **특정 어조·포맷·스타일을 영구 주입** → Fine-tuning (RAG와 병행 가능)

"코드베이스 전체 분석"처럼 한 문서 전체의 구조를 봐야 하는 작업은 long context가 맞고, "10,000개 고객 문의에서 유사 사례 찾기"는 RAG가 맞습니다. 실전 에이전트는 보통 **RAG로 후보 좁힌 뒤 선별된 문서는 long context로 통째 전달**하는 하이브리드입니다.

### 비용 감각

Gemini 2.5 Pro를 1M 토큰 long context로 돌리면 쿼리당 약 $2. 같은 질의를 RAG로 처리하면 top-10 문서(약 5K 토큰) + 생성(1K 토큰) ≈ $0.016. **약 125배 차이**. 수천 쿼리/일 규모에선 이 차이가 바로 인프라 예산을 결정합니다.

→ [API 비용 최적화 §5 컨텍스트 관리](./cost-optimization.md#5-컨텍스트-관리--비용도-선형이-아니다)

---

## 8. 고급 패턴 (2026)

### GraphRAG: 전체 말뭉치를 관통하는 질문

일반 RAG는 "이 문서의 이 부분에 답이 있다"를 찾습니다. "이 문서들 **전체**의 공통 주제는?", "주요 인물들의 관계망은?" 같은 **말뭉치 수준 질의**에는 답하지 못합니다.

**GraphRAG**(Microsoft, 2024)는 문서에서 엔티티·관계를 추출해 지식 그래프를 만들고, 커뮤니티 탐지 → 요약 → 계층적 색인을 구축합니다. 쿼리 시 그래프 구조를 이용해 "전체 관점의 답"을 생성합니다.

문제는 **인덱싱 비용**: 말뭉치당 $33,000 규모까지 치솟은 사례도 있습니다. 2024-11 발표된 **LazyGraphRAG**는 인덱싱 비용을 **0.1%까지 절감**(쿼리 시점 그래프 구축 방식) — 대부분의 팀에게 실용화 가능한 임계점을 넘겼습니다.

적합: 연구·컨설팅 보고서 생성, 큰 문서 모음의 요약·종합, 이해관계자 관계 분석.
부적합: 단순 FAQ·고객 문의 응대.

### Corrective RAG (CRAG) · Self-RAG

검색 결과를 평가해 **동적으로 전략을 바꾸는** 패턴:

- 검색 품질이 낮으면 → 웹 검색으로 fallback
- 답변이 근거 없이 생성되면 → 재검색
- 쿼리가 모호하면 → 쿼리 재작성 후 재검색

Claude Code의 agent 루프와 자연스럽게 결합합니다. 에이전트가 "이 문서로는 부족하다"고 판단하면 도구 호출로 추가 검색을 돌리는 구조. → [README §4 서브에이전트](../README.md#4-서브에이전트-컨텍스트를-위임하라)

### Query 재작성 (HyDE)

**HyDE(Hypothetical Document Embeddings)**: 쿼리가 짧거나 모호할 때, LLM에게 **"이 질문에 대한 가상의 답변"을 먼저 작성시키고**, 그 가상 답변을 임베딩해 검색합니다.

쿼리 "번아웃 대처" → 가상 답변 "번아웃은 장기 스트레스로 인한 정신적 소진 상태로, 휴식·전문 상담·업무 조정이 필요하다..." → 이걸 벡터화 → 훨씬 정확한 문서 검색.

짧은 쿼리가 많은 고객 지원·검색 앱에서 recall을 5-15% 끌어올리는 저비용 기법.

---

## 9. MCP를 통한 RAG 연결

에이전트가 RAG를 쓰는 방식은 두 가지입니다:

1. **에이전트 내부에 RAG 파이프라인 내장** — 코드로 retrieval 로직 작성, 에이전트가 직접 호출
2. **RAG를 외부 MCP 서버로 분리** — 에이전트는 "관련 문서 찾아줘" 도구만 호출

두 번째 패턴이 2026년 Claude Code 기반 스택의 기본입니다. Anthropic이 공개한 `filesystem`, `github`, `slack` MCP 서버처럼, 사내 문서 전용 RAG 서버를 하나 띄워두면:

```
Claude Code --(MCP 도구 호출)--> RAG 서버 --(검색 결과)--> Claude Code
```

에이전트 코드는 그대로 두고 RAG 스택만 독립적으로 개선할 수 있습니다. 임베딩 모델을 바꾸거나 리랭커를 추가해도 에이전트 쪽은 한 줄도 수정할 필요 없습니다. → [Claude Code 부록 — MCP](./claude-code.md)

이 MCP 분리 패턴은 Claude Code 뿐만 아니라 LangGraph·CrewAI·OpenAI Agents SDK 등 다른 프레임워크에서도 동일하게 쓰입니다. "RAG는 MCP 뒤로, 에이전트 프레임워크는 교체 가능"이 2026년 표준 구성입니다. → [에이전트 프레임워크 지형도 §10 Claude Code를 오케스트레이터로](./agent-frameworks.md#10-claude-code를-오케스트레이터로--통합-패턴)

이 구조의 추가 장점은 **프롬프트 캐싱이 자연스럽게 효율화된다는 점**입니다. 에이전트가 매 호출마다 검색 결과를 포함한 프롬프트 전체를 재생성하지 않고, 시스템 프롬프트·도구 정의는 캐시에 두고 검색 결과만 변동 블록으로 붙입니다. → [API 비용 최적화 §2](./cost-optimization.md#2-프롬프트-캐싱--반복되는-입력의-90-할인)

---

## 10. 설계 체크리스트

새 RAG 파이프라인을 설계할 때 빠르게 점검할 순서:

1. **지식의 성격이 RAG에 맞는가** — 업데이트 빈도·용량·출처 제시 요구 → 3번 중 하나라도 '예'면 RAG. 모두 '아니오'면 long context 또는 fine-tuning 먼저 검토
2. **쿼리의 성격** — 정확 매칭 비중 > 30%면 **Hybrid (BM25+Dense) 필수**. 의미 검색만이면 dense 단독도 가능
3. **청킹 전략** — 베이스라인은 recursive 512 + 10%, 참조·대명사 많은 도메인이면 late chunking 또는 Contextual Retrieval
4. **임베딩 모델** — 영어·기술문서 Voyage-3-large · 다국어 Cohere embed-v4 · 초저비용 Google Gemini Embedding · 폐쇄망 BGE-M3
5. **벡터 DB 규모** — 1M 이하 pgvector · 50M 이하 Qdrant · 이상 Milvus
6. **리랭커 필수** — top-50을 top-5로 압축. Cohere Rerank 3.5가 기본, 폐쇄망이면 bge-reranker-v2-m3
7. **LLM 비용 최적화** — 프롬프트 캐싱으로 시스템 프롬프트·도구 정의 고정, 검색 결과만 변동 블록. Haiku로 초안 → Opus로 검수 패턴 가능 → [cost-optimization §2, §4](./cost-optimization.md)
8. **MCP 분리 여부** — 에이전트 여러 대가 같은 RAG를 쓴다면 MCP 서버로 분리
9. **RAGAS 4-metric 벤치마크** — faithfulness·answer relevancy·context precision·context recall 기준점(v1)을 먼저 박제. 청킹·임베딩·리랭커 변경마다 동일 golden set으로 비교. → [LLM 평가 §8 RAG 평가 특화](./llm-evaluation.md#8-rag-평가-특화)

### 자주 틀리는 것

- **top-K를 크게 잡고 리랭커 생략** — 리랭커 없이 top-20을 LLM에 넣으면 컨텍스트 오염으로 품질 하락. top-K는 리랭커의 입력이지 LLM의 입력이 아님
- **쿼리와 문서에 같은 prefix를 쓰지 않음** — 일부 임베딩 모델은 prefix 비대칭이 기본(`passage:` vs `query:`). 빠뜨리면 10% 손실
- **청크 크기를 너무 크게** — 2048 tok 청크는 임베딩이 흐려짐. 한 청크는 **한 주제**여야 함
- **업데이트 주기 무시** — 일일 배치 리인덱싱 없이 "RAG니까 최신이겠지"는 착각. 업데이트 파이프라인이 RAG의 절반

---

## 네비게이션

**왔던 길** — 이 부록에 도달하는 전형적인 경로

- "사내 위키·매뉴얼을 LLM에 어떻게 연결?" → 여기
- "컨텍스트 1M 열린 시대에 RAG가 아직 필요해?" → 여기
- "청킹·임베딩·리랭커 뭘 어떻게 조합?" → 여기

**갈 길** — 이어 읽으면 자연스러운 3개

1. [LLM 평가](./llm-evaluation.md) — RAGAS 4-metric(faithfulness·answer relevancy·context precision/recall)로 RAG 품질 측정
2. [에이전트 프레임워크 지형도](./agent-frameworks.md) — RAG를 MCP로 분리한 에이전트 아키텍처
3. [API 비용 최적화](./cost-optimization.md) — long-context 대비 RAG의 1,250배 비용 우위를 실제 파이프라인으로 구현

---

## 연관 문서

- [API 비용 최적화](./cost-optimization.md) — 프롬프트 캐싱(90%)·배치 API(50%)·모델 라우팅이 RAG 파이프라인에 적용되는 방식
- [파라미터 착시: KV 캐시의 진실](./kv-cache.md) — long context의 물리 비용. RAG가 비용·지연 모두에서 우위인 근거
- [Claude Code 3대 설정 축](./claude-code.md) — MCP로 RAG 서버를 에이전트에 연결하는 패턴
- [에이전트 프레임워크 지형도](./agent-frameworks.md) — RAG를 MCP 뒤로 분리한 뒤 에이전트 프레임워크를 교체 가능하게 두는 2026년 표준 구성
- [LLM 평가](./llm-evaluation.md) — RAG는 retrieval·생성 두 축을 각각 평가해야 함. RAGAS의 4-metric(faithfulness·answer relevancy·context precision/recall)이 기준
- [경량 모델의 빠른 발전](./small-model-advancement.md) — 리랭커·쿼리 재작성·Contextual Retrieval에 쓰는 소형 모델의 역량
- [리즈닝 모델의 딜레마](./reasoning-models.md) — 쿼리 재작성·다단계 retrieval 계획에 reasoning 모델을 쓰는 경우의 비용/지연
- [트랜스포머 추론의 두 단계](./transformer-inference.md) — long-context의 O(n²) prefill이 RAG가 물리적으로 더 유리한 근거
- [notes/rag-stack-2026.md](../notes/rag-stack-2026.md) — 임베딩·벡터DB·리랭커 상세 비교표와 출처
