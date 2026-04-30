# 5장. Mapping과 Field Types — 스키마 설계의 모든 것

Elasticsearch는 "schemaless"를 표방하지만, 실제로는 모든 인덱스가 **mapping**이라는 스키마를 가진다. mapping이 잘못되면 정렬/집계가 안 되거나, 디스크가 폭발하거나, 재색인 외에는 복구할 방법이 없다. 이 장은 mapping의 동작 원리와 설계 원칙을 다룬다.

공식 문서: [Mapping](https://www.elastic.co/docs/manage-data/data-store/mapping)

---

## 5.1 Mapping이란

Mapping은 인덱스 내 문서가 어떤 **필드**로 구성되고, 각 필드가 어떻게 **저장·색인·검색**되는지를 정의하는 메타데이터다. RDBMS의 DDL과 유사하지만, 몇 가지 결정적인 차이가 있다.

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name":        { "type": "text" },
      "sku":         { "type": "keyword" },
      "price":       { "type": "scaled_float", "scaling_factor": 100 },
      "stock":       { "type": "integer" },
      "created_at":  { "type": "date" },
      "tags":        { "type": "keyword" },
      "location":    { "type": "geo_point" }
    }
  }
}
```

Mapping이 결정하는 것:

- **저장 형식**: 역색인(inverted index)에 들어갈지, doc_values(컬럼 스토어)에 들어갈지, _source에만 둘지
- **분석기**: text 필드라면 어떤 analyzer를 거쳐 토큰화될지
- **검색 가능성**: term 쿼리, range 쿼리, full-text 쿼리 가능 여부
- **정렬·집계 가능성**: doc_values 활성화 여부에 좌우
- **디스크/메모리 사용량**: 타입에 따라 천차만별

> "Mapping is the process of defining how a document, and the fields it contains, are stored and indexed." — [Elastic Docs](https://www.elastic.co/docs/manage-data/data-store/mapping)

---

## 5.2 Dynamic vs Explicit Mapping

새 필드가 들어왔을 때 어떻게 처리할지 결정하는 것이 `dynamic` 파라미터다. 4가지 값이 있다.

| 값 | 동작 | 사용 시나리오 |
|----|------|---------------|
| `true` (기본) | 새 필드를 자동 추론하여 mapping에 추가 | 프로토타이핑, 로그 |
| `false` | 무시. 색인은 되나 검색·집계 불가 (`_source`만 보존) | 자유 형식 메타데이터 |
| `strict` | 새 필드 발견 시 **문서 거부**(예외) | 운영 인덱스, 스키마 엄격성 필요 |
| `runtime` | 새 필드를 runtime field로 추가 (역색인 X, 쿼리 시 평가) | 변동 잦은 로그 스키마 |

```json
PUT /strict_index
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "user_id":  { "type": "keyword" },
      "event":    { "type": "keyword" }
    }
  }
}

POST /strict_index/_doc
{ "user_id": "u-1", "event": "login", "extra": "oops" }
// → 400: mapping set to strict, dynamic introduction of [extra] within [_doc] is not allowed
```

**왜 strict가 권장되는가**: dynamic=true에서는 오타 한 번이 새 필드를 만들고(`uesr_id`), 그 필드는 영원히 mapping에 남는다. mapping은 1000개 필드 한계가 있고(`index.mapping.total_fields.limit`), 한 번 잘못 추론된 타입(예: 첫 값이 `"42"`라 text가 되었는데 사실은 숫자)은 되돌릴 수 없다.

공식 문서: [Dynamic field mapping](https://www.elastic.co/docs/manage-data/data-store/mapping/dynamic-field-mapping)

---

## 5.3 핵심 Field Type

### 문자열: text vs keyword

ES에는 "string" 타입이 없다. 문자열은 **두 개의 다른 타입**이다.

| 타입 | 동작 | 용도 |
|------|------|------|
| `text` | analyzer로 토큰화 후 역색인 저장. doc_values 없음 (집계/정렬 비효율) | 본문, 제목 같은 full-text 검색 대상 |
| `keyword` | 분석 없이 **원본 문자열 전체**를 단일 토큰으로 색인. doc_values 기본 활성 | 정확 매칭, 정렬, 집계, term 쿼리 |

### 숫자

| 타입 | 범위 | 비고 |
|------|------|------|
| `byte` | -128 ~ 127 | |
| `short` | -32,768 ~ 32,767 | |
| `integer` | -2^31 ~ 2^31-1 | 일반적인 정수 |
| `long` | -2^63 ~ 2^63-1 | 큰 ID, 타임스탬프 |
| `unsigned_long` | 0 ~ 2^64-1 | 부호 없는 큰 정수 |
| `float` | 32-bit IEEE 754 | |
| `double` | 64-bit IEEE 754 | 정밀도 우선 |
| `half_float` | 16-bit IEEE 754 | 정밀도가 중요하지 않은 메트릭 |
| `scaled_float` | long을 `scaling_factor`로 나눠 표현 | **금액에 권장** (정확한 소수점 2자리) |

```json
"price": { "type": "scaled_float", "scaling_factor": 100 }
// 12.34 → 내부 1234 long → 정확한 소수점 보존, double보다 작은 디스크/메모리
```

### 날짜

```json
"event_time": {
  "type": "date",
  "format": "strict_date_optional_time||epoch_millis"
}
```

내부 저장은 항상 **epoch milliseconds (long)**. `format`은 입력 파싱과 출력 포맷에만 영향. ES의 기본 포맷은 `strict_date_optional_time||epoch_millis` — ISO 8601 (`2025-04-30T12:34:56.789Z`) 또는 epoch ms 정수.

`date_nanos`는 나노초 정밀도가 필요할 때(고빈도 트레이딩 로그 등). 단 메모리·디스크 비용 증가.

### Boolean

```json
"is_active": { "type": "boolean" }
// 입력: true / false / "true" / "false"
```

### IP, geo_point, geo_shape

```json
"client_ip":  { "type": "ip" },
"location":   { "type": "geo_point" },
"region":     { "type": "geo_shape" }
```

- `ip`: IPv4/IPv6 모두 지원. CIDR 쿼리 가능.
- `geo_point`: 위·경도 한 점. 거리/경계 박스 쿼리.
- `geo_shape`: 다각형/선/원 등 복합 도형. 비싸다.

### Vector — 8.x/9.x 핵심

```json
"embedding": {
  "type": "dense_vector",
  "dims": 768,
  "index": true,
  "similarity": "cosine"
}
```

- `dense_vector`: 고정 차원의 부동소수점 벡터. kNN 검색용.
- `sparse_vector`: ELSER 같은 학습된 토큰-가중치 모델용.
- 9.x에서는 `semantic_text` 타입이 자동으로 inference 엔드포인트를 호출해 벡터를 생성한다.

공식 문서: [dense_vector](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector), [semantic_text](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/semantic-text)

---

## 5.4 text vs keyword — 가장 흔한 실수

이것이 ES 신규 사용자가 가장 많이 잘못하는 지점이다.

### 잘못 사용한 결과

```json
PUT /orders
{
  "mappings": {
    "properties": {
      "status": { "type": "text" }    // ← 잘못됨
    }
  }
}

POST /orders/_doc
{ "status": "shipped" }

// 의도: status별로 그룹핑
GET /orders/_search
{
  "size": 0,
  "aggs": { "by_status": { "terms": { "field": "status" } } }
}
// → 400: Text fields are not optimised for operations that require
//        per-document field data like aggregations and sorting,
//        so these operations are disabled by default. Please use a keyword field instead.
```

### 왜 이런 일이 일어나나

```
text 필드의 처리:
  "shipped" → analyzer → ["shipped"]
                       ↓
                    역색인 (검색용)
                       ↓
                  doc_values 없음 → 집계·정렬 불가

keyword 필드의 처리:
  "shipped" → 그대로 → "shipped" (단일 토큰)
                       ↓
                    역색인 + doc_values
                       ↓
                  검색·정렬·집계 모두 가능
```

```
ASCII: 잘못된 text 사용의 영향
─────────────────────────────────────────────────────────
  "Shipped" 와 "shipped" 의 운명
─────────────────────────────────────────────────────────
  text + standard analyzer
    "Shipped"  ──[lowercase]──▶  "shipped"
    "shipped"  ──[lowercase]──▶  "shipped"
       └─ 검색 시 둘 다 매칭됨 (의도일 수도, 아닐 수도)
       └─ 정확히 "Shipped"인 행을 못 찾음
       └─ 정렬/집계 ❌
       └─ Sort by status: 동작 안 함

  keyword
    "Shipped"  ──▶  "Shipped"
    "shipped"  ──▶  "shipped"
       └─ 별개 값으로 취급 → terms aggregation에서 분리
       └─ 정확 매칭 가능
       └─ 정렬·집계 ✅
─────────────────────────────────────────────────────────
```

### 일반 원칙

- **사람이 자연어로 검색하는 필드** → `text`
- **시스템 식별자, enum, 태그, 정확 매칭/집계 대상** → `keyword`
- **둘 다 필요** → multi-fields

---

## 5.5 Multi-fields — 한 필드를 여러 방식으로

같은 데이터를 여러 분석 방식으로 색인할 수 있다. 가장 흔한 패턴:

```json
PUT /articles
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 },
          "korean":  { "type": "text", "analyzer": "nori" }
        }
      }
    }
  }
}
```

이 mapping에서 `title`은 사실 **3개의 색인**을 갖는다:

- `title` → standard analyzer로 분석된 text (검색)
- `title.keyword` → 원본 문자열 그대로 (정렬, 집계, 정확 매칭)
- `title.korean` → Nori로 분석된 한국어 형태소 (한국어 검색)

```json
// 검색
GET /articles/_search
{ "query": { "match": { "title.korean": "검색엔진" } } }

// 집계 (keyword 사용)
GET /articles/_search
{
  "size": 0,
  "aggs": { "top_titles": { "terms": { "field": "title.keyword" } } }
}
```

`ignore_above: 256`은 256자 초과 문자열은 keyword 색인에서 제외한다. 디스크 폭발 방지의 안전장치.

> dynamic mapping이 자동으로 만드는 string 필드는 기본적으로 `text` + `fields.keyword` (`ignore_above: 256`) 조합이다.

---

## 5.6 Object vs Nested vs Flattened

JSON 구조를 갖는 필드를 다룰 때 세 가지 선택지가 있다. 잘못 고르면 **검색 결과가 틀린다**.

### Object (기본): 평탄화의 함정

```json
PUT /orders
{ "mappings": { "properties": { "items": { "type": "object" } } } }

POST /orders/_doc
{
  "items": [
    { "sku": "A1", "qty": 5 },
    { "sku": "B2", "qty": 1 }
  ]
}
```

내부적으로 ES는 다음과 같이 **평탄화**해 저장한다:

```
items.sku: ["A1", "B2"]
items.qty: [5, 1]
```

→ `sku=A1 AND qty=1`로 검색하면 매칭된다(잘못!). 둘이 같은 객체에 속한다는 정보가 사라졌다.

### Nested: 객체 단위 격리

```json
"items": {
  "type": "nested",
  "properties": {
    "sku": { "type": "keyword" },
    "qty": { "type": "integer" }
  }
}
```

각 객체가 **숨겨진 별도 문서**로 색인된다. `nested` 쿼리로만 검색 가능.

```json
GET /orders/_search
{
  "query": {
    "nested": {
      "path": "items",
      "query": {
        "bool": {
          "must": [
            { "term": { "items.sku": "A1" } },
            { "term": { "items.qty": 1 } }
          ]
        }
      }
    }
  }
}
```

**비용**: 문서 1개 = 부모 + N개 nested 자식 → 색인 크기 증가, 쿼리 비용 증가. `index.mapping.nested_objects.limit` (기본 10,000) 한계 있음.

### Flattened: 동적 키, 단일 필드

```json
"labels": { "type": "flattened" }

POST /logs/_doc
{ "labels": { "env": "prod", "team": "search", "version": "1.2.3" } }
```

전체 객체가 **하나의 필드**로 취급된다. 키별 mapping 폭증을 막는다. 단점: 모든 값이 keyword로 취급, 분석 불가, range 쿼리 제한적.

### 선택 가이드

```
구조화된 객체이고 하위 필드 정확 검색이 중요? → object (단, 배열에 주의)
배열 객체에서 같은 항목 내 AND 조건이 필요? → nested
키가 동적이고 매핑 폭증을 피하고 싶음?      → flattened
```

공식 문서: [Nested field type](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/nested), [Flattened](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/flattened)

---

## 5.7 Runtime Fields — Schema-on-Read

8.x에서 강화된 기능. 색인 시점이 아닌 **쿼리 시점**에 값을 계산한다.

```json
PUT /logs/_mapping
{
  "runtime": {
    "response_seconds": {
      "type": "double",
      "script": {
        "source": "emit(doc['response_ms'].value / 1000.0)"
      }
    }
  }
}
```

- **장점**: 재색인 없이 새 필드 추가. 잘못된 mapping 임시 우회.
- **단점**: 쿼리마다 계산 → CPU 비용. 집계·정렬·필터 시 큰 비용.

운영 패턴: **자주 쓰는 runtime field는 결국 진짜 필드로 승격**(reindex). 탐색·시연용으로 시작해 검증되면 정식 mapping에 추가.

공식 문서: [Runtime fields](https://www.elastic.co/docs/manage-data/data-store/mapping/runtime-fields)

---

## 5.8 Index Templates와 Component Templates

운영에서 인덱스를 일일이 PUT으로 만들지 않는다. **템플릿**으로 자동 적용한다.

```json
// 1) Component template — 재사용 가능한 mapping/settings 조각
PUT /_component_template/logs-mappings
{
  "template": {
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "@timestamp": { "type": "date" },
        "level":      { "type": "keyword" },
        "message":    { "type": "text" }
      }
    }
  }
}

PUT /_component_template/logs-settings
{
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy"
    }
  }
}

// 2) Index template — 패턴 매칭으로 적용
PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "composed_of": ["logs-mappings", "logs-settings"],
  "priority": 100,
  "data_stream": {}
}
```

이제 `logs-2025-04-30` 같은 인덱스를 만들면 자동으로 mapping/settings가 적용된다. **데이터 스트림**(rollover로 시계열 관리)과 함께 거의 표준 패턴.

공식 문서: [Index templates](https://www.elastic.co/docs/manage-data/data-store/templates)

---

## 5.9 Mapping은 변경 불가 — Re-index 전략

**한 번 정한 필드의 타입은 바꿀 수 없다.** 새 필드 추가, 기존 필드 multi-fields 추가는 가능하지만, 타입 자체(text → keyword 등)나 analyzer 변경은 재색인이 유일한 길이다.

```json
// ❌ 이것은 안 됨
PUT /products/_mapping
{
  "properties": {
    "status": { "type": "keyword" }   // 원래 text였다면 거부됨
  }
}
// → 400: mapper [status] cannot be changed from type [text] to [keyword]
```

### Re-index 표준 절차

```
1. 새 인덱스 생성 (올바른 mapping)
   PUT /products_v2 { ... }

2. _reindex API
   POST /_reindex
   {
     "source": { "index": "products" },
     "dest":   { "index": "products_v2" }
   }

3. alias 스왑 (다운타임 0)
   POST /_aliases
   {
     "actions": [
       { "remove": { "index": "products",    "alias": "products_search" } },
       { "add":    { "index": "products_v2", "alias": "products_search" } }
     ]
   }

4. 구 인덱스 삭제
   DELETE /products
```

> **운영 원칙**: 처음부터 인덱스를 alias 뒤에 두고 시작하라. `products` 직접 사용 금지, `products_search` alias로만 접근. 그러면 reindex가 늘 무중단으로 가능하다.

`_reindex`는 **slice**로 병렬화 가능 (`?slices=auto`). 큰 인덱스는 며칠 걸릴 수 있으므로 ILM/snapshot 등 백업과 결합해 운영.

공식 문서: [Reindex API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-reindex)

---

## 5.10 운영 시 주의점

### 필드 수 폭발 방지

- `index.mapping.total_fields.limit` (기본 1000). 동적 매핑 + 사용자 제출 필드명에서 자주 터진다.
- 대응: `dynamic: "strict"` 또는 `"runtime"` / 사용자 제공 키는 `flattened` 사용.

### doc_values 비활성으로 디스크 절약

검색만 하고 정렬/집계가 필요 없는 keyword 필드는:

```json
"trace_id": { "type": "keyword", "doc_values": false }
```

대량 로그에서 30%+ 디스크 절감 가능.

### 큰 텍스트는 _source 제외

본문 같은 거대 필드를 검색만 하고 응답에서 돌려줄 필요 없으면:

```json
"_source": { "excludes": ["body"] }
```

또는 `store: true`로 별도 저장 후 _source에서 제외.

### text 필드의 norms

text 필드는 BM25 점수 계산을 위한 `norms`(필드 길이) 데이터를 저장한다. 점수가 필요 없는(필터 전용) text 필드라면 `"norms": false`로 절감.

---

## 5.11 Key Takeaways

| 항목 | 핵심 |
|------|------|
| **Mapping** | 필드별 저장·색인·검색 규칙. 한 번 정하면 타입 변경 불가 |
| **Dynamic** | 운영 인덱스는 `strict` 권장. 오타로 인한 mapping 오염 방지 |
| **text vs keyword** | full-text 검색 = text / 정확 매칭·정렬·집계 = keyword |
| **Multi-fields** | 한 데이터, 여러 색인. `field` + `field.keyword`가 표준 |
| **Object 배열의 함정** | 평탄화로 인한 잘못된 매칭 → nested 사용 |
| **Runtime fields** | 재색인 없이 임시 추가. 검증 후 진짜 필드로 승격 |
| **템플릿 + alias** | 운영 표준. reindex를 무중단으로 만드는 기반 |
| **변경 불가 → reindex** | 처음부터 alias 뒤에 두라 |

---

*다음 장: 6장. Text Analysis — Analyzer 파이프라인과 한국어 처리*
