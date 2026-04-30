# A2. text / keyword 혼동

## 증상

전형적인 에러 또는 이상 동작:

```
{
  "type": "illegal_argument_exception",
  "reason": "Text fields are not optimised for operations that require per-document
             field data like aggregations and sorting, so these operations are
             disabled by default. Please use a keyword field instead."
}
```

또는:

```
{
  "type": "illegal_argument_exception",
  "reason": "Fielddata is disabled on [name] in [products]"
}
```

증상 양상:

- `terms` 집계가 작동하지 않거나, 동작은 하는데 토큰 단위로 쪼개진 결과가 나옴 (`갤럭시`, `s24`, `울트라`로 분리)
- `sort`가 일관되지 않거나 ordering이 이상함
- 정확한 일치(`term`) 검색이 안 됨 (`name = "iPhone 15"` 인데 hit이 0)
- fielddata를 켰더니 heap이 폭주

---

## 원인

`text`와 `keyword`는 완전히 다른 용도의 타입이다.

| 측면 | `text` | `keyword` |
|------|--------|-----------|
| 색인 방식 | 분석기로 토큰화 | 원문 그대로 1개 토큰 |
| 용도 | 전문 검색 (full-text) | 정확 일치, 정렬, 집계, 필터 |
| `match` 쿼리 | 토큰 매칭 | 분석 안 됨 (사실상 term) |
| `term` 쿼리 | 토큰 단위 매칭 (사용 비권장) | 원문 정확 일치 |
| 정렬/집계 | 기본 불가 (fielddata 필요) | 기본 가능 (`doc_values`) |
| 메모리 | doc_values 없음 | doc_values 사용 |

```
"갤럭시 S24 울트라" 를 'text' (standard analyzer) 로 색인 시:
  → 토큰: ["갤럭시", "s24", "울트라"]
  → 정렬하면 어느 토큰을 기준으로 정렬할지 모호
  → 집계하면 토큰 별로 카운트 (의도 X)

같은 값을 'keyword' 로 색인 시:
  → 토큰: ["갤럭시 S24 울트라"]  (원문 그대로 1개)
  → 정렬/집계가 정상 동작
```

흔한 실수: 동적 매핑이 만든 `text` 필드(서브 필드 `keyword`)인데 본 필드만 사용.

```
동적 매핑 결과:
  name (text)
    └── name.keyword (keyword, ignore_above: 256)

쿼리에서:
  "term": { "name": "iPhone 15" }       ← 토큰화된 text와 비교 → hit 0
  "term": { "name.keyword": "iPhone 15" } ← 원문과 비교 → 정상 hit
```

---

## 진단

### 1) 매핑 확인

```bash
GET /products/_mapping/field/name

# 응답:
# "name": {
#   "type": "text",
#   "fields": {
#     "keyword": { "type": "keyword", "ignore_above": 256 }
#   }
# }
```

### 2) 분석기로 어떻게 토큰화되는지 확인

```bash
GET /products/_analyze
{
  "field": "name",
  "text": "갤럭시 S24 울트라"
}

# 응답: 토큰 배열
# [{ "token": "갤럭시" }, { "token": "s24" }, { "token": "울트라" }]
```

### 3) 잘못된 쿼리가 실제로 무엇을 매칭하는지

```bash
GET /products/_search
{
  "explain": true,
  "query": {
    "term": { "name": "iPhone 15" }
  }
}
# explanation에 "no matching term"으로 나옴
```

### 4) fielddata 사용량 (이미 켜져 있다면)

```bash
GET /_cat/fielddata?v&s=size:desc

# 또는 노드별
GET /_nodes/stats/indices/fielddata?fields=name
```

---

## 해결

### 케이스 1: 정확 일치 검색만 필요 (정렬/집계 위주)

```bash
# 인덱스 재설계: keyword로 매핑
PUT /products-v2
{
  "mappings": {
    "properties": {
      "sku":   { "type": "keyword" },         // 정렬·집계 OK
      "brand": { "type": "keyword" }
    }
  }
}
```

### 케이스 2: 전문 검색 + 집계 둘 다 필요 — 멀티 필드

```bash
PUT /products-v2
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "ko_index",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      }
    }
  }
}
```

쿼리에서 용도 분리:

```bash
# 전문 검색
GET /products-v2/_search
{
  "query": { "match": { "name": "갤럭시 울트라" } }
}

# 정확 일치
GET /products-v2/_search
{
  "query": { "term": { "name.keyword": "갤럭시 S24 울트라" } }
}

# 집계
GET /products-v2/_search
{
  "size": 0,
  "aggs": {
    "by_brand": {
      "terms": { "field": "name.keyword", "size": 10 }
    }
  }
}

# 정렬
GET /products-v2/_search
{
  "sort": [{ "name.keyword": "asc" }]
}
```

### 케이스 3: 이미 운영 중이라 reindex 부담 — runtime field

```bash
# 매핑 추가 (기존 데이터 변경 없이)
PUT /products/_mapping
{
  "runtime": {
    "name_kw": {
      "type": "keyword",
      "script": {
        "source": "emit(params._source.name)"
      }
    }
  }
}

# 정렬/집계 가능
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_name": { "terms": { "field": "name_kw", "size": 10 } }
  }
}
```

> Runtime field는 매 검색마다 스크립트가 실행됨 → 대량 데이터/잦은 호출에는 부적합. 임시 회피용.

### 절대 하지 말 것: `text` 에 fielddata 켜기

```bash
# 위험! heap OOM 가능성
PUT /products/_mapping
{
  "properties": {
    "name": { "type": "text", "fielddata": true }
  }
}
```

`fielddata`는 모든 토큰을 heap에 올린다. 토큰 수가 백만 단위로 늘면 GC 폭증, OOM.
→ 별도 케이스 [B3](B3_fielddata_on_text.md) 참고.

---

## 예방

### 1) 명시적 매핑을 항상 사용

```bash
PUT /my-index
{
  "mappings": {
    "dynamic": "strict",     // 동적 매핑 차단
    "properties": {
      "id":      { "type": "keyword" },
      "name":    {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } }
      },
      "tags":    { "type": "keyword" },
      "content": { "type": "text" }
    }
  }
}
```

### 2) Dynamic templates로 자동 멀티 필드

```json
"dynamic_templates": [
  {
    "strings_as_text_keyword": {
      "match_mapping_type": "string",
      "mapping": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      }
    }
  }
]
```

기본 동적 매핑이 이미 이런 구조를 만들지만, 통제하에 두는 편이 명확함.

### 3) 결정 가이드

```
"이 필드를 어떻게 쓸까?"

  전문 검색 (단어/구문 매칭)            → text
  정확 일치 (= 비교)                    → keyword
  정렬                                  → keyword (또는 numeric/date)
  집계 (terms, cardinality)             → keyword
  필터 (where IN)                       → keyword
  ID/SKU/이메일/IP                      → keyword (분석 불필요)
  본문/제목/설명                        → text (검색 + .keyword 멀티)
```

### 4) 안티패턴

- 모든 string을 `text` 하나로만 매핑 → 정렬/집계 불가
- 모든 string을 `keyword` 하나로만 매핑 → 전문 검색 불가
- `text`에 `fielddata: true` → heap 폭주
- `name.keyword`가 있는데 `name`으로 정확 매칭 시도 → 미매치

**핵심: 같은 데이터라도 "검색용 text" + "집계용 keyword" 멀티 필드가 디폴트.**
