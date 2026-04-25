# RAG 스택 2026 — 임베딩·벡터DB·리랭커 참조

2026-04-25 조사 기준. `appendix/rag-architecture.md` 작성용 원자료.

---

## 1. 임베딩 모델 비교 (2026-04)

| 모델 | 단가 (/1M tok) | 차원 | 최대 입력 | 특이점 |
|---|---|---|---|---|
| **OpenAI text-embedding-3-large** | $0.13 | 3072 (→256 축소 가능) | 8K | Matryoshka representation (차원 줄여도 품질 유지) |
| **OpenAI text-embedding-3-small** | $0.02 | 1536 | 8K | 저비용 기본값 |
| **Cohere embed-v4** | $0.10 (검색 최적화 세팅 기준; 일부 요약글의 $0.01은 보조 임베딩 단가) | 1024 / 1536 / 256 | 128K | MTEB 65.2 (3-large 64.6 초과), 100+ 언어, 멀티모달 |
| **Voyage-3-large** | $0.18 (첫 200M 무료) | 1024/2048 | 32K | 기술문서·코드 특화, 3-large 대비 +9.74% |
| **Voyage-4-lite** (2026-01 출시) | $0.02 | 1024 | 32K | voyage-3.5 계열과 동일 단가로 성능 업 |
| **Voyage-4** | $0.12 | 1024 | 32K | v4 기본 라인 |
| **Voyage-4-large** | $0.18 | 1024-2048 | 32K | v4 최상위, **MoE 아키텍처** (임베딩 모델 중 최초), MTEB/RTEB에서 OpenAI-3-large +14%, Cohere v4 +8.2% 상회 |

> **Voyage 4 shared embedding space (2026-01)**: nano·lite·standard·large가 전부 동일 벡터 공간. 문서는 large로 embedding, 쿼리는 lite로 embedding해도 재인덱싱 없이 호환 — 비용·성능 트레이드오프를 쿼리 시점에 선택 가능.
| **Google Gemini Embedding** | $0.006 | 3072 (Matryoshka) | 8K | MTEB retrieval 67.71 (2026-04 기준 1위), 멀티모달 (텍스트·이미지·비디오·오디오·PDF) |
| **BGE-M3 (오픈소스)** | 자체 호스팅 | 1024 | 8K | BAAI, 다국어, sparse+dense+multi-vector 동시 출력 |
| **NV-Embed-v2 (오픈소스)** | 자체 호스팅 | 4096 | 32K | NVIDIA, MTEB 영어 상위 |

### 출처
- [TokenMix: Text Embedding Models 2026](https://tokenmix.ai/blog/text-embedding-models-comparison) — Google $0.006/M 확인
- [DeployBase: Best Embedding Models & APIs in 2026](https://deploybase.ai/articles/best-embedding-models)
- [Voyage AI blog — voyage-3-large](https://blog.voyageai.com/2025/01/07/voyage-3-large/) — 3-large가 OpenAI-v3-large 대비 +9.74%
- [Voyage docs — Pricing](https://docs.voyageai.com/docs/pricing)
- [Awesome Agents: MTEB March 2026](https://awesomeagents.ai/leaderboards/embedding-model-leaderboard-mteb-march-2026/)
- [pecollective: Best Embedding Models 2026](https://pecollective.com/tools/best-embedding-models/)

### 선택 기준 (실무)
- **비용 최우선**: Google Gemini Embedding ($0.006) 또는 OpenAI 3-small ($0.02)
- **영어 + 기술문서**: Voyage-3-large (코드·API 문서)
- **다국어 (한국어 포함)**: Cohere embed-v4 또는 BGE-M3
- **멀티모달 (이미지·PDF)**: Google Gemini Embedding 또는 Voyage-multimodal
- **오픈소스·온프레미스**: BGE-M3 / NV-Embed-v2

---

## 2. 벡터 DB 비교 (2026-04)

### 성능 (100K~10M 벡터, HNSW 기준)

| DB | p99 latency | QPS | 특장점 | 약점 |
|---|---|---|---|---|
| **Qdrant** | 2ms | 12,000 | ACORN 알고리즘(2026)으로 filtered HNSW 해결, Rust 성능 | 대규모(>100M)는 Milvus보다 튜닝 필요 |
| **FAISS (라이브러리)** | 3ms | 15,000 | 단일 프로세스 raw 성능 최상 | 분산·영속성 직접 구현 |
| **Milvus** | 5ms | 8,000 | billion 스케일·샤딩 성숙, 2.5에서 하이브리드 30x 가속 | 운영 복잡 |
| **pgvector** (PostgreSQL) | 변동 | 중 | 기존 PG 인프라 재사용, pgvectorscale은 50M에서 471 QPS | 빌리언 스케일엔 부적합 |
| **Pinecone** | 8ms | 5,000 | 완전 관리형 serverless | 볼륨 커지면 비용 급증 |
| **Weaviate** | 10ms | 4,000 | 하이브리드·멀티모달 내장, GraphQL | raw 속도는 상대적 하위 |

### 규모별 선택

| 규모 | 권장 |
|---|---|
| ~1M 벡터 | pgvector (기존 PG 있으면 무조건) |
| 1M~50M | Qdrant · pgvectorscale |
| 50M~1B | Milvus · Qdrant (클러스터) |
| 1B+ | Milvus |
| PoC · 제로 오프스 | Pinecone · Weaviate Cloud |

### 출처
- [Vecstore: Vector DB Performance Compared](https://vecstore.app/blog/vector-database-performance-compared)
- [Firecrawl: Best Vector Databases in 2026](https://www.firecrawl.dev/blog/best-vector-databases)
- [DataCamp: Top 5 Vector DBs 2026](https://www.datacamp.com/blog/the-top-5-vector-databases)
- [Qdrant ACORN 알고리즘](https://qdrant.tech/articles/filtered-hnsw/) — filtered HNSW 정확도 개선

### 2026년 핵심 변화
- **Qdrant ACORN** — 기존 filtered HNSW의 recall drop 문제를 구조적으로 해결. filter 조건 있는 검색(예: `user_id=X AND date>Y`)에서 성능 급상승
- **Milvus 2.6** — 2026-Q1 Zilliz Cloud GA. hot/cold tiering으로 콜드 벡터를 저렴한 스토리지로 계층화 (장기 보관 코스트 절감). 2.5의 하이브리드 30x 가속은 유지.
- **pgvectorscale** — TimescaleDB 팀이 공개, pgvector 위에 StreamingDiskANN 레이어. 50M 벡터에서 Qdrant 41 QPS → 471 QPS
- **Pinecone Dedicated Read Nodes (DRN)** — 2025-12 공개 프리뷰. 예측 가능한 성능·비용을 원하는 볼륨 고객용
- **Turbopuffer·LanceDB** — 오브젝트 스토리지(S3) 기반 신생 DB, serverless 단가 경쟁력

---

## 3. 리랭커 (Reranker) 비교 (2026-04)

### API 리랭커

| 모델 | 가격 | 지연 | 특징 |
|---|---|---|---|
| **Cohere Rerank 3.5** | $2.00 / 1K searches (1 search = query + 최대 100 docs) | ~200ms | 100+ 언어, 금융·호스피탈리티 SOTA |
| **Cohere Rerank 4 Fast** | $5.00 / search unit (min $3,250 커밋) | ~150ms | 최신 경량 |
| **Cohere Rerank 4 Pro** | $5.00 / search unit (min $3,250 커밋) | ~300ms | 최상 품질 |
| **Voyage rerank-2** | $0.05 / 1M tokens | ~200ms | Voyage 임베딩과 스택 |
| **Jina Reranker v2** | $0.02 / 1M tokens | ~150ms | 멀티링구얼, 저비용 |

### 오픈소스 리랭커

| 모델 | 크기 | 특징 |
|---|---|---|
| **bge-reranker-v2-m3** | 568M | BAAI, 다국어, Cohere 3급 품질 (MTEB rerank) |
| **bge-reranker-v2-gemma** | 2B | Gemma 기반, 성능 상위 |
| **mxbai-rerank-large** | 500M | Mixedbread, 영어 특화 |
| **jina-reranker-v2-base-multilingual** | 278M | 경량 |

### 출처
- [Cohere Pricing](https://cohere.com/pricing)
- [MetaCTO: Cohere API Pricing 2026](https://www.metacto.com/blogs/cohere-pricing-explained-a-deep-dive-into-integration-development-costs)
- [Pinecone Docs: cohere-rerank-3.5](https://docs.pinecone.io/models/cohere-rerank-3.5)

### 실무 패턴
- **API 사용**: 대부분의 프로덕션 RAG는 Cohere Rerank 3.5가 기본값. 100 docs당 $0.002로 매우 저렴
- **자체 호스팅**: bge-reranker-v2-m3 GPU 1장(A10·L4)에서 초당 수백 query 처리 가능
- **LLM-as-reranker**: Haiku·GPT-5 nano로 "이 문서가 쿼리에 얼마나 관련 있나 1-5점" 프롬프트. 품질 최상이지만 토큰 비용 누적 → 상위 10개 정도에만 적용

---

## 4. RAG vs Long Context (2026)

### 벤치마크 현실

| 지표 | Gemini 1.5/2.5 Pro (1M ctx) | RAG |
|---|---|---|
| 단일 Needle in Haystack | 99.7% recall | 95%+ (hybrid+rerank) |
| **멀티 팩트 검색 (실전)** | ~60% recall | **85%+** |
| 1M 토큰 query 지연 | 30-60초 | ~500ms |
| 비용 (1M tok query) | ~$2 (Claude 4.7 기준 $5) | ~$0.002 (retrieval + 짧은 프롬프트) |
| 비용 배수 | **~1,250x** | 기준 |

### 출처
- [TianPan: Long-Context vs RAG Production Framework (2026-04)](https://tianpan.co/blog/2026-04-09-long-context-vs-rag-production-decision-framework)
- [U-NIAH: Unified RAG and LLM Evaluation (ACM TOIS)](https://dl.acm.org/doi/10.1145/3786609)
- [LangChain: Multi Needle in a Haystack](https://blog.langchain.com/multi-needle-in-a-haystack/)

### 판단 기준
- 문서량 < 컨텍스트 윈도우 & 단발 질의 → **Long context + 캐싱**
- 문서량 수백 MB 이상, 반복 질의, 업데이트 잦음 → **RAG**
- 둘이 섞이는 에이전트형 워크로드 → **RAG가 기본, 선별 문서는 long context로 통째 전달**

---

## 5. 청킹 전략 (2026)

| 방식 | 핵심 | 추가 비용 | 효과 |
|---|---|---|---|
| **Fixed-size** (512/1024 tokens) | 토큰 기준 균등 분할 | 0 | 베이스라인, recall 65% 수준 |
| **Recursive text splitter** | 문단 → 문장 → 토큰 계층 | 미미 | 가장 흔한 기본값 |
| **Semantic chunking** | 문장 임베딩 유사도 급락 지점에서 자름 | 임베딩 비용 1회 추가 | recall 5-10% 상승 |
| **Late chunking** (Jina 2024, Voyage 2026) | 문서 전체를 long-context 임베딩 → 청크 단위로 평균 pooling | 임베딩 1회 (긴 입력) | anaphoric reference(대명사 참조)에서 +10-12% |
| **Contextual Retrieval** (Anthropic 2024) | LLM이 각 청크에 "이 청크가 전체 문서에서 어느 맥락인지" 50-100토큰 자동 prepend | 문서당 LLM 호출 1회 (캐싱하면 저렴) | retrieval failure -49%, rerank 결합 시 -67% |

### 출처
- [Anthropic: Introducing Contextual Retrieval (2024-09)](https://www.anthropic.com/news/contextual-retrieval) — 원 논문
- [Weaviate: Chunking Strategies for RAG](https://weaviate.io/blog/chunking-strategies-for-rag)
- [Jina AI: Late Chunking](https://jina.ai/news/late-chunking-in-long-context-embedding-models/)

### 실무 순서
1. Fixed 512 토큰 + 10% overlap으로 시작
2. 품질 부족하면 → Semantic chunking
3. 대명사·참조가 많은 문서(논문·계약서) → Late chunking
4. 여전히 recall 문제면 → Contextual Retrieval + 프롬프트 캐싱 (Anthropic 권장 스택)

---

## 6. 하이브리드 검색 (Hybrid Search)

### 구조

```
Query
  ├─ BM25 (sparse, 키워드 매칭) ─┐
  ├─ Vector (dense, 의미 검색) ──┤─── RRF(k=60) ───> Top-K candidates ──> Reranker ──> LLM
  └─ (선택) ColBERT late interaction ┘
```

### RRF (Reciprocal Rank Fusion)
```
score(doc) = Σ (1 / (k + rank_i))   # k=60 기본값
```
- 여러 retriever의 **순위**만 사용 (점수 스케일 무관)
- k=60이 업계 표준 (다양성 vs 상위 가중 균형)

### 효과
- BM25 단독: recall@10 65-70%
- Vector 단독: recall@10 70-78%
- **Hybrid + RRF: recall@10 91%**
- 추가 지연: ~6ms (LLM 500-2000ms 대비 무시 가능)

### 언제 Hybrid가 필수인가
- 제품 SKU·에러 코드·IP·계정 ID 등 **정확 매칭** 필요
- 전문 용어·약어가 많은 도메인 (법률·의학·재무)
- 쿼리에 고유명사가 섞임

### 출처
- [Elastic: Hybrid Search Guide](https://www.elastic.co/what-is/hybrid-search)
- [Supermemory: Hybrid Search Guide (2026-04)](https://blog.supermemory.ai/hybrid-search-guide/)
- [Weaviate: Hybrid Search Explained](https://weaviate.io/blog/hybrid-search-explained)

---

## 7. 고급 패턴 (2026)

### ColBERT late interaction
- 기존 dense: 문서 전체를 벡터 1개로 요약 → 정보 손실
- ColBERT: **토큰 단위 벡터** 다수 보존, 쿼리 토큰마다 best match 찾아 MaxSim 집계
- 품질 ↑↑, 스토리지·지연 ↓↓
- 실무 패턴: BM25/dense로 100~1000개 후보 추림 → ColBERT 리랭크
- 구현체: `Vespa`, `Qdrant multivector`, `RAGatouille`

### GraphRAG (Microsoft)
- 문서 → 엔티티·관계 추출 → 커뮤니티 탐지 → 요약 → 계층적 색인
- **장점**: "전체 말뭉치를 관통하는 질문"("이 보고서들의 공통 주제는?")에 압도적
- **단점**: 인덱싱 비용이 일반 RAG 대비 100-1000x ($33K / 대형 코퍼스 사례)
- **LazyGraphRAG** (2024-11 발표): 인덱싱 비용을 0.1%까지 절감 — 쿼리 시 그래프 구축

### Contextual Retrieval (Anthropic)
- 각 청크 앞에 "이 청크가 어느 문서의, 어느 섹션의, 어떤 맥락인지" 50-100토큰 자동 추가
- LLM(보통 Haiku)이 문서 전체 + 청크를 보고 생성
- **프롬프트 캐싱 결합 시** 문서당 LLM 호출의 90%가 캐시 히트 → 거의 공짜
- retrieval failure -49%, reranker 스택 시 -67%

### 출처
- [ColBERT: Late Interaction (Medium)](https://medium.com/@mpuig/colbert-and-beyond-advancing-retrieval-techniques-81df1b2324d6)
- [Microsoft GraphRAG](https://microsoft.github.io/graphrag/)
- [Graph RAG in 2026: Practitioner's Guide](https://medium.com/graph-praxis/graph-rag-in-2026-a-practitioners-guide-to-what-actually-works-dca4962e7517)
- [Anthropic Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)

---

## 8. 2026년의 RAG 스택 권장 디폴트

```
[문서 수집]
   │
   ├─ Contextual Retrieval 전처리 (Haiku + prompt caching)
   │
   ▼
[청킹]  Recursive splitter (512 tokens, 10% overlap)
   │
   ▼
[임베딩]  Cohere embed-v4  OR  Voyage-3-large  OR  Google Gemini Embedding
   │
   ▼
[저장]  Qdrant (수천만 이하)  OR  Milvus (빌리언)  OR  pgvector (통합 선호)
   │
   ▼
[검색 (런타임)]
   ├─ BM25 (Elasticsearch / OpenSearch / Qdrant sparse)
   └─ Dense (벡터 DB)
   │
   ▼ RRF(k=60)
   │
   ▼
[리랭크]  Cohere Rerank 3.5  OR  bge-reranker-v2-m3 (자체)
   │
   ▼ top-5~10
   │
   ▼
[LLM]  프롬프트 캐싱 + 모델 라우팅
```

---

## 9. 관련 비용 분석

RAG 파이프라인 자체의 비용 구조:

| 단계 | 1M 쿼리 기준 비용 |
|---|---|
| 쿼리 임베딩 | $0.006~$0.18 (모델 선택) |
| Vector 검색 | $0 (자체) ~ ~$1 (Pinecone serverless) |
| BM25 검색 | $0 (Elasticsearch 자체) |
| Reranker | $2,000 (Cohere 3.5, query + 100 docs 기준) |
| LLM 생성 | $500~$25,000 (Haiku~Opus, 10K tok 출력 기준) |

→ 실제 총비용의 **90%는 LLM 생성 단계**. 리랭커·임베딩은 비용보다 품질 증분으로 판단.
