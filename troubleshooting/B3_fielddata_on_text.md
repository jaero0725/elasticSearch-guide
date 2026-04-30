# B3. text 필드의 fielddata로 인한 Heap OOM

## 증상

```
{
  "type": "circuit_breaking_exception",
  "reason": "[fielddata] Data too large, data for [name] would be [...]
             which is larger than the limit of [...]"
}
```

또는 더 심각하게:

- 노드가 `OutOfMemoryError`로 다운
- GC 시간이 초당 수 초로 길어짐 (`Old GC pause: 8.5s`)
- `_cat/fielddata` 출력에서 단일 필드가 수 GB 차지
- Kibana 대시보드 일부 위젯이 응답 안 함

---

## 원인

### text 필드의 정렬/집계는 기본적으로 불가능

text는 토큰화된 inverted index만 갖는다. 정렬/집계용 `doc_values`가 없다.

```
keyword 필드:
  inverted index + doc_values (디스크 정주, mmap, 효율적)
  → 정렬/집계가 자연스럽게 빠름

text 필드:
  inverted index만
  → 정렬/집계 불가, 켜려면 fielddata = on-heap 자료구조
```

### fielddata 활성화의 위험

```bash
# 위험! (수행하지 말 것)
PUT /products/_mapping
{
  "properties": {
    "description": { "type": "text", "fielddata": true }
  }
}
```

```
fielddata 동작:
  1) 정렬/집계 첫 요청 시
  2) 필드의 모든 문서, 모든 토큰을 heap에 메모리로 로드
  3) heap 위 자료구조에서 정렬·집계 수행

문제:
  토큰 수 = 문서 수 × 평균 토큰 수
  description 필드 1억 문서 × 평균 50 토큰 = 50억 토큰
  → 수십 GB heap 점유 → OOM
```

### 흔한 잘못된 호출

```json
// 'name' 이 text 인 경우
GET /products/_search
{
  "size": 0,
  "aggs": { "by_name": { "terms": { "field": "name" } } }   // ← 위험
}

// 또는
GET /products/_search
{
  "sort": [{ "name": "asc" }]   // ← 위험
}
```

ES는 "fielddata를 써야 하는데 disabled"라고 거부한다. 그 시점에 누군가 잘못된 해법(fielddata=true)을 적용하면서 사고가 시작된다.

---

## 진단

### 1) fielddata 사용량 확인

```bash
GET /_cat/fielddata?v&s=size:desc

# 응답 예:
# id    host       ip           node    field          size
# xxx   es-data-1  10.0.0.1     node-1  description    8.2gb
# xxx   es-data-2  10.0.0.2     node-2  description    8.1gb
# ...
```

```bash
# 더 자세한 노드별 통계
GET /_nodes/stats/indices/fielddata?fields=*&human

# heap 사용량 % 와 fielddata 비율 함께 보기
GET /_cat/nodes?v&h=name,heap.percent,heap.current,heap.max
```

### 2) circuit breaker 상태

```bash
GET /_nodes/stats/breaker

# 응답 예:
# "breakers": {
#   "fielddata":     { "limit_size_in_bytes": ..., "estimated_size_in_bytes": ..., "tripped": 12 },
#   "request":       { ... },
#   "in_flight_requests": { ... },
#   "parent":        { ... }
# }
# tripped > 0 이면 이미 발동한 적이 있다는 뜻
```

### 3) 어떤 필드에 fielddata가 켜져 있는지

```bash
GET /_all/_mapping?filter_path=**.properties.*.fielddata,**.properties.*.type

# fielddata: true 가 설정된 필드 식별
```

### 4) Hot threads / GC 로그

```bash
GET /_nodes/hot_threads
GET /_nodes/stats/jvm

# jvm.gc.collectors.old.collection_time_in_millis 가 빠르게 증가하면 GC 폭주
```

---

## 해결

### 단계 1: 즉각 완화 — fielddata 캐시 비우기

```bash
POST /_cache/clear?fielddata=true

# 특정 인덱스만
POST /products/_cache/clear?fielddata=true

# 특정 필드만
POST /_cache/clear?fielddata=true&fields=description
```

> 일회성 응급 처방. 다음 정렬/집계 요청에서 다시 로드된다.

### 단계 2: fielddata 끄기

```bash
PUT /products/_mapping
{
  "properties": {
    "description": {
      "type": "text",
      "fielddata": false      // 명시적으로 끄기
    }
  }
}
```

### 단계 3: 근본 해결 — 멀티 필드로 keyword 추가

```bash
# 새 인덱스 매핑
PUT /products-v2
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "description": {
        "type": "text"
      },
      "category": {
        "type": "keyword"   // 처음부터 keyword (분석 필요 없음)
      }
    }
  }
}

# 정렬/집계는 .keyword 사용
GET /products-v2/_search
{
  "size": 0,
  "aggs": {
    "by_name": { "terms": { "field": "name.keyword", "size": 10 } }
  }
}
```

### 단계 4: 기존 인덱스에 keyword 멀티필드 추가

```bash
# 매핑에는 새 멀티 필드 추가 가능 (기존 필드 변경은 불가)
PUT /products/_mapping
{
  "properties": {
    "name": {
      "type": "text",
      "fields": {
        "keyword": { "type": "keyword", "ignore_above": 256 }
      }
    }
  }
}

# 단, 이미 색인된 문서는 새 멀티 필드에 색인 안 됨
# 기존 문서를 새 멀티필드에 채우려면 update_by_query 또는 reindex
POST /products/_update_by_query?wait_for_completion=false
{
  "query": { "match_all": {} }
}
```

### 단계 5: text 필드를 사용해야만 한다면 — runtime field

색인을 다시 못 하는 상황 + 즉시 정렬/집계 필요 시:

```bash
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

GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_name": { "terms": { "field": "name_kw" } }
  }
}
```

> Runtime field도 무료가 아니다 — 매 요청마다 _source 파싱. 그러나 fielddata처럼 heap을 점유하지는 않는다.

### 단계 6: circuit breaker 한도 조정 (응급 시)

```bash
PUT /_cluster/settings
{
  "persistent": {
    "indices.breaker.fielddata.limit": "30%",       // 기본 40%
    "indices.breaker.request.limit":   "40%",
    "indices.breaker.total.limit":     "70%"
  }
}
```

> 한도를 낮추면 거부가 빨라져서 OOM은 줄지만, 정상 쿼리도 거부될 수 있음.

---

## 예방

### 1) 인덱스 매핑 가이드라인

```
text 필드 = 검색 전용. 정렬/집계 절대 금지.
정렬/집계가 필요한 필드는 항상 멀티필드로 .keyword 추가.

좋은 매핑:
  PUT /docs {
    "mappings": {
      "properties": {
        "title": {
          "type": "text",
          "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } }
        },
        "tags":   { "type": "keyword" },
        "category": { "type": "keyword" },
        "body":   { "type": "text" }    // 본문은 검색만
      }
    }
  }
```

### 2) 모니터링 알람

```bash
# 정기 체크 — fielddata 사용량
GET /_cat/fielddata?bytes=mb&s=size:desc

# 임계치 예: 노드당 fielddata > 1GB 이면 알람
GET /_nodes/stats/indices/fielddata?human
```

### 3) circuit breaker tripped 카운트 모니터

```bash
GET /_nodes/stats/breaker?filter_path=**.fielddata.tripped
# 0보다 크면 곧 OOM 후보 → 매핑 점검
```

### 4) 안티패턴

- `text` 필드를 정렬·집계 키로 사용
- `fielddata: true`를 켜는 것 (거의 항상 설계 실수의 대안)
- `name`만 매핑하고 `name.keyword` 멀티필드 안 만들기
- `match` 쿼리 결과를 그대로 정렬 ("스코어 순"이 아니면 keyword 필드 사용)

**핵심: text 정렬·집계가 필요하다 = 매핑이 잘못됐다는 신호. fielddata=true는 거의 항상 잘못된 답이다.**
