# 7장. Query DSL — bool, 점수, 페이지네이션, Hybrid Search

Elasticsearch의 검색은 **Query DSL**이라는 JSON 기반 쿼리 언어로 표현된다. 단순한 SQL과 달리 Query DSL은 **점수 계산**, **컨텍스트(쿼리/필터)**, **컴파운드 트리** 같은 검색 특유의 개념을 명시적으로 다룬다. 이 장은 Query DSL의 정신적 모델과 운영 패턴을 다룬다.

공식 문서: [Query DSL](https://www.elastic.co/docs/reference/query-languages/query-dsl)

---

## 7.1 Query Context vs Filter Context

Query DSL의 가장 근본적인 구분이다. 같은 쿼리도 **어느 컨텍스트**에 들어가느냐에 따라 동작이 달라진다.

| | Query Context | Filter Context |
|---|---|---|
| **점수 계산** | O — `_score` 산출 (BM25 등) | X — 매칭 여부만 |
| **캐시** | 캐시 안 됨 | **node-level filter cache** 적용 |
| **사용 위치** | `query`, `bool.must`, `bool.should` | `bool.filter`, `bool.must_not`, `constant_score.filter` |
| **용도** | "얼마나 잘 맞는가" | "예/아니오" |

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must":   [ { "match": { "name": "노트북" } } ],   ← Query context (점수 계산)
      "filter": [
        { "term":  { "in_stock": true } },              ← Filter context (캐시)
        { "range": { "price": { "lte": 2000000 } } }    ← Filter context (캐시)
      ]
    }
  }
}
```

```
ASCII: Query vs Filter Context
─────────────────────────────────────────────────────────
  Query Context (점수 ON):
    ┌──────────────┐    ┌──────────────────────┐
    │ match 노트북 │ ──▶│ BM25 점수 매기기     │ ──▶ _score=4.21
    │              │    │ TF·IDF·필드 길이 등  │
    └──────────────┘    └──────────────────────┘
        매번 계산 (캐시 X)

  Filter Context (점수 OFF):
    ┌──────────────┐    ┌──────────────────────┐
    │ in_stock=true│ ──▶│ 비트셋 매칭          │ ──▶ 매칭 O/X
    │              │    │ (BitSet)             │     점수 영향 X
    └──────────────┘    └──────────────────────┘
        세그먼트 단위 캐시 (LRU)
─────────────────────────────────────────────────────────
```

### 운영 원칙

> **점수에 영향을 주지 않는 모든 조건은 filter에 넣어라.**

이 한 줄이 ES 운영의 80%다. `bool.must`에 부질없이 `term` 조건을 쌓으면 점수도 망치고 캐시도 못 받는다.

공식 문서: [Query and filter context](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-filter-context)

---

## 7.2 Leaf Queries — 단독으로 동작하는 쿼리

### Term-level (정확 매칭, 분석 X)

```json
{ "term":   { "status": "shipped" } }              // 단일 값 정확
{ "terms":  { "status": ["shipped", "delivered"] } } // OR
{ "range":  { "price": { "gte": 1000, "lte": 5000 } } }
{ "exists": { "field": "promo_code" } }            // 필드 존재
{ "prefix": { "sku": "AB-" } }                     // 접두사
{ "wildcard": { "email": "*@example.com" } }       // 와일드카드 (느림)
{ "regexp":   { "username": "j[a-z]+" } }          // 정규식 (느림)
{ "ids":      { "values": ["1", "2", "3"] } }      // _id 직접
```

**주의**: `term`은 **분석되지 않는다**. text 필드에 `term`을 쓰면 거의 항상 매칭이 안 된다 (역색인엔 토큰화된 값이 들어 있으니까).

```json
// ❌ name이 text 필드면 거의 매칭 안 됨
{ "term": { "name": "Quick Brown Fox" } }

// ✅ keyword 서브필드를 쓰거나
{ "term": { "name.keyword": "Quick Brown Fox" } }

// ✅ match를 사용
{ "match": { "name": "Quick Brown Fox" } }
```

### Full-text (분석 O)

```json
{ "match":         { "title": "검색엔진 노트북" } }     // OR (기본)
{ "match":         { "title": { "query": "검색엔진 노트북", "operator": "AND" } } }
{ "match_phrase":  { "title": "검색 엔진" } }           // 구문 (순서·인접)
{ "match_phrase_prefix": { "title": "검색 엔" } }       // 마지막 단어 prefix
{ "multi_match":   {
    "query": "검색엔진",
    "fields": ["title^3", "body", "tags"]            // ^3 = boost
} }
{ "query_string":  {                                   // Lucene 문법 (사용자 입력에 위험)
    "query": "title:(검색 OR engine) AND status:active"
} }
{ "simple_query_string": {                             // query_string의 안전 버전
    "query": "+검색 +엔진 -광고",
    "fields": ["title", "body"]
} }
```

### multi_match의 type 옵션

같은 쿼리어를 여러 필드에 던질 때 **점수를 어떻게 합산할지** 결정.

| type | 동작 |
|------|------|
| `best_fields` (기본) | 최고 점수 필드의 점수 사용 (대부분의 경우) |
| `most_fields` | 모든 필드 점수 합산 (multi-fields 동일 분석기 다른 변형) |
| `cross_fields` | 여러 필드를 하나의 큰 필드처럼 처리 (이름+성 등) |
| `phrase` | 각 필드에 match_phrase 적용 |
| `bool_prefix` | match_bool_prefix 적용 (자동완성성) |

```json
{ "multi_match": {
    "query": "검색엔진",
    "type": "best_fields",
    "fields": ["title^3", "title.korean^2", "body"],
    "tie_breaker": 0.3
} }
```

`tie_breaker`는 best_fields에서 차상위 필드 점수를 일부 반영한다.

---

## 7.3 Compound Queries — 쿼리를 조합하기

### bool — 가장 중요한 컴파운드 쿼리

4개의 절을 조합한다.

| 절 | 점수 | 매칭 필요 | 용도 |
|------|------|------|------|
| `must` | O | 모두 매칭 | 필수, 점수 계산 |
| `should` | O | (기본) 0개 이상, `minimum_should_match`로 강제 가능 | 선택 OR, 점수 가산 |
| `filter` | X | 모두 매칭 | 필터 (점수 X, 캐시 O) |
| `must_not` | X | 어느 것도 매칭 X | 제외 (점수 X) |

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "무선 노트북" } }
      ],
      "should": [
        { "match": { "tags": "프리미엄" } },
        { "match": { "brand": "삼성" } }
      ],
      "filter": [
        { "term":  { "category": "laptop" } },
        { "range": { "price": { "lte": 2000000 } } },
        { "term":  { "in_stock": true } }
      ],
      "must_not": [
        { "term": { "discontinued": true } }
      ],
      "minimum_should_match": 1
    }
  }
}
```

**해석**:
- 이름에 "무선 노트북"이 매칭되어야 하고 (필수, 점수 영향)
- 태그가 "프리미엄"이거나 브랜드가 "삼성"이면 점수 가산 (1개 이상 일치 필수)
- 카테고리·가격·재고 조건은 필수 (점수 X, 캐시 O)
- 단종된 상품은 제외

### dis_max — 최고 점수 채택

```json
{ "dis_max": {
    "queries": [
      { "match": { "title": "검색엔진" } },
      { "match": { "body":  "검색엔진" } }
    ],
    "tie_breaker": 0.3
} }
```

여러 쿼리 중 **최고 점수**를 채택. `bool.should`가 합산하는 것과 다르다. multi_match `best_fields`의 내부 구현이기도 하다.

### function_score — 점수 커스터마이징

```json
{
  "function_score": {
    "query": { "match": { "title": "노트북" } },
    "functions": [
      { "field_value_factor": {
          "field": "popularity", "modifier": "log1p", "factor": 1.2
      } },
      { "gauss": {
          "created_at": {
            "origin": "now",
            "scale":  "30d",
            "decay":  0.5
          }
      } }
    ],
    "score_mode": "sum",   // 함수들끼리 합치는 방법
    "boost_mode": "multiply" // 원래 query 점수와 합치는 방법
  }
}
```

전형적 사용: **인기도 가산**, **최신성 감쇠**, **사용자 위치 거리 감쇠**.

### boosting — Negative boost

```json
{ "boosting": {
    "positive": { "match": { "name": "노트북" } },
    "negative": { "term":  { "brand": "discontinued_brand" } },
    "negative_boost": 0.2
} }
```

매칭은 시키되 점수를 깎는다. `must_not`(완전 제외)과 다르다.

### constant_score — 점수 고정

```json
{ "constant_score": {
    "filter": { "term": { "category": "laptop" } },
    "boost": 1.0
} }
```

filter 결과를 query처럼 쓰되, 점수를 1.0으로 고정. 매우 흔하게 사용.

---

## 7.4 점수 계산 — BM25

ES 5.x부터 기본 알고리즘은 **BM25 (Okapi BM25)**.

### 직관적 공식

```
score(D, Q) = Σ over t in Q  IDF(t) × ─────────────────────────────
                                       TF(t,D) × (k1+1)
                                       ───────────────────────────
                                       TF(t,D) + k1 × (1 - b + b × |D|/avgDL)
```

핵심 변수:
- **TF (term frequency)**: 문서 내 해당 토큰 출현 횟수. 많을수록 점수↑ (단, **포화**)
- **IDF (inverse document frequency)**: `log((N-df+0.5)/(df+0.5))`. 희귀 토큰일수록 점수↑
- **|D| / avgDL**: 문서 길이 / 평균 문서 길이. 긴 문서는 점수 보정 (`b`로 강도 조절)
- **k1** (기본 1.2): TF 포화 정도
- **b** (기본 0.75): 길이 정규화 강도

### `explain`으로 점수 내역 보기

```json
GET /products/_search
{
  "explain": true,
  "query": { "match": { "name": "노트북" } }
}
```

응답의 `_explanation`에 BM25 각 항의 값이 트리로 펼쳐진다. 점수가 의외일 때 첫 번째 디버깅 도구.

### 흔한 점수 함정

- **상위 결과가 짧은 spam 같은 문서**: TF/길이 정규화가 짧은 문서를 선호 → length boost나 function_score로 보정
- **희귀어가 너무 강하게 작용**: 오타나 고유명사가 IDF로 과대평가 → multi-fields와 분석기 조정으로 완화
- **filter에 넣었는데 점수에 영향 가는 것 같다**: filter context는 점수 0. 하지만 should를 같이 쓰면 점수에 들어감 — 의도 확인

---

## 7.5 정렬, search_after, from/size

### 기본: 점수 내림차순

`sort`를 지정하지 않으면 `_score` desc.

### 명시적 정렬

```json
{
  "sort": [
    { "created_at": { "order": "desc" } },
    { "_score":     { "order": "desc" } },
    { "_id":        { "order": "asc" } }    // 안정 정렬 위한 tiebreaker
  ]
}
```

text 필드는 정렬 불가 (doc_values 없음). keyword 또는 multi-fields의 keyword 서브필드 사용.

### from/size — 얕은 페이지네이션

```json
{ "from": 20, "size": 10 }
```

기본 한계: `index.max_result_window` (기본 **10,000**). 그 이상 깊이는 거부됨. 이는 안전장치다.

> **왜 한계가 있나**: from + size까지의 모든 hit을 모든 shard에서 모아 메모리에 로드 후 정렬해야 함. 깊은 페이지일수록 메모리 폭발.

```
ASCII: from/size의 비용 구조
─────────────────────────────────────────────────────
  from=10000, size=10 (페이지 1001)

  shard A: 상위 10010개 hit 메모리에 로드, 정렬
  shard B: 상위 10010개 hit 메모리에 로드, 정렬
  shard C: 상위 10010개 hit 메모리에 로드, 정렬
       ↓ coordinating node로 모두 전송
  coordinating: 30030개를 다시 정렬 후 [10000:10010] 반환
       ↓
  실제 사용은 10개. 30020개는 버려짐.
─────────────────────────────────────────────────────
```

---

## 7.6 search_after + PIT — 깊은 페이지네이션

### 표준 패턴 (8.x)

```json
// 1) PIT(point-in-time) 열기
POST /products/_pit?keep_alive=1m
// → { "id": "46To...long-pit-id" }

// 2) 첫 페이지
POST /_search
{
  "size": 100,
  "pit":  { "id": "46To...long-pit-id", "keep_alive": "1m" },
  "sort": [
    { "created_at": "desc" },
    { "_shard_doc": "asc" }      // 8.x 권장 tiebreaker
  ]
}
// 응답의 마지막 hit.sort = [1714512345000, 42]

// 3) 다음 페이지 — search_after로 이어서
POST /_search
{
  "size": 100,
  "pit":  { "id": "46To...long-pit-id", "keep_alive": "1m" },
  "sort": [
    { "created_at": "desc" },
    { "_shard_doc": "asc" }
  ],
  "search_after": [1714512345000, 42]
}

// 4) 종료 시 PIT 닫기
DELETE /_pit
{ "id": "46To...long-pit-id" }
```

### 왜 PIT이 필요한가

PIT 없이 `search_after`만 쓰면 페이지 사이에 새 문서가 색인되거나 삭제되어 **결과가 일관되지 않을 수 있다**. PIT는 시작 시점의 인덱스 스냅샷을 보장한다.

### scroll은 deprecated

`/_search/scroll`은 여전히 동작하지만, 새 코드에서는 **권장되지 않는다**. PIT + search_after가 표준.

> "If you need to preserve the index state while paging through more than 10,000 hits, use the search_after parameter with a point in time (PIT)."  
> — [Paginate search results](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/paginate-search-results)

---

## 7.7 Highlight, Suggest

### Highlight — 검색어 강조

```json
GET /articles/_search
{
  "query": { "match": { "body": "검색엔진" } },
  "highlight": {
    "fields": {
      "body": {
        "fragment_size":   150,
        "number_of_fragments": 3,
        "pre_tags":  ["<em>"],
        "post_tags": ["</em>"]
      }
    }
  }
}
```

3가지 highlighter:
- `unified` (기본, 권장)
- `plain` (단순)
- `fvh` (fast vector highlighter — `term_vector: with_positions_offsets` 필요)

### Suggester — 제안

| 종류 | 용도 |
|------|------|
| `term` | 토큰 단위 오타 교정 (single-word) |
| `phrase` | 구 단위 오타 교정 |
| `completion` | 자동완성 (FST 기반, 매우 빠름) |

```json
// completion 매핑
"name_suggest": { "type": "completion" }

// 자동완성 쿼리
GET /products/_search
{
  "suggest": {
    "product-suggest": {
      "prefix": "노트",
      "completion": {
        "field": "name_suggest",
        "size":  10,
        "skip_duplicates": true
      }
    }
  }
}
```

`completion`은 별도 자료구조(FST)로 매우 빠르지만, 색인 시점에 후보를 미리 등록해야 한다.

---

## 7.8 Retriever, RRF, Hybrid Search (8.x/9.x)

8.16 GA된 **retriever** 프레임워크는 검색 파이프라인을 명시적으로 표현한다. kNN(벡터)과 BM25를 함께 쓰는 hybrid search의 표준 방법.

### Retriever 기본 형태

```json
GET /docs/_search
{
  "retriever": {
    "standard": {
      "query": { "match": { "body": "검색엔진 아키텍처" } }
    }
  }
}
```

### kNN retriever

```json
{
  "retriever": {
    "knn": {
      "field": "embedding",
      "query_vector": [0.12, -0.04, ...],
      "k": 50,
      "num_candidates": 200
    }
  }
}
```

### RRF (Reciprocal Rank Fusion) — Hybrid

```json
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": { "match": { "body": "검색엔진 아키텍처" } }
          }
        },
        {
          "knn": {
            "field": "embedding",
            "query_vector": [0.12, -0.04, ...],
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

각 retriever에서 상위 N개를 받아 **rank 기반**으로 재합산:

```
RRF score(d) = Σ over retrievers  1 / (rank_constant + rank(d))
```

점수 스케일이 다른 BM25와 코사인 유사도를 안전하게 결합하는 표준 기법.

### semantic_text (9.x)

9.x에서는 `semantic_text` 필드 타입이 자동으로 inference 엔드포인트(ELSER 등)를 호출해 임베딩을 만들고 검색해준다.

```json
"mappings": {
  "properties": {
    "content": { "type": "semantic_text", "inference_id": "my-elser-endpoint" }
  }
}

// 검색
{
  "retriever": {
    "standard": {
      "query": {
        "semantic": { "field": "content", "query": "검색엔진의 BM25 알고리즘" }
      }
    }
  }
}
```

벡터 차원, similarity 같은 저수준 설정을 모두 추상화한다.

공식 문서: [Retrievers](https://www.elastic.co/docs/solutions/search/retrievers-overview), [semantic_text](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/semantic-text)

---

## 7.9 실전 패턴: 운영급 bool 쿼리

```json
GET /products/_search
{
  "size": 20,
  "track_total_hits": false,
  "_source": ["id", "name", "price", "thumbnail"],
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {
              "multi_match": {
                "query":  "{user_query}",
                "type":   "best_fields",
                "fields": ["name^3", "name.korean^2", "tags^2", "body"],
                "tie_breaker": 0.3,
                "operator": "and"
              }
            }
          ],
          "filter": [
            { "term":  { "status": "published" } },
            { "term":  { "in_stock": true } },
            { "range": { "price": { "gte": 0, "lte": 5000000 } } },
            { "terms": { "category": ["laptop", "desktop"] } }
          ],
          "must_not": [
            { "term": { "is_adult": true } }
          ],
          "should": [
            { "term": { "tags": { "value": "추천", "boost": 0.5 } } }
          ]
        }
      },
      "functions": [
        { "field_value_factor": { "field": "popularity", "modifier": "log1p", "missing": 1 } },
        { "gauss": { "created_at": { "origin": "now", "scale": "30d", "decay": 0.5 } } }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  },
  "sort": ["_score", { "_id": "asc" }]
}
```

**여기서 일어나는 일**:
- 사용자 입력 → `multi_match`로 여러 필드 동시 검색 (`best_fields`)
- 모든 필수 조건은 `filter`로 → 점수 영향 X, 캐시 O
- 부적절 콘텐츠는 `must_not`
- "추천" 태그는 약한 boost (`should`)
- 인기도 + 최신성으로 점수 보정
- `track_total_hits: false`로 정확한 총 개수 계산 생략 (성능)
- `_source`로 응답 필드 제한 (네트워크 절약)

---

## 7.10 운영 시 주의점

### track_total_hits

기본적으로 ES는 정확한 총 일치 개수를 10,000까지만 센다. 그 이상은 `gte` 표시.

```json
// 정확한 총 개수가 필요하면 (느림)
{ "track_total_hits": true, ... }

// 페이지네이션만 필요하고 총 개수 표시 안 해도 됨 (빠름)
{ "track_total_hits": false, ... }
```

### profile API

쿼리 비용 분석.

```json
{ "profile": true, "query": { ... } }
```

각 shard에서의 Lucene 단계별 시간을 보여준다. 실제로 무엇이 느린지 진단할 때 사용.

### query_string은 사용자 입력에 노출 금지

`query_string`은 강력하지만 Lucene 문법 오류 시 전체 쿼리 실패. 사용자 검색창에는 **`simple_query_string`**을 사용하라 — 잘못된 문법은 무시한다.

### wildcard, regexp는 비싸다

특히 leading wildcard (`*foo`)는 매우 느리다. 자동완성이 목적이면 `edge_ngram`이나 `completion suggester` 사용.

### 깊은 페이지네이션 = PIT + search_after

`from > 10000`은 거부되도록 그대로 두고, 큰 페이지가 필요한 곳은 PIT/search_after로 변경. UI 측에서 "X페이지 이동"보다 "더 보기" 패턴 권장.

---

## 7.11 Key Takeaways

| 항목 | 핵심 |
|------|------|
| **Query vs Filter context** | 점수에 영향 없는 모든 조건은 filter로 |
| **bool** | must/should/filter/must_not 조합. 가장 자주 쓰는 컴파운드 |
| **term ≠ match** | term은 분석 안 함. text 필드엔 거의 항상 match 사용 |
| **multi_match best_fields** | 가장 흔한 multi-field 검색 패턴. tie_breaker로 보조 |
| **BM25** | TF, IDF, 문서 길이의 함수. explain으로 디버깅 |
| **from/size 한계** | 10,000 초과 거부. PIT + search_after로 |
| **PIT + search_after + _shard_doc** | 8.x 깊은 페이지네이션 표준 |
| **retriever + RRF** | 8.x hybrid search 표준. BM25 + kNN 결합 |
| **semantic_text** | 9.x. inference 호출까지 추상화 |

---

*다음 장: 8장. Aggregations — 집계의 종류와 정확도*
