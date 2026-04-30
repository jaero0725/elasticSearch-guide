# Query DSL 퀵레퍼런스

자주 쓰는 쿼리 패턴 모음. 8.x / 9.x 기준.

---

## 컨텍스트 구분

| Context | 점수(score) 계산 | 캐시 | 용도 |
|---------|-----------------|------|------|
| **query** | 함 (relevance) | query cache | 관련도 정렬 |
| **filter** | 안 함 (yes/no) | request + node cache | 필터링·집계 전 좁히기 |

```json
{
  "query": {
    "bool": {
      "must":   [{ "match": { "title": "elastic" } }],   // score 영향
      "filter": [{ "term":  { "status": "published" } }] // score 무관, 캐시
    }
  }
}
```

**원칙**: 점수가 필요 없으면 무조건 `filter`. 캐시까지 받는다.

---

## 1. Term-level (정확 일치)

```json
// 단일 값
{ "term": { "status": "active" } }

// 다중 값 (OR)
{ "terms": { "tags": ["red", "blue", "green"] } }

// 범위
{ "range": { "price": { "gte": 100, "lt": 500 } } }
{ "range": { "@timestamp": { "gte": "now-1h", "lte": "now" } } }

// 존재 여부
{ "exists": { "field": "user.email" } }

// ID 직접
{ "ids": { "values": ["1", "2", "3"] } }

// prefix (high-cardinality 주의)
{ "prefix": { "user_id": "u-" } }
```

**핵심**: term은 분석하지 않는다. `text` 필드에 term 쓰면 미스. `keyword` 또는 `.keyword` multi-field에 사용.

---

## 2. Full-text

```json
// 단일 필드
{ "match": { "title": "elasticsearch tutorial" } }

// AND 조합
{ "match": { "title": { "query": "elasticsearch tutorial", "operator": "and" } } }

// 구문 (단어 순서 + 위치 일치)
{ "match_phrase": { "title": "elasticsearch tutorial" } }

// 구문 + slop (단어 사이 거리 허용)
{ "match_phrase": { "title": { "query": "elasticsearch tutorial", "slop": 2 } } }

// 자동완성 (prefix matching on last term)
{ "match_phrase_prefix": { "title": "elastic tut" } }
```

---

## 3. multi_match: 여러 필드 동시 매칭

```json
{
  "multi_match": {
    "query": "elasticsearch tutorial",
    "fields": ["title^3", "body", "tags^2"],
    "type": "best_fields"
  }
}
```

### 타입 비교

| type | 동작 | 권장 케이스 |
|------|------|-------------|
| **best_fields** (기본) | 가장 잘 맞는 단일 필드의 점수 | 검색어가 한 필드에 모여있을 때 |
| **most_fields** | 모든 필드 점수의 합 | 같은 텍스트가 여러 analyzer로 색인된 경우 (e.g., 영어 + 한국어 분석기) |
| **cross_fields** | 모든 필드를 합쳐 하나의 큰 필드처럼 매칭 | 이름·주소처럼 단어가 여러 필드에 흩어진 경우 |
| **phrase**         | 각 필드에 phrase match 후 best | "정확한 구문" 검색 |
| **phrase_prefix**  | phrase + 마지막 prefix | 구문 자동완성 |
| **bool_prefix**    | bool + 마지막 prefix | 검색어 자동완성 (search-as-you-type 필드와 함께) |

```json
// cross_fields 예: "John Smith"가 first_name + last_name에 분산
{
  "multi_match": {
    "query": "John Smith",
    "fields": ["first_name", "last_name"],
    "type": "cross_fields",
    "operator": "and"
  }
}
```

---

## 4. bool 조합

```json
{
  "bool": {
    "must":     [ /* score 영향, AND */ ],
    "should":   [ /* score 영향, OR. minimum_should_match로 강제 */ ],
    "must_not": [ /* 제외, score 무관 */ ],
    "filter":   [ /* yes/no, score 무관, 캐시 O */ ]
  }
}
```

### minimum_should_match

```json
{
  "bool": {
    "should": [
      { "match": { "title": "elastic" } },
      { "match": { "title": "search" } },
      { "match": { "title": "engine" } }
    ],
    "minimum_should_match": 2
  }
}
```

3개 중 최소 2개 매칭. 검색어 토큰 수에 따라 동적 지정도 가능: `"75%"`.

### 실무 패턴: 점수 + 필터

```json
{
  "query": {
    "bool": {
      "must":   { "match": { "title": "elastic" } },
      "filter": [
        { "term":  { "status": "published" } },
        { "range": { "@timestamp": { "gte": "now-7d" } } }
      ]
    }
  }
}
```

---

## 5. function_score: 점수 커스터마이즈

```json
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "functions": [
        {
          "filter": { "term": { "category": "premium" } },
          "weight": 2
        },
        {
          "gauss": {
            "@timestamp": {
              "origin": "now",
              "scale":  "1d",
              "decay":  0.5
            }
          }
        },
        {
          "field_value_factor": {
            "field":   "popularity",
            "factor":  1.2,
            "modifier":"log1p",
            "missing": 1
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

| `score_mode` | 함수들의 점수 결합 |
|--------------|-------------------|
| multiply (기본) | 곱 |
| sum | 합 |
| avg | 평균 |
| max / min | 최대 / 최소 |

| `boost_mode` | query 점수와 함수 점수 결합 |
|--------------|-------------------------|
| multiply (기본) | 곱 |
| sum | 합 |
| replace | 함수 점수만 사용 |

**권장**: 새 코드에서는 `distance_feature`, `script_score` 쿼리 또는 `rank_feature`를 우선 검토. function_score는 무겁다.

---

## 6. kNN search (벡터 검색)

```json
GET /my-index/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.12, -0.43, ...],
    "k": 10,
    "num_candidates": 100,
    "filter": { "term": { "status": "active" } }
  }
}
```

| 파라미터 | 의미 |
|----------|------|
| `k` | 최종 반환 이웃 수 |
| `num_candidates` | shard별 후보 수 (recall vs latency trade-off) |
| `filter` | 사전 필터링 (post-filter 아님) |

### Mapping (HNSW)

```json
{
  "mappings": {
    "properties": {
      "embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}
```

`similarity`: `cosine` | `dot_product` | `l2_norm` | `max_inner_product`

### 하이브리드 검색

```json
{
  "query": { "match": { "title": "machine learning" } },
  "knn":   { "field": "embedding", "query_vector": [...], "k": 10, "num_candidates": 100 }
}
```

top-level `query` + `knn`을 동시 사용하면 두 점수의 **합**(`bm25_score + knn_score`)이 기본 결합이다. RRF (Reciprocal Rank Fusion)는 자동으로 적용되지 않으며, **retriever** 절을 명시적으로 써야 한다 (`retriever: { rrf: { retrievers: [...] } }`, 8.16 GA). 별도 예제는 가이드 참조.

---

## 7. Retrievers (8.x 후반 ~ 9.x)

Query, kNN, RRF, semantic을 합성하는 새로운 추상.

```json
GET /my-index/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": { "match": { "title": "elasticsearch" } }
          }
        },
        {
          "knn": {
            "field": "embedding",
            "query_vector": [...],
            "k": 50,
            "num_candidates": 200
          }
        }
      ],
      "rank_window_size": 50,
      "rank_constant": 60
    }
  }
}
```

### text_similarity_reranker (재정렬)

```json
{
  "retriever": {
    "text_similarity_reranker": {
      "retriever": { "standard": { "query": { "match": { "body": "..." } } } },
      "field": "body",
      "inference_id": "my-rerank-model",
      "inference_text": "user question text"
    }
  }
}
```

검색 결과 상위 N개를 cross-encoder로 재정렬. 정확도 향상, latency 추가.

---

## 8. 정렬·페이징

```json
{
  "size": 20,
  "sort": [
    { "@timestamp": "desc" },
    "_score",
    { "_id": "asc" }       // tiebreaker
  ],
  "search_after": [1714435200000, null, "doc-id-prev"]
}
```

**`from + size`는 10000 미만에서만**. 그 이상은 `search_after` + PIT.

```
POST /my-index/_pit?keep_alive=1m
GET  /_search { "pit": {"id":"...","keep_alive":"1m"}, "sort":[...], "search_after":[...] }
```

---

## 9. Aggregation 자주 쓰는 패턴

```json
{
  "size": 0,
  "aggs": {
    "by_status": {
      "terms": { "field": "status", "size": 10 },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    },
    "by_day": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1d"
      }
    },
    "p95_latency": {
      "percentiles": { "field": "latency_ms", "percents": [50, 95, 99] }
    }
  }
}
```

`size: 0` — hits를 안 받겠다는 뜻. 집계만 필요할 때 필수 (네트워크·메모리 절약).

---

## 10. Highlighting

```json
{
  "query": { "match": { "body": "elasticsearch" } },
  "highlight": {
    "fields": {
      "body": {
        "fragment_size": 150,
        "number_of_fragments": 3,
        "pre_tags":  ["<mark>"],
        "post_tags": ["</mark>"]
      }
    }
  }
}
```

`unified` highlighter (기본)은 대부분 충분. 큰 텍스트 + `term_vector: with_positions_offsets` 매핑이면 `fvh`가 빠름.

---

## 안티 패턴 빠른 경고

| 쓰지 마라 | 대신 |
|-----------|------|
| `query_string` (사용자 입력 직접) | `simple_query_string` 또는 직접 파싱 |
| Leading wildcard `"*foo"` | `wildcard` 타입 또는 reverse keyword |
| `from + size` 깊은 페이징 | `search_after` + PIT |
| `match` on `keyword` | `term` |
| `term` on `text` | `match` |
| `script` 쿼리 (런타임) | painless보다 사전 색인된 필드 |

---

*관련 cheatsheets: mapping_field_types, analyzer_selection*
