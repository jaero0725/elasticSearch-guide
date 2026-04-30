# Mapping & 필드 타입 선택

8.x / 9.x 기준. 필드 타입은 한번 정하면 reindex 없이는 못 바꾼다.

---

## 필드 타입 선택 플로우차트

```
                    ┌─ 정확 매칭만? ─→ keyword
                    │  (ID, status, tags)
   문자열 ─→ ─┤
                    │  ┌─ 정확 + 전문검색 둘 다? ─→ text + .keyword multi-field
                    └─→
                       │  ┌─ 카테고리 자동완성? ─→ search_as_you_type
                       └─→
                          └─ 구조화된 sub-key? ─→ flattened

                    ┌─ 정수 ─→ byte / short / integer / long
   숫자 ─→ ─┤  (가능한 작은 타입)
                    │  ┌─ 부동소수 ─→ float / double
                    └─→ ┌─ 0~1 사이 확률? ─→ half_float
                          └─ 정밀 통화? ─→ scaled_float

   날짜  ─→ date | date_nanos
   불리언 ─→ boolean

                    ┌─ 점/영역 ─→ geo_point / geo_shape
   지리   ─→ ─┤

                    ┌─ 부모-자식 무관 객체 ─→ object (기본)
   객체  ─→ ─┤  ┌─ 배열 요소 간 관계 보존? ─→ nested
                    └─ 임의 sub-key? ─→ flattened

   구조 ─→ ip | range types | join | percolator
   ML   ─→ dense_vector | sparse_vector | semantic_text
   특수 ─→ wildcard | constant_keyword | rank_feature(s)
```

---

## 핵심 타입 한눈 비교

### 문자열

| 타입 | 분석 | 정렬/집계 | 디스크 | 권장 케이스 |
|------|------|-----------|--------|-------------|
| `keyword` | 안 함 | O | 작음 | ID, enum, tag, status, hostname |
| `text` | O (analyzer) | X (fielddata 시 비추) | 중 | 본문, 제목, 코멘트 |
| `wildcard` | 안 함 | O | 중 | 부분 문자열 검색이 잦은 URL/path |
| `constant_keyword` | 안 함 | O | 0 | 인덱스당 단일 값 (e.g. tenant_id) |
| `flattened` | 안 함 (keyword처럼) | 제한 | 중 | 동적 sub-key 객체 |
| `search_as_you_type` | edge_ngram | - | 중-대 | 검색창 자동완성 |

### 숫자

| 타입 | 범위 | 바이트 | 권장 |
|------|------|--------|------|
| `byte` | -128~127 | 1 | 작은 enum 코드 |
| `short` | ±32K | 2 | 일/시간 단위 |
| `integer` | ±2.1B | 4 | 일반 정수 |
| `long` | ±9.2E18 | 8 | timestamp ms, 대형 ID |
| `unsigned_long` | 0 ~ 2^64-1 | 8 | 64비트 unsigned ID |
| `half_float` | IEEE 754 16-bit | 2 | 0~1 확률, 점수 |
| `float` | 32-bit | 4 | 일반 실수 |
| `double` | 64-bit | 8 | 정밀이 필요한 실수 |
| `scaled_float` | float × scale_factor → long | 8 | 통화 (0.01 단위) |

> 8.x는 정수 타입에 BKD tree 사용 → range query·sort 모두 빠름. 무조건 `long` 쓰지 말고 적정 크기 타입 사용.

### 날짜

| 타입 | 정밀도 | 권장 |
|------|--------|------|
| `date` | ms | 일반 timestamp |
| `date_nanos` | ns | 분산 트레이싱, 고정밀 이벤트 |

```json
{ "@timestamp": {
    "type": "date",
    "format": "strict_date_optional_time||epoch_millis"
  }
}
```

format을 명시하지 않으면 ISO 8601 + epoch_millis 자동 인식.

### Geo

| 타입 | 의미 |
|------|------|
| `geo_point` | 단일 위경도 점 |
| `geo_shape` | 다각형, 라인, 폴리곤 등 복합 지형 |

```json
{ "location": { "type": "geo_point" } }
```

색인 형식: `"location": "37.5,127.0"` 또는 `[127.0, 37.5]` (lon, lat 순)

### 벡터 (ML)

| 타입 | 용도 |
|------|------|
| `dense_vector` | 임베딩 벡터 (HNSW 인덱싱) |
| `sparse_vector` | ELSER 등 sparse 임베딩 |
| `semantic_text` | (8.x 후반~) 자동 임베딩 + 검색. inference endpoint 필요 |

```json
{
  "embedding": { "type": "dense_vector", "dims": 768, "index": true, "similarity": "cosine" },
  "title_semantic": { "type": "semantic_text", "inference_id": "my-elser" }
}
```

---

## Multi-field 패턴

```json
{
  "title": {
    "type": "text",
    "analyzer": "standard",
    "fields": {
      "keyword":  { "type": "keyword", "ignore_above": 256 },
      "english":  { "type": "text", "analyzer": "english" },
      "korean":   { "type": "text", "analyzer": "nori" },
      "edge":     { "type": "search_as_you_type" }
    }
  }
}
```

용도:
- `title` — 기본 검색
- `title.keyword` — 정렬, 집계, 정확 일치
- `title.english`, `title.korean` — 언어별 분석
- `title.edge` — 자동완성

---

## ignore_above

```json
{ "user_input": { "type": "keyword", "ignore_above": 256 } }
```

색인 시 256 byte 초과는 **인덱싱 skip** (저장은 됨, 검색만 안 됨). 사용자 입력 등 길이 폭주 방지.

---

## doc_values

| 타입 | 기본 doc_values | 의미 |
|------|----------------|------|
| 대부분 | true | 정렬·집계·script 가능 |
| `text` | false | 정렬·집계 불가 (fielddata로 가능하나 권장 X) |

```json
{ "logger": { "type": "keyword", "doc_values": false } }
```

검색만 하고 정렬/집계 절대 안 한다면 disable해서 디스크 절약 가능. 후회할 가능성 있음.

---

## index, store, source 옵션

| 옵션 | 기본 | 효과 |
|------|------|------|
| `index: false` | true | inverted index 미생성 (검색 불가, 저장만) |
| `store: true` | false | _source와 별개로 필드 단독 저장 (큰 _source 일부만 fetch) |
| `enabled: false` | true | object 통째로 색인·검색 비활성, 저장만 |

```json
{
  "raw_payload": { "type": "object", "enabled": false }
}
```

큰 raw JSON을 저장만 하고 싶을 때 (검색 안 함).

---

## _source 비활성?

```json
{ "_source": { "enabled": false } }
```

**거의 모든 경우 비추**. _source가 없으면 reindex, update, highlight, partial fetch가 모두 깨진다.

대신 `_source.includes` / `excludes`로 필드 단위 제외:

```json
{ "_source": { "excludes": ["raw_html", "binary_blob"] } }
```

---

## copy_to: _all 대체

```json
{
  "title":    { "type": "text", "copy_to": "all_text" },
  "body":     { "type": "text", "copy_to": "all_text" },
  "all_text": { "type": "text" }
}
```

여러 필드를 하나의 검색용 필드에 합쳐 색인. 검색 시 `all_text` 단일 필드만 매칭.

---

## Dynamic 정책

```json
{
  "mappings": {
    "dynamic": "strict",
    "properties": { ... },
    "dynamic_templates": [
      {
        "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": { "type": "keyword", "ignore_above": 256 }
        }
      }
    ]
  }
}
```

| 값 | 동작 |
|----|------|
| `true` (기본) | 새 필드 자동 추가 (위험) |
| `runtime` | runtime field로 등록, 색인은 안 함 |
| `false` | 무시 (저장은 됨, 검색 안 됨) |
| `strict` | 에러 |

운영 권장: 핵심 인덱스는 `strict`, 로그처럼 진화 가능한 곳은 `runtime` + dynamic_templates.

---

## Index template + component template

```
[component template: settings_default]
[component template: mappings_logs_common]
       ↓ 합성
[index template: logs-template (patterns: logs-*)]
       ↓ 매칭
[새 인덱스 생성 시 자동 적용]
```

```json
PUT _component_template/settings_default
{
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "refresh_interval": "30s"
    }
  }
}

PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "composed_of": ["settings_default", "mappings_logs_common"],
  "priority": 100
}
```

---

## 운영 팁

| 팁 | 이유 |
|----|------|
| 인덱스 생성 전 mapping 명시 | dynamic 폭주 방지 |
| `dynamic: strict` (가능하면) | 스키마 드리프트 차단 |
| keyword에 `ignore_above` 항상 | 길이 폭주 차단 |
| text 필드는 정렬·집계 안 함을 명시적으로 인지 | fielddata 함정 회피 |
| `index.mapping.total_fields.limit` 모니터링 | 1000 기본, 폭증 시 위험 신호 |
| 같은 데이터, 다른 분석 필요 → multi-field | reindex 회피 |
| 큰 _source 부분만 자주 쓴다 → `store: true` 후보 | 디스크 vs fetch 비용 |
| date format은 명시 | 파싱 실패 추적 용이 |
| nested는 마지막 수단 | 10.3 참고 |

---

## 흔한 실수 → 즉시 처방

| 실수 | 처방 |
|------|------|
| `status: text`로 둠 | multi-field로 `.keyword` 추가 후 새 코드에서 `.keyword` 사용 |
| `_id`로 큰 hash 사용 | 자동 ID로 전환 (auto-id 색인 경로 최적화) |
| `dims`를 잘못 설정 | 새 인덱스로 reindex (dims 변경 불가) |
| analyzer 변경 필요 | reindex만 가능. ILM이나 alias 스왑 활용 |
| 필드 누락 | dynamic_templates로 향후 자동 매핑 |

---

*관련 cheatsheets: analyzer_selection, ilm_policy_templates*
