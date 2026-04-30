# 예제 3. 시계열 메트릭

쿠버네티스/호스트/애플리케이션 메트릭을 Elasticsearch에 저장하고 대시보드에 사용하는 시나리오. 8.7+의 TSDS(Time Series Data Stream), 8.10+의 downsampling, runtime field가 어떻게 쓰이는지 본다.

대상 버전: Elasticsearch 8.7+ (TSDS GA), 8.10+ (downsampling GA), 9.x

---

## 1. 요구사항

- Metricbeat가 수집하는 호스트/컨테이너 메트릭 (CPU, memory, network 등)
- 1만 개 호스트 × 30초 간격 → 약 일 28억 개 데이터포인트
- 보관 정책:
  - 최근 7일: 원본 30초 해상도
  - 7~30일: 5분 해상도로 다운샘플
  - 30~90일: 1시간 해상도
  - 90일 후: 삭제
- 분석:
  - 대시보드: avg/max CPU per service over time
  - 메모리 누수 감지: 시간 흐름에 따른 메모리 증가 추세
  - 네트워크 throughput: counter의 delta(rate) 계산

---

## 2. TSDS (Time Series Data Stream) 개념

기존 Data Stream 위에 시계열 최적화를 얹은 구조 (8.7 GA).

```
일반 Data Stream:
  필드 자유, _id 자동 생성

TSDS (`index.mode: time_series`):
  - dimension 필드로 _tsid 자동 생성 (host.name + container.id 등)
  - dimension + @timestamp가 같은 doc은 하나로 간주 (중복 제거)
  - dimension 정렬로 디스크 압축률 향상 (delta encoding)
  - downsampling 가능
  - 파티셔닝 단위 시간 범위 자동 관리 (start_time/end_time)
```

핵심 필드 종류:

| 필드 종류 | 메타 | 의미 |
|----------|------|------|
| dimension | `time_series_dimension: true` | 시리즈 식별자 (호스트, 컨테이너, 메트릭셋 등) |
| metric    | `time_series_metric: gauge\|counter` | 측정값 |
| 일반 필드 | (없음) | 자유롭게 저장 |

---

## 3. 인덱스 템플릿

```json
PUT /_component_template/metrics-mappings
{
  "template": {
    "settings": {
      "index.mode": "time_series",
      "index.routing_path": [
        "host.name",
        "service.name",
        "metricset.name"
      ],
      "index.lifecycle.name": "metrics-policy",
      "index.codec": "best_compression"
    },
    "mappings": {
      "properties": {
        "@timestamp":      { "type": "date" },
        "host.name":       { "type": "keyword", "time_series_dimension": true },
        "host.ip":         { "type": "ip" },
        "service.name":    { "type": "keyword", "time_series_dimension": true },
        "container.id":    { "type": "keyword", "time_series_dimension": true },
        "metricset.name":  { "type": "keyword", "time_series_dimension": true },

        "cpu.usage.pct":         { "type": "double", "time_series_metric": "gauge" },
        "memory.usage.bytes":    { "type": "long",   "time_series_metric": "gauge" },
        "memory.usage.pct":      { "type": "double", "time_series_metric": "gauge" },
        "memory.total.bytes":    { "type": "long",   "time_series_metric": "gauge" },
        "network.in.bytes":      { "type": "long",   "time_series_metric": "counter" },
        "network.out.bytes":     { "type": "long",   "time_series_metric": "counter" },
        "disk.read.ops":         { "type": "long",   "time_series_metric": "counter" }
      }
    }
  }
}

PUT /_index_template/metrics-template
{
  "index_patterns": ["metrics-*"],
  "data_stream": {},
  "composed_of": ["metrics-mappings"],
  "priority": 250
}
```

### 설계 근거

| 결정 | 이유 |
|------|------|
| `index.mode: time_series` | TSDS 기능 활성화 (필수) |
| `index.routing_path` | dimension 필드 중 라우팅 키. 같은 호스트의 데이터는 같은 샤드로 → 압축/스캔 효율 |
| `time_series_dimension` | 시리즈 키. 같은 dimension+timestamp의 문서는 중복으로 거부됨 |
| `gauge` (cpu.usage.pct) | 임의의 값. 평균/최대 의미 있음 |
| `counter` (network.in.bytes) | 단조 증가. 그대로 평균 내면 의미 없음 → `rate`/`serial_diff` 필요 |
| `routing_path`에 host.name 포함 | host 단위로 데이터를 한 샤드에 모아 압축 효과 |

> **주의**: TSDS에서는 `_id`가 자동으로 dimension hash + @timestamp로 결정된다. 동일 dimension+timestamp의 두 번째 문서는 거부된다.

---

## 4. ILM + Downsampling

```json
PUT /_ilm/policy/metrics-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_primary_shard_size": "50gb"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "downsample": { "fixed_interval": "5m" },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "downsample": { "fixed_interval": "1h" },
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

### Downsampling 동작

```
원본 (TSDS):  30초마다 host/cpu.usage.pct 1.5억 row/일
   ↓ ILM warm 단계 (7일 후)
다운샘플 인덱스: 5분마다 1 row, 각 metric 필드는 자동으로
                avg/min/max/sum/value_count 통계로 변환
   ↓ ILM cold 단계 (30일 후)
다운샘플 인덱스: 1시간마다 1 row
```

다운샘플된 인덱스에 쿼리하면 동일한 `aggs`(avg, max 등)가 자동으로 통계 필드를 사용해 결과를 만들어 준다 → 클라이언트 코드는 그대로.

```bash
# 수동 다운샘플 (테스트)
POST /metrics-system.cpu-default-2025.04.01-000001/_downsample/metrics-system.cpu-default-2025.04.01-000001-5m
{
  "fixed_interval": "5m"
}
```

---

## 5. 색인 (Metricbeat)

```yaml
# metricbeat.yml
metricbeat.modules:
  - module: system
    period: 30s
    metricsets:
      - cpu
      - memory
      - network
      - load

output.elasticsearch:
  hosts: ["https://es:9200"]
  api_key: "${ES_API_KEY}"
  # data stream 자동 라우팅 (8.x 기본)
```

Metricbeat는 ECS + TSDS를 자동으로 따른다. 8.x 기본 통합 사용 시 별도 매핑 작업 없이 동작.

---

## 6. 검색·집계 패턴

### 6.1 호스트별 평균 CPU 시계열

```json
GET /metrics-system.cpu-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "service.name": "payment-api" } },
        { "range": { "@timestamp": { "gte": "now-6h" } } }
      ]
    }
  },
  "aggs": {
    "by_host": {
      "terms": { "field": "host.name", "size": 20 },
      "aggs": {
        "cpu_over_time": {
          "date_histogram": {
            "field": "@timestamp",
            "fixed_interval": "1m"
          },
          "aggs": {
            "avg_cpu": { "avg": { "field": "cpu.usage.pct" } },
            "max_cpu": { "max": { "field": "cpu.usage.pct" } }
          }
        }
      }
    }
  }
}
```

### 6.2 counter 메트릭의 delta (rate) 계산

`network.in.bytes`처럼 단조 증가하는 counter는 그대로 평균을 내면 의미가 없다. `serial_diff` (pipeline aggregation)를 써서 이전 버킷과의 차이를 구한다.

```json
GET /metrics-system.network-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "host.name": "app-pod-7f9d" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "per_minute": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1m"
      },
      "aggs": {
        "in_bytes_max":  { "max": { "field": "network.in.bytes" } },
        "in_bytes_rate": {
          "serial_diff": {
            "buckets_path": "in_bytes_max",
            "lag": 1
          }
        }
      }
    }
  }
}
```

`in_bytes_rate`가 1분당 증가량(bytes/min). 60으로 나누면 bytes/sec.

### 6.3 Runtime Field로 derived metric

색인하지 않고 검색 시점에 계산되는 필드. 메모리 사용 비율을 % 단위로 보여주고 싶지만 색인은 bytes만 들어 있을 때:

```json
GET /metrics-system.memory-*/_search
{
  "runtime_mappings": {
    "memory_used_pct_calc": {
      "type": "double",
      "script": {
        "source": """
          if (doc['memory.total.bytes'].size() == 0) return;
          double total = doc['memory.total.bytes'].value;
          double used  = doc['memory.usage.bytes'].value;
          if (total > 0) emit(used / total * 100);
        """
      }
    }
  },
  "size": 0,
  "aggs": {
    "avg_used_pct": { "avg": { "field": "memory_used_pct_calc" } }
  }
}
```

> Runtime field는 색인 부담 없이 새 필드를 시도해 볼 수 있는 장점이 있지만, 매 검색마다 스크립트가 실행된다 → 자주 쓰면 인덱스 시점에 계산해 두는 편이 효율적.

### 6.4 메모리 증가 추세 감지 (moving_avg)

```json
GET /metrics-system.memory-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "service.name": "payment-api" } },
        { "range": { "@timestamp": { "gte": "now-7d" } } }
      ]
    }
  },
  "aggs": {
    "hourly": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      },
      "aggs": {
        "avg_used":     { "avg": { "field": "memory.usage.bytes" } },
        "moving_avg":   {
          "moving_fn": {
            "buckets_path": "avg_used",
            "window": 24,
            "script": "MovingFunctions.linearWeightedAvg(values)"
          }
        }
      }
    }
  }
}
```

---

## 7. 운영 팁

### 7.1 TSDS 인덱스 정보 확인

```bash
GET /metrics-system.cpu-default-2025.04.30-000001
# settings.index.time_series.start_time / end_time 가 자동 채워짐

GET /_data_stream/metrics-system.cpu-default
```

### 7.2 dimension 필드 변경

dimension은 매핑 후 변경 불가. 새로 추가하려면 인덱스 템플릿을 수정하고 다음 rollover에서 적용된다.

```bash
# 강제 rollover
POST /metrics-system.cpu-default/_rollover
```

### 7.3 다운샘플 결과 확인

```bash
GET /_cat/indices/metrics-system.cpu-*?v&h=index,docs.count,store.size

# 예시:
# index                                    docs.count store.size
# metrics-system.cpu-default-2025.04.30...   480000     1.2gb     ← 원본
# metrics-system.cpu-default-2025.04.20...   48000      150mb     ← 5m downsample
# metrics-system.cpu-default-2025.03.20...   4000       18mb      ← 1h downsample
```

### 7.4 counter 리셋 처리

`network.in.bytes`는 호스트 재기동 시 0으로 리셋된다. `serial_diff` 결과가 음수가 나오면 리셋으로 간주하고 0으로 처리:

```painless
if (currentValue < previousValue) {
  emit(0);  // counter reset
} else {
  emit(currentValue - previousValue);
}
```

Elastic Stack에서는 `rate` aggregation이 더 안전한 대안이지만, TSDS의 다운샘플 인덱스에서는 `serial_diff`가 더 흔히 쓰인다.

---

## 8. 안티패턴

### 8.1 dimension 너무 많이 잡기

```
host.name + pod.name + container.id + thread.name + ...
→ _tsid 카디널리티 폭증 → 디스크/메모리 사용량 ↑
```

dimension은 시리즈를 식별하는 최소 셋만. 보조 메타데이터는 dimension이 아닌 일반 keyword로.

### 8.2 gauge/counter 잘못 표기

`cpu.usage.pct`는 0~1 사이 비율 → gauge.
`bytes_total` 같은 누적값은 counter. 잘못 표기하면 다운샘플 결과 무의미.

### 8.3 일반 인덱스에 시계열 데이터 색인

`index.mode: time_series` 없이는 dimension 정렬도, downsample도 불가. logging이라면 일반 data stream으로 충분하지만, 메트릭은 TSDS가 정답.

---

## 9. 요약

| 항목 | 선택 |
|------|------|
| 인덱스 모드 | `time_series` (TSDS) |
| 시리즈 식별 | dimension 필드 (host, service, metricset) |
| 메트릭 타입 | `gauge` (스냅샷) / `counter` (단조증가) |
| 라이프사이클 | hot → 5m downsample → 1h downsample → delete |
| Counter rate | `serial_diff` 또는 `rate` aggregation |
| Derived metric | runtime field (가벼운 경우) |
| 트렌드 감지 | `moving_fn` + `date_histogram` |
| 색인 도구 | Metricbeat (ECS + TSDS 자동) |
