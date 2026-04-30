# A1. 매핑 폭증 (Mapping Explosion)

## 증상

```
{
  "type": "illegal_argument_exception",
  "reason": "Limit of total fields [1000] has been exceeded while adding new fields [N]"
}
```

다른 양상:

- 인덱스가 시간이 갈수록 점점 느려지고 메모리가 부족하다고 클러스터 로그에 나타남
- `GET /my-index/_mapping` 응답이 메가바이트 단위
- `_cluster/state`가 거대해서 마스터 노드 CPU 사용률이 치솟음
- 새 필드 색인이 거부됨

---

## 원인

Elasticsearch는 모든 인덱스 매핑을 **클러스터 상태(cluster state)** 에 보관한다. 매핑이 1만, 10만 필드로 늘어나면:

```
영향:
  ① cluster state 크기 증가 → 마스터 노드 메모리/CPU 부담
  ② 새 노드 합류 시 cluster state 전송 시간 증가
  ③ 각 샤드의 Lucene 메타데이터 메모리 증가 (heap 사용)
  ④ 매핑 lookup이 O(n)이라 색인/검색 모두 느려짐
```

전형적 원인:

```json
// 동적 매핑이 켜진 상태에서 매번 새 키가 들어오는 문서
{ "user_id_alice":   123 }
{ "user_id_bob":     456 }
{ "user_id_charlie": 789 }
// → user_id_alice, user_id_bob, user_id_charlie 가 각각 별도 필드로 매핑됨
```

또는 외부 시스템에서 들어오는 JSON에 가변 키가 들어있는 경우 (예: feature flag 이름, 동적 라벨).

---

## 진단

### 1) 인덱스의 필드 수 확인

```bash
# 8.x: field_caps 또는 _mapping 응답 크기로 추정
GET /my-index/_field_caps?fields=*
# 응답의 fields 객체 키 개수 = 필드 수
```

```bash
# 더 직접적: _mapping 으로 받아서 jq 등으로 카운트
GET /my-index/_mapping
```

```bash
# 클러스터 전체에서 가장 큰 매핑
GET /_cat/indices?v&s=ss:desc&h=index,docs.count,store.size,pri,rep
```

### 2) 현재 한도 확인

```bash
GET /my-index/_settings?include_defaults=true&filter_path=*.settings.index.mapping
GET /my-index/_settings?include_defaults=true&filter_path=*.defaults.index.mapping
```

기본값:

```
index.mapping.total_fields.limit:        1000
index.mapping.depth.limit:                  20
index.mapping.nested_fields.limit:          50
index.mapping.field_name_length.limit:    50000
```

### 3) 클러스터 상태 크기

```bash
GET /_cluster/state/metadata?human&filter_path=metadata.indices.*.mappings
# 응답 크기로 매핑 비대 여부 판단
```

### 4) 어떤 필드가 폭증하고 있는가

```bash
# field_caps 응답에서 prefix 별 카운트
GET /my-index/_field_caps?fields=*
# → fields 객체에서 패턴 분석
# 예: user_id_* 가 5만 개 → 동적 매핑 폭증 의심
```

---

## 해결

### 단계 1: 즉각 완화 — 동적 매핑 차단

```bash
# 새 필드 자동 추가 막기 (기존 인덱스에)
PUT /my-index/_mapping
{
  "dynamic": "false"
}

# 옵션 의미:
# "true"   : 새 필드 자동 매핑 (기본, 위험)
# "runtime": 새 필드를 runtime field로 → 디스크 매핑에 안 들어감
# "false"  : 새 필드 무시 (저장은 _source에, 검색 불가)
# "strict" : 새 필드 발견 시 색인 거부 (가장 엄격)
```

### 단계 2: 한도 임시 상향 (응급)

```bash
PUT /my-index/_settings
{
  "index.mapping.total_fields.limit": 2000
}
```

> 임시방편. 근본 원인을 해결하지 않으면 곧 다시 한도에 도달.

### 단계 3: 근본 해결 — `flattened` 또는 `runtime` 사용

가변 키-값을 한 필드로 묶기:

```bash
# 새 인덱스 매핑
PUT /my-index-v2
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "@timestamp":  { "type": "date" },
      "service":     { "type": "keyword" },
      "labels":      { "type": "flattened" },
      "metrics":     { "type": "flattened" }
    }
  }
}

# 색인할 때
POST /my-index-v2/_doc
{
  "@timestamp": "2025-04-30T10:00:00Z",
  "service": "payment-api",
  "labels": {
    "user_id_alice":   123,
    "user_id_bob":     456,
    "user_id_charlie": 789
  }
}

# 쿼리: 점 표기로
GET /my-index-v2/_search
{
  "query": {
    "term": { "labels.user_id_alice": "123" }
  }
}
```

`flattened` 타입의 특징:

```
- 객체 전체가 하나의 필드로 매핑됨 → 매핑 폭증 없음
- 모든 값이 keyword로 색인됨 (text 분석 안 됨)
- 점 표기로 하위 키 쿼리 가능
- agg/sort 가능 (keyword와 동일)
- 단점: full-text 분석/range 쿼리는 불가
```

### 단계 4: 데이터 마이그레이션

```bash
# Reindex API
POST /_reindex
{
  "source": { "index": "my-index" },
  "dest":   { "index": "my-index-v2" },
  "script": {
    "source": """
      def labels = new HashMap();
      for (entry in ctx._source.entrySet()) {
        if (entry.getKey().startsWith("user_id_")) {
          labels.put(entry.getKey(), entry.getValue());
        }
      }
      ctx._source.labels = labels;
      // 원래 user_id_* 필드는 제거
      ctx._source.entrySet().removeIf(e -> e.getKey().startsWith("user_id_"));
    """
  }
}

# 트래픽 전환 후
DELETE /my-index
```

---

## 예방

### 1) 인덱스 템플릿에서 명시적 매핑 + dynamic 제어

```json
PUT /_index_template/safe-template
{
  "index_patterns": ["myapp-*"],
  "template": {
    "settings": {
      "index.mapping.total_fields.limit": 2000,
      "index.mapping.depth.limit": 5
    },
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "@timestamp": { "type": "date" },
        "message":    { "type": "match_only_text" },
        "labels":     { "type": "flattened" }
      }
    }
  }
}
```

### 2) Dynamic Templates로 패턴별 매핑

```json
PUT /my-index
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": { "type": "keyword", "ignore_above": 256 }
        }
      },
      {
        "no_long_strings": {
          "match_mapping_type": "string",
          "match": "*_text",
          "mapping": { "type": "text", "norms": false }
        }
      }
    ]
  }
}
```

### 3) 모니터링 알람

```bash
# 인덱스별 필드 수 모니터링 (Watcher 또는 외부 모니터링)
GET /_field_caps?fields=*&include_unmapped=false
# 응답 필드 수가 800 넘어가면 경고
```

### 4) 운영 가이드라인

```
필드 수 권장:
  < 500개   : 안전
  500~1000  : 주의 (한도 임박)
  > 1000    : 동적 매핑 점검, flattened/runtime 검토
```

**핵심 원칙: 가변 키를 가진 데이터는 절대 dynamic mapping에 맡기지 말고, `flattened` 또는 별도 keyword/value 페어로 정규화하라.**
