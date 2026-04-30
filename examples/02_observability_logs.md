# 예제 2. 로그/Observability

서비스 수십~수백 개에서 발생하는 애플리케이션 로그를 Elasticsearch에 모아 검색·분석하는 시나리오. ECS, Data Stream, ILM, hot-warm-cold 아키텍처가 어떻게 결합되는지 살펴본다.

대상 버전: Elasticsearch 8.x / 9.x

---

## 1. 요구사항

- 일 평균 5억 건, 피크 1.5만 EPS (event/sec)
- 서비스 50개, 환경 3개 (prod/stage/dev)
- 로그 보관 정책:
  - 최근 7일: 빠른 검색 (hot)
  - 7~30일: 가끔 조회 (warm)
  - 30~90일: 감사용 (cold)
  - 90일 후: 삭제
- 검색 패턴:
  - 시간 범위 + 서비스 + 로그 레벨 + 메시지 키워드
  - 에러 빈도 시계열 차트
  - trace_id로 단일 요청 추적

---

## 2. ECS (Elastic Common Schema)

여러 서비스의 로그를 한 인덱스에 넣을 때 필드 이름 규칙을 통일하는 표준. 8.x에서는 모든 통합(integration)이 ECS를 따른다.

핵심 필드:

| 필드 | 의미 | 예시 |
|------|------|------|
| `@timestamp` | 이벤트 발생 시각 (필수) | `2025-04-30T10:00:00.000Z` |
| `service.name` | 서비스 식별자 | `payment-api` |
| `service.environment` | 배포 환경 | `prod`, `stage` |
| `host.name` | 호스트명 | `app-pod-7f9d` |
| `log.level` | 로그 레벨 | `error`, `warn`, `info`, `debug` |
| `message` | 사람이 읽는 메시지 | `Payment failed: timeout` |
| `event.dataset` | 데이터셋 | `nginx.access`, `app.log` |
| `trace.id`, `span.id` | 분산 추적 ID | `4bf92f3577b34da6` |
| `error.stack_trace` | 스택트레이스 | (text) |
| `client.ip`, `url.path`, `http.response.status_code` | HTTP 컨텍스트 | |

ECS를 따르면 Kibana의 Logs/APM/SIEM 앱이 즉시 동작한다.

---

## 3. Data Stream + ILM 아키텍처

### 3.1 노드 역할 분리

```yaml
# elasticsearch.yml — hot 노드
node.roles: [ data_hot, data_content, ingest ]
node.attr.data: hot

# warm 노드 (느린 디스크 OK)
node.roles: [ data_warm ]
node.attr.data: warm

# cold 노드 (object store/HDD)
node.roles: [ data_cold ]
node.attr.data: cold
```

### 3.2 ILM 정책

```json
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_primary_shard_size": "50gb"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": { "include": { "data": "warm" }, "number_of_replicas": 1 },
          "shrink":   { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": { "include": { "data": "cold" }, "number_of_replicas": 0 },
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

**phase 의미:**

```
hot (0~7d)   : SSD, replicas=1, 색인 진행 중
warm (7~30d) : HDD/대용량, shrink로 샤드 수 축소, force_merge로 세그먼트 1개
cold (30~90d): replica=0 (스냅샷 또는 searchable snapshot 활용 가능)
delete (90d~): 인덱스 삭제
```

### 3.3 인덱스 템플릿 (Data Stream용)

```json
PUT /_component_template/logs-mappings
{
  "template": {
    "mappings": {
      "_source": { "mode": "synthetic" },
      "properties": {
        "@timestamp":            { "type": "date" },
        "service.name":          { "type": "keyword" },
        "service.environment":   { "type": "keyword" },
        "service.version":       { "type": "keyword" },
        "host.name":             { "type": "keyword" },
        "host.ip":               { "type": "ip" },
        "log.level":             { "type": "keyword" },
        "message":               { "type": "match_only_text" },
        "event.dataset":         { "type": "keyword" },
        "trace.id":              { "type": "keyword" },
        "span.id":               { "type": "keyword" },
        "error.message":         { "type": "match_only_text" },
        "error.stack_trace":     { "type": "wildcard" },
        "url.path":              { "type": "keyword" },
        "http.response.status_code": { "type": "short" },
        "client.ip":             { "type": "ip" },
        "labels":                { "type": "flattened" }
      }
    }
  }
}

PUT /_component_template/logs-settings
{
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-policy",
      "index.codec": "best_compression",
      "index.refresh_interval": "5s",
      "index.number_of_shards": 2,
      "index.number_of_replicas": 1,
      "index.routing.allocation.include.data": "hot"
    }
  }
}

PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "composed_of": ["logs-mappings", "logs-settings"],
  "priority": 200
}
```

### 설계 근거

| 결정 | 이유 |
|------|------|
| Data Stream | rollover/ILM이 자동. 직접 `logs-2025-04-30` 같은 인덱스를 만들지 않음 |
| `match_only_text` | 점수 계산 메타데이터 미저장 → 디스크 30~50% 절감, 로그처럼 단순 매칭만 필요할 때 적합 |
| `wildcard` (stack_trace) | 와일드카드 검색 최적화된 신규 타입. `*NullPointerException*` 빠름 |
| `flattened` (labels) | 동적 키-값을 한 필드에 저장 → 매핑 폭증 방지 (→ `troubleshooting/A3`) |
| `synthetic _source` | `_source`를 저장하지 않고 필요 시 재구성 → 디스크 절감 (logsdb/timeseries에서 권장) |
| `best_compression` | DEFLATE로 더 강하게 압축 (CPU↑, 디스크↓) |
| `max_primary_shard_size: 50gb` | shard 1개가 50GB 도달 시 rollover (시간보다 크기 기준) |

> **참고**: 8.17부터 `index.mode: logsdb`가 GA되어 로그 인덱스에 더욱 최적화된 저장 포맷을 제공한다. 신규 클러스터는 logsdb 모드 검토 권장.

---

## 4. Filebeat → Elasticsearch

### 4.1 Filebeat 설정

```yaml
# filebeat.yml
filebeat.inputs:
  - type: filestream
    id: app-logs
    paths:
      - /var/log/app/*.log
    parsers:
      - ndjson:
          target: ""
          add_error_key: true
    fields:
      service.name: payment-api
      service.environment: prod
    fields_under_root: true

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~

output.elasticsearch:
  hosts: ["https://es:9200"]
  api_key: "${ES_API_KEY}"
  # data stream 자동 라우팅
  index: "logs-app.payment-%{[agent.version]}"
  bulk_max_size: 4096
  worker: 4
```

### 4.2 인입 흐름

```
앱 (JSON 로그) → /var/log/app/*.log
         ↓
       Filebeat (파일 tail, NDJSON 파싱)
         ↓ bulk
       Elasticsearch ingest pipeline (필요시 grok/dissect)
         ↓
       Data Stream `logs-app.payment-default`
         ↓ 매일 또는 50GB → rollover
       hot → warm → cold → delete (ILM 자동)
```

### 4.3 ingest pipeline (선택)

```json
PUT /_ingest/pipeline/logs-app-enrich
{
  "processors": [
    { "set":    { "field": "event.ingested", "value": "{{_ingest.timestamp}}" } },
    { "rename": { "field": "msg", "target_field": "message", "ignore_missing": true } },
    { "lowercase": { "field": "log.level", "ignore_missing": true } },
    {
      "geoip": {
        "field": "client.ip",
        "target_field": "client.geo",
        "ignore_missing": true
      }
    }
  ]
}
```

`logs-settings` 컴포넌트 템플릿에 `index.default_pipeline: logs-app-enrich` 추가하면 인덱싱 시 자동 적용.

---

## 5. 검색 쿼리 패턴

### 5.1 시간 + 서비스 + 레벨 (Kibana KQL → DSL)

KQL: `service.name : "payment-api" and log.level : "error" and message : "timeout"`

동등 DSL:

```json
GET /logs-app.payment*/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term":  { "service.name": "payment-api" } },
        { "term":  { "log.level":    "error" } },
        { "range": { "@timestamp":   { "gte": "now-1h" } } }
      ],
      "must": [
        { "match": { "message": "timeout" } }
      ]
    }
  },
  "sort": [{ "@timestamp": "desc" }],
  "size": 50
}
```

### 5.2 에러 빈도 시계열 (date_histogram)

```json
GET /logs-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "log.level": "error" } },
        { "range": { "@timestamp": { "gte": "now-24h" } } }
      ]
    }
  },
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "5m",
        "min_doc_count": 0,
        "extended_bounds": { "min": "now-24h", "max": "now" }
      },
      "aggs": {
        "by_service": {
          "terms": { "field": "service.name", "size": 10 }
        }
      }
    }
  }
}
```

### 5.3 단일 요청 추적 (trace_id)

```json
GET /logs-*/_search
{
  "query": {
    "term": { "trace.id": "4bf92f3577b34da6a3ce929d0e0e4736" }
  },
  "sort": [{ "@timestamp": "asc" }],
  "size": 200
}
```

### 5.4 stack_trace 검색 (`wildcard` 필드)

```json
GET /logs-*/_search
{
  "query": {
    "wildcard": {
      "error.stack_trace": "*NullPointerException*at com.shop.payment*"
    }
  }
}
```

---

## 6. 운영 팁

### 6.1 ILM 진행 상태 확인

```bash
GET /logs-app.payment*/_ilm/explain

# 응답에서 phase, step, action_time_millis 확인
# step이 "ERROR"면 actions로 무엇이 막혔는지 추적
```

### 6.2 rollover 강제 트리거 (테스트용)

```bash
POST /logs-app.payment-default/_rollover
{
  "conditions": {
    "max_age":  "1d",
    "max_size": "50gb"
  }
}
```

### 6.3 Data Stream 백업

```bash
# 스냅샷 → S3 등 reposository
PUT /_snapshot/s3_repo/snap-20250430
{
  "indices": "logs-*",
  "include_global_state": false
}

# searchable snapshot (cold/frozen tier)
POST /logs-app.payment-2024.12.01-000123/_searchable_snapshot
```

### 6.4 비용 절감 체크리스트

```
✓ codec: best_compression
✓ replicas warm phase에서 그대로 유지하지 말기 (cold에서는 0으로)
✓ force_merge → max_num_segments=1 (warm phase)
✓ shrink (warm) → shard 수 축소
✓ logsdb 모드 또는 synthetic _source
✓ unused field disable (index: false)
```

### 6.5 안티패턴

- **단일 거대 인덱스**: `logs-2025` 한 곳에 다 넣지 않기 → 시간 기반 인덱스로 분리
- **모든 필드 인덱싱**: `enabled: false` 또는 `index: false`로 검색 안 쓰는 필드 제외
- **Refresh 1초 고집**: 로그는 5~30s가 일반적 (→ `troubleshooting/D2`)

---

## 7. 요약

| 항목 | 선택 |
|------|------|
| 표준 스키마 | ECS |
| 인덱스 형태 | Data Stream + Index Template |
| 라이프사이클 | ILM 4단계 (hot→warm→cold→delete) |
| 노드 분리 | `data_hot`, `data_warm`, `data_cold` |
| 본문 필드 | `match_only_text` (스코어 메타 미저장) |
| 동적 라벨 | `flattened` |
| 색인 파이프라인 | Filebeat → ingest pipeline → Data Stream |
| 검색 패턴 | filter 컨텍스트 + `date_histogram` + `terms` agg |
