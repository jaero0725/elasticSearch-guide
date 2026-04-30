# 예제 5. 시맨틱 / 벡터 검색

문서·상품 설명·고객 문의를 의미 기반으로 검색하는 시나리오. dense_vector 필드, kNN 검색, BM25와의 hybrid retrieval(RRF), 그리고 9.x의 `semantic_text` / ELSER가 어떻게 결합되는지 본다.

대상 버전: Elasticsearch 8.16+ (`retriever`/RRF GA, 8.14 도입), 8.11+ (ELSER), 8.18 / 9.0 (`semantic_text` GA, 8.15 도입)

---

## 1. 요구사항

- 사내 위키/매뉴얼 문서 약 50만 개
- 검색 시나리오:
  - 자연어 질의 (`로그인이 안 될 때 어떻게 하나요`)
  - 정확한 키워드 매칭도 함께 (BM25)
  - 문서 부서/카테고리 필터
  - 답변 스니펫 추출 (highlight)
- 다국어 (한국어 + 영어 혼재)
- 임베딩 모델 후보:
  - ELSER v2 (sparse, 영어 위주)
  - E5 multilingual (dense, 한·영 모두 OK)
  - 자체 모델 (e.g. `BAAI/bge-m3`)

---

## 2. dense_vector 필드 기초

```json
PUT /docs
{
  "mappings": {
    "properties": {
      "doc_id":      { "type": "keyword" },
      "title":       { "type": "text" },
      "content":     { "type": "text" },
      "department":  { "type": "keyword" },
      "category":    { "type": "keyword" },
      "embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "hnsw",
          "m": 16,
          "ef_construction": 100
        }
      }
    }
  }
}
```

### 핵심 옵션

| 옵션 | 의미 |
|------|------|
| `dims` | 벡터 차원. 모델에 맞춰 설정 (e5-small=384, e5-large=1024, bge-m3=1024, OpenAI ada=1536) |
| `index: true` | HNSW 인덱스 생성 (kNN 가능). false면 brute force만 |
| `similarity` | `cosine` / `l2_norm` / `dot_product` / `max_inner_product` |
| `index_options.type: hnsw` | 기본. `int8_hnsw`, `int4_hnsw`로 양자화 가능 (8.13+) |
| `m` | HNSW 그래프 노드당 이웃 수. 16~32. 클수록 정확도↑ 메모리↑ |
| `ef_construction` | 색인 시 탐색 범위. 100~500. 클수록 색인 느림, 그러나 검색 정확도↑ |

### similarity 선택 가이드

```
cosine            : 벡터를 정규화하지 않은 모델, 일반적인 텍스트 임베딩
l2_norm           : 거리 기반, 이미지 임베딩에서 흔함
dot_product       : 사전에 정규화된 벡터 (모델이 normalize 보장 시) — 가장 빠름
max_inner_product : 정규화 안 된 dot product (Anthropic Voyage 등 일부)
```

> dot_product를 쓰면서 정규화 안 된 벡터를 넣으면 결과가 이상해진다. 모델 문서 확인 필수.

---

## 3. 양자화 (8.13+)

벡터는 메모리를 많이 먹는다. 50만 doc × 768 dim × 4 bytes = 약 1.5GB.

```json
"embedding": {
  "type": "dense_vector",
  "dims": 768,
  "index": true,
  "similarity": "cosine",
  "index_options": {
    "type": "int8_hnsw",   // 1 byte/dim, 4배 절감
    "m": 16,
    "ef_construction": 100
  }
}
```

옵션:

| 타입 | 메모리 절감 | 정확도 손실 |
|------|-----------|------------|
| `hnsw` | 1× | 0% |
| `int8_hnsw` | 4× | < 1% (대부분의 모델에서 무시 가능) |
| `int4_hnsw` (8.15+) | 8× | 1~3% |
| `bbq_hnsw` (8.18+/9.x) | 32× | 2~5% (rerank로 보정 가능) |

대부분의 프로덕션은 `int8_hnsw`가 좋은 시작점.

---

## 4. 색인 (외부 임베딩 모델)

```bash
# 1) 임베딩 생성 (애플리케이션에서 모델 호출)
# 2) 결과를 doc에 첨부해 인덱싱

POST /docs/_bulk
{ "index": { "_id": "D001" } }
{ "doc_id": "D001", "title": "로그인 문제 해결", "content": "비밀번호를 잊은 경우 ...", "department": "고객지원", "embedding": [0.012, -0.034, 0.087, ...] }
```

### 4.1 ELSER v2 (sparse)

ELSER는 sparse vector 모델. ES 안에서 모델을 다운받아 ingest pipeline에서 직접 추론한다.

```bash
# 1) ELSER 모델 다운로드 + start (Kibana ML 또는 API)
PUT _ml/trained_models/.elser_model_2_linux-x86_64/deployment/_start

# 2) 매핑에 sparse_vector 필드
PUT /docs-elser
{
  "mappings": {
    "properties": {
      "content":          { "type": "text" },
      "content_embedding":{ "type": "sparse_vector" }
    }
  }
}

# 3) ingest pipeline
PUT /_ingest/pipeline/elser-pipeline
{
  "processors": [
    {
      "inference": {
        "model_id": ".elser_model_2_linux-x86_64",
        "input_output": [
          { "input_field": "content", "output_field": "content_embedding" }
        ]
      }
    }
  ]
}

# 4) 인덱싱
POST /docs-elser/_doc?pipeline=elser-pipeline
{ "content": "비밀번호 재설정 절차" }
```

### 4.2 semantic_text (9.x — 가장 간편)

8.15에서 도입(tech preview), 8.18/9.0에서 GA. 매핑 한 줄이면 임베딩 생성/저장/검색이 모두 자동화된다.

```json
PUT /_inference/sparse_embedding/my-elser
{
  "service": "elasticsearch",
  "service_settings": {
    "num_allocations": 1,
    "num_threads": 1,
    "model_id": ".elser_model_2_linux-x86_64"
  }
}

PUT /docs-semantic
{
  "mappings": {
    "properties": {
      "content": {
        "type": "semantic_text",
        "inference_id": "my-elser"
      }
    }
  }
}

POST /docs-semantic/_doc
{ "content": "결제 실패 시 재시도 로직 안내" }
```

`semantic_text`는 청킹·임베딩·매핑 관리·검색 시 query 임베딩까지 모두 처리한다.

---

## 5. 검색 쿼리

### 5.1 단순 kNN

```json
GET /docs/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.011, -0.035, 0.082, ...],
    "k": 10,
    "num_candidates": 100,
    "filter": [
      { "term": { "department": "고객지원" } }
    ]
  }
}
```

| 옵션 | 의미 |
|------|------|
| `k` | 반환 개수 |
| `num_candidates` | HNSW 그래프 탐색 범위 (k의 5~10배 권장). 클수록 정확도↑ 느림↑ |
| `filter` | pre-filter. 후보 탐색 자체를 줄임 (post-filter 아님) |

### 5.2 query_vector_builder — 외부 모델로 자동 임베딩 (8.15+)

```json
GET /docs/_search
{
  "knn": {
    "field": "embedding",
    "k": 10,
    "num_candidates": 100,
    "query_vector_builder": {
      "text_embedding": {
        "model_id": "intfloat__multilingual-e5-small",
        "model_text": "로그인이 안 될 때 어떻게 하나요"
      }
    }
  }
}
```

쿼리 텍스트만 보내면 ES가 모델을 호출해 벡터로 변환 → kNN 실행. 클라이언트 코드가 단순해진다.

### 5.3 Hybrid: kNN + BM25 + RRF (8.14+ retriever)

`retriever`는 8.14에 도입된 검색 추상화. 여러 retriever 결과를 RRF로 합칠 수 있다.

```json
GET /docs/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "multi_match": {
                "query": "로그인 안 될 때",
                "fields": ["title^2", "content"]
              }
            }
          }
        },
        {
          "knn": {
            "field": "embedding",
            "k": 50,
            "num_candidates": 200,
            "query_vector_builder": {
              "text_embedding": {
                "model_id": "intfloat__multilingual-e5-small",
                "model_text": "로그인 안 될 때"
              }
            }
          }
        }
      ],
      "rank_window_size": 50,
      "rank_constant": 60
    }
  },
  "size": 10,
  "_source": ["doc_id", "title", "content"]
}
```

### RRF (Reciprocal Rank Fusion) 동작

```
각 retriever가 자기 순위(rank=1, 2, 3, ...)를 매김
RRF score(d) = Σ over retrievers: 1 / (rank_constant + rank(d))

장점:
  - 점수 스케일이 다른 retriever들을 정규화 없이 합치기 좋음
  - BM25 (텍스트) + kNN (시맨틱)의 상호보완
  - 조정 파라미터가 rank_constant 하나뿐
```

**왜 합쳐야 하나?**

```
BM25만:   "에러 코드 E1023" 같은 정확한 토큰 매칭에 강함
          그러나 동의어/의도/오타에 약함
kNN만:    "로그인 안 됨" ↔ "비밀번호 입력 실패" 의미 매칭 OK
          그러나 짧은 키워드/고유명사 약함
RRF 합산: 두 약점을 보완 — 일반적으로 nDCG가 5~15% 향상
```

### 5.4 semantic_text 쿼리 (9.x)

```json
GET /docs-semantic/_search
{
  "query": {
    "semantic": {
      "field": "content",
      "query": "로그인이 안 될 때 어떻게 하나요"
    }
  }
}
```

`semantic_text` 필드면 매핑 시 지정한 inference로 자동 임베딩 → 검색.

---

## 6. 운영 팁

### 6.1 메모리 계산

```
HNSW 그래프 메모리 ≈ docs * dims * (bytes_per_dim + m * 8)
  - hnsw (float32):  768 * (4 + 16*8) = 약 100KB / doc → 50만 doc = 50GB
  - int8_hnsw:       768 * (1 + 16*8) = 약 100KB / doc (그래프는 동일) → 데이터만 4× 절감
실질 메모리 사용:
  - vector data (off-heap, mmap 가능)
  - HNSW 그래프 (RAM에 올라와야 빠름)
권장: 노드당 dense_vector 데이터 합 < (전체 RAM의 50%)
```

### 6.2 차원 수 트레이드오프

```
dims 감소: 메모리/디스크 절감, 검색 빠름, 정확도 약간 저하
  e5-small (384) vs e5-base (768) vs e5-large (1024)
  → 작은 모델로 시작해서 nDCG@10 측정 후 결정
```

### 6.3 색인 속도

```
HNSW 색인은 일반 색인보다 느리다 (그래프 빌드 비용).
대량 색인 시:
  - bulk 사이즈 증가 (5~10MB)
  - refresh_interval = "60s"
  - replicas = 0 (색인 후 1로)
  - ef_construction은 100에서 시작, 정확도 부족 시 200~500
```

### 6.4 진단 쿼리

```bash
# kNN이 실제로 사용 중인지
GET /docs/_search
{
  "profile": true,
  "knn": { "field": "embedding", "query_vector": [...], "k": 10, "num_candidates": 100 }
}
# response.profile.shards[*].dfs.knn 에 시간/노드 정보

# 모델 deployment 상태
GET /_ml/trained_models/_stats
GET /_ml/trained_models/.elser_model_2_linux-x86_64/_stats

# inference endpoint
GET /_inference/_all
```

### 6.5 안티패턴

- **임베딩 캐시 안 함**: 같은 쿼리 텍스트 반복인데 매번 모델 호출. 애플리케이션에서 LRU 캐시 필수
- **차원 수 변경 후 reindex 안 함**: dims는 매핑 후 변경 불가 → 새 인덱스로 마이그레이션
- **filter 위치 잘못**: `knn.filter`(pre)가 아니라 `bool.filter` 안에 kNN을 넣으면 후보가 K개로 좁혀진 뒤 필터링 → 결과 부족. 8.x는 `knn.filter`로 pre-filter 가능
- **`num_candidates` 너무 작음**: k=10에 num_candidates=10이면 정확도 급락. 최소 50, 일반적으로 100~200

---

## 7. 요약

| 항목 | 선택 |
|------|------|
| 벡터 필드 | `dense_vector` (`hnsw` 또는 `int8_hnsw`) |
| 유사도 | 모델에 맞춰 `cosine` / `dot_product` / `max_inner_product` |
| 색인 자동화 | `semantic_text` (9.x) 또는 ingest pipeline + inference |
| 검색 | `knn` 절 + `query_vector_builder` |
| Hybrid | `retriever.rrf` 로 BM25 + kNN 결합 |
| 양자화 | `int8_hnsw` 가 메모리 4× 절감, 정확도 손실 거의 없음 |
| 메모리 가이드 | dense_vector data < node RAM의 50% |
| 한국어/다국어 | E5 multilingual / bge-m3 등 dense 모델, ELSER는 영어 위주 |
