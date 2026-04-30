# 예제 1. E-commerce 상품 검색

쇼핑몰의 한국어 상품명/설명을 검색하는 시나리오. Elasticsearch의 multi-field, Nori 분석기, function_score, 패싯 집계가 어떻게 결합되어 실제 상품 검색을 구성하는지 끝까지 따라가 본다.

대상 버전: Elasticsearch 8.x / 9.x

---

## 1. 요구사항

- 약 500만 개 상품 카탈로그
- 검색 시나리오:
  - 한국어 상품명/설명 본문 검색 (형태소 분석)
  - 동의어 (`아이폰` ↔ `iPhone`, `티비` ↔ `TV`)
  - 오타 허용 (`아이패드` 입력 시 `아이폰` 후보도 함께)
  - 자동완성 (앞글자 입력 시 후보 제시)
  - 카테고리/가격대/브랜드 필터
  - 정렬: 관련도순 (기본), 최저가, 인기순
  - 패싯: 카테고리·브랜드·가격대 카운트
- 인기 상품(`sales_count`가 높은 것)이 상위에 노출

---

## 2. Nori 플러그인 + 동의어/사용자 사전

### 2.1 플러그인 설치

```bash
# 모든 노드에서 실행 후 롤링 재시작
bin/elasticsearch-plugin install analysis-nori
```

### 2.2 사용자 사전과 동의어 사전

```
# config/userdict_ko.txt — 명사 추가 (탭/공백 구분)
아이폰
갤럭시
에어팟
프로맥스
```

```
# config/synonym_ko.txt — Solr 포맷
아이폰, iPhone, 아이폰15
티비, TV, 텔레비전
운동화, 스니커즈, sneakers
```

> 동의어 파일은 모든 데이터 노드의 동일 경로에 배포해야 한다. 파일이 없으면 인덱스 오픈 실패.

---

## 3. 인덱스 매핑 설계

### 3.1 분석기 정의

```json
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.refresh_interval": "5s",
    "analysis": {
      "tokenizer": {
        "nori_user_dict": {
          "type": "nori_tokenizer",
          "decompound_mode": "mixed",
          "user_dictionary": "userdict_ko.txt"
        },
        "edge_ngram_tk": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 20,
          "token_chars": ["letter", "digit"]
        }
      },
      "filter": {
        "ko_synonym": {
          "type": "synonym_graph",
          "synonyms_path": "synonym_ko.txt",
          "updateable": true
        },
        "ko_stop": {
          "type": "nori_part_of_speech",
          "stoptags": ["E", "IC", "J", "MAG", "MAJ", "MM", "SP", "SSC", "SSO", "SC", "SE", "XPN", "XSA", "XSN", "XSV", "UNA", "NA", "VSV"]
        }
      },
      "analyzer": {
        "ko_index": {
          "type": "custom",
          "tokenizer": "nori_user_dict",
          "filter": ["lowercase", "nori_readingform", "ko_stop"]
        },
        "ko_search": {
          "type": "custom",
          "tokenizer": "nori_user_dict",
          "filter": ["lowercase", "nori_readingform", "ko_stop", "ko_synonym"]
        },
        "autocomplete_index": {
          "type": "custom",
          "tokenizer": "edge_ngram_tk",
          "filter": ["lowercase"]
        },
        "autocomplete_search": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "product_id":   { "type": "keyword" },
      "name": {
        "type": "text",
        "analyzer": "ko_index",
        "search_analyzer": "ko_search",
        "fields": {
          "keyword":      { "type": "keyword", "ignore_above": 256 },
          "autocomplete": {
            "type": "text",
            "analyzer": "autocomplete_index",
            "search_analyzer": "autocomplete_search"
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "ko_index",
        "search_analyzer": "ko_search"
      },
      "brand":      { "type": "keyword" },
      "category_id":{ "type": "keyword" },
      "category_path": { "type": "keyword" },
      "price":      { "type": "scaled_float", "scaling_factor": 100 },
      "stock":      { "type": "integer" },
      "sales_count":{ "type": "integer" },
      "rating":     { "type": "half_float" },
      "tags":       { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}
```

### 설계 근거

| 결정 | 이유 |
|------|------|
| `index_analyzer` ≠ `search_analyzer` | 색인 시에는 동의어 미적용, 검색 시 `synonym_graph`로 확장 → 인덱스 크기 절약 + 동의어 사전 변경 시 `_reload_search_analyzers`만으로 반영 |
| `decompound_mode: mixed` | 복합명사를 원형 + 분해형 모두 색인 (`프로맥스` → `프로맥스`, `프로`, `맥스`) |
| `nori_readingform` | 한자 → 한글 변환 |
| `synonym_graph` + `updateable: true` | 동의어 핫 리로드 가능. 단 search-time에서만 동작 |
| `name.autocomplete` | 별도 멀티필드. edge_ngram은 인덱스 크기가 크므로 자동완성 전용 서브필드로 분리 |
| `name.keyword` | 정렬·집계·정확 일치 매칭용 |
| `scaled_float` | 가격은 부동소수점 오차 위험. 100배 스케일링으로 정수 저장 (`9900` → 99.00원) |

---

## 4. 색인 파이프라인

### 4.1 Bulk API (배치)

```bash
POST /products/_bulk?refresh=false
{ "index": { "_id": "P001" } }
{ "product_id": "P001", "name": "애플 아이폰 15 프로맥스 256GB", "brand": "Apple", "category_id": "C0101", "category_path": ["가전", "스마트폰"], "price": 1750000.00, "stock": 32, "sales_count": 1820, "rating": 4.8, "tags": ["신상품", "5G"], "created_at": "2024-09-22" }
{ "index": { "_id": "P002" } }
{ "product_id": "P002", "name": "삼성 갤럭시 S24 울트라 512GB", "brand": "Samsung", "category_id": "C0101", "category_path": ["가전", "스마트폰"], "price": 1690000.00, "stock": 18, "sales_count": 1500, "rating": 4.7, "tags": ["베스트셀러"], "created_at": "2024-01-30" }
```

> 대량 색인 중에는 `refresh_interval: -1`, `number_of_replicas: 0`으로 줄였다가 끝나면 복구. 처리량이 5~10배 향상된다.

### 4.2 동의어 사전 핫 리로드

```bash
# config/synonym_ko.txt 수정 후 (모든 노드)
POST /products/_reload_search_analyzers
```

---

## 5. 검색 쿼리

### 5.1 기본 키워드 검색 (관련도 + 부스팅)

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {
              "multi_match": {
                "query": "아이폰 케이스",
                "fields": ["name^3", "description"],
                "type": "best_fields",
                "fuzziness": "AUTO",
                "operator": "and"
              }
            }
          ],
          "filter": [
            { "term":  { "category_path": "스마트폰" } },
            { "range": { "price": { "gte": 10000, "lte": 2000000 } } },
            { "range": { "stock": { "gt": 0 } } }
          ]
        }
      },
      "functions": [
        {
          "field_value_factor": {
            "field": "sales_count",
            "modifier": "log1p",
            "factor": 0.3,
            "missing": 0
          }
        },
        {
          "field_value_factor": {
            "field": "rating",
            "modifier": "sqrt",
            "factor": 0.5,
            "missing": 3.5
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  },
  "size": 20
}
```

**핵심 포인트:**

- `must` (스코어링) vs `filter` (캐싱, 스코어 없음) 분리 → 가격·재고는 `filter` 컨텍스트로
- `name^3`: 상품명에 3배 가중치
- `fuzziness: AUTO`: 단어 길이에 따라 자동으로 편집거리 0~2 적용 (오타 허용)
- `operator: and`: 토큰이 모두 매칭되어야 함 (`아이폰` AND `케이스`)
- `field_value_factor` + `log1p`: `_score * (1 + log(1 + sales_count * 0.3))` → 인기도가 높을수록 boost, 그러나 차이가 무한히 벌어지지 않음

### 5.2 자동완성

```json
GET /products/_search
{
  "query": {
    "match": {
      "name.autocomplete": {
        "query": "아이",
        "operator": "and"
      }
    }
  },
  "_source": ["product_id", "name"],
  "size": 8
}
```

### 5.3 패싯 (카테고리/브랜드/가격대)

```json
GET /products/_search
{
  "size": 0,
  "query": {
    "match": { "name": "노트북" }
  },
  "aggs": {
    "by_brand": {
      "terms": { "field": "brand", "size": 10 }
    },
    "by_category": {
      "terms": { "field": "category_path", "size": 20 }
    },
    "price_buckets": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 500000, "key": "~50만" },
          { "from": 500000, "to": 1000000, "key": "50~100만" },
          { "from": 1000000, "to": 2000000, "key": "100~200만" },
          { "from": 2000000, "key": "200만~" }
        ]
      }
    },
    "avg_rating": { "avg": { "field": "rating" } }
  }
}
```

### 5.4 정렬: 인기순/최저가

```json
GET /products/_search
{
  "query": {
    "bool": {
      "filter": [{ "term": { "category_path": "노트북" } }]
    }
  },
  "sort": [
    { "sales_count": "desc" },
    { "price": "asc" }
  ]
}
```

> 정렬은 `keyword`/numeric 필드에서만. `text` 필드 정렬은 `fielddata: true`가 필요하고 메모리 폭증 위험 (→ `troubleshooting/B3` 참고).

---

## 6. 안티패턴 피하기

### 잘못된 예 1: text 필드를 그대로 정렬·집계

```json
// 금지
"sort": [ { "name": "asc" } ]
// → fielddata error 또는 느림. name.keyword를 써야 함
```

### 잘못된 예 2: 가격 필터를 must에 넣기

```json
// 좋지 않음 (스코어 영향)
"must": [{ "range": { "price": { "lte": 50000 } } }]

// 좋음 (캐시되고 스코어 영향 없음)
"filter": [{ "range": { "price": { "lte": 50000 } } }]
```

### 잘못된 예 3: 모든 필드에 ngram 적용

```
모든 필드를 ngram으로 색인하면 인덱스 크기가 5~10배 증가하고
색인 속도가 떨어진다. 자동완성이 필요한 필드 1~2개에만 적용.
```

---

## 7. 운영 체크리스트

```bash
# 1) 분석기가 의도대로 동작하는가
GET /products/_analyze
{
  "analyzer": "ko_search",
  "text": "아이폰 프로맥스"
}

# 2) 동의어 핫 리로드 후 확인
POST /products/_reload_search_analyzers

# 3) 인덱스 크기/세그먼트
GET /_cat/indices/products?v&h=index,docs.count,store.size,pri.store.size
GET /_cat/segments/products?v

# 4) 느린 쿼리 추적 — slow log 활성화
PUT /products/_settings
{
  "index.search.slowlog.threshold.query.warn": "1s",
  "index.search.slowlog.threshold.query.info": "500ms"
}

# 5) profile API로 단일 쿼리 분석
GET /products/_search
{
  "profile": true,
  "query": { "match": { "name": "아이폰" } }
}
```

---

## 8. 요약

| 기능 | 결정 |
|------|------|
| 한국어 분석 | Nori + 사용자 사전 + `nori_part_of_speech` 불용 품사 제거 |
| 동의어 | `synonym_graph` + `updateable: true` (search-time) |
| 오타 허용 | `multi_match` + `fuzziness: AUTO` |
| 자동완성 | `edge_ngram` 멀티필드 (`name.autocomplete`) |
| 정렬·집계 | `keyword` 멀티필드 (`name.keyword`) |
| 인기 부스팅 | `function_score` + `field_value_factor(log1p)` |
| 필터 분리 | 가격·재고·카테고리는 `filter` 컨텍스트로 |
