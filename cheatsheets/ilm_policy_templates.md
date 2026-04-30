# ILM 정책 템플릿

로그·메트릭·이벤트 데이터의 라이프사이클 표준 템플릿. 8.x / 9.x 기준.

---

## Phase별 hardware tier 매핑

```
┌──────────┬───────────────────┬──────────────┬──────────────┐
│  Phase   │   Tier            │  Storage     │  Replica     │
├──────────┼───────────────────┼──────────────┼──────────────┤
│  hot     │  data_hot         │  NVMe SSD    │  1+          │
│  warm    │  data_warm        │  SSD         │  1           │
│  cold    │  data_cold        │  SSD/HDD     │  0-1 (or SS) │
│  frozen  │  data_frozen      │  Object cache│  - (SS only) │
│  delete  │  -                │  -           │  -           │
└──────────┴───────────────────┴──────────────┴──────────────┘
```

`data_hot`, `data_warm` 등은 노드 role. `node.roles: [data_hot, ...]`.

---

## 템플릿 1: 로그 (1주 hot / 30일 warm / 90일 cold / 1년 frozen)

```json
PUT _ilm/policy/logs-policy
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
          "set_priority": { "priority": 50 },
          "allocate":     { "include": { "_tier_preference": "data_warm" } },
          "shrink":       { "number_of_shards": 1 },
          "forcemerge":   { "max_num_segments": 1 },
          "readonly":     {}
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": { "priority": 0 },
          "allocate": { "include": { "_tier_preference": "data_cold" } },
          "searchable_snapshot": { "snapshot_repository": "my_repo" }
        }
      },
      "frozen": {
        "min_age": "90d",
        "actions": {
          "searchable_snapshot": { "snapshot_repository": "my_repo" }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": { "delete_searchable_snapshot": true }
        }
      }
    }
  }
}
```

### 핵심 포인트

- `min_age`는 **rollover 시점부터** 측정 (인덱스 생성 시점 아님)
- `forcemerge`는 반드시 warm phase 이후 (read-only 전환 후)
- `searchable_snapshot`은 cold/frozen에서만. 한번 적용된 인덱스는 다시 hot/warm 못 감
- `delete_searchable_snapshot: true`는 snapshot까지 함께 삭제

---

## 템플릿 2: 메트릭 (단기 보존)

```json
PUT _ilm/policy/metrics-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_primary_shard_size": "30gb"
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "forcemerge":   { "max_num_segments": 1 },
          "readonly":     {},
          "allocate": { "include": { "_tier_preference": "data_warm,data_hot" } }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

- 메트릭은 시간 경과에 따라 가치 빠르게 감소 → 30일 보존
- `_tier_preference: "data_warm,data_hot"` — warm 노드 우선, 없으면 hot

---

## 템플릿 3: 감사 로그 (장기 보존, 변경 불가)

```json
PUT _ilm/policy/audit-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_age": "7d", "max_primary_shard_size": "50gb" }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": { "snapshot_repository": "audit_repo" }
        }
      },
      "frozen": {
        "min_age": "180d",
        "actions": {
          "searchable_snapshot": { "snapshot_repository": "audit_repo" }
        }
      },
      "delete": {
        "min_age": "2555d",
        "actions": { "delete": { "delete_searchable_snapshot": false } }
      }
    }
  }
}
```

- 7년 보존 (2555일), snapshot은 삭제 후에도 보존 (`delete_searchable_snapshot: false`)
- 별도 snapshot repository (격리, 다른 권한)

---

## Index template + Data stream 결합

ILM 정책은 인덱스 템플릿을 통해 적용된다.

```json
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "priority": 100,
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-policy",
      "index.number_of_shards": 3,
      "index.number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message":    { "type": "text" },
        "level":      { "type": "keyword" }
      }
    }
  }
}
```

첫 색인 시:
```
POST /logs-app/_doc
{ "@timestamp": "...", "level": "INFO", "message": "..." }
```

자동으로 `.ds-logs-app-...-000001` 생성, ILM 정책 적용 시작.

---

## ILM 진단

### 정책 적용 상태

```
GET /logs-*/_ilm/explain
```

```json
{
  "indices": {
    ".ds-logs-app-2026.04.30-000001": {
      "index": "...",
      "managed": true,
      "policy":  "logs-policy",
      "phase":   "hot",
      "action":  "rollover",
      "step":    "check-rollover-ready",
      "phase_time_millis": 1714435200000,
      "step_info": {
        "message": "Waiting for rollover conditions"
      }
    }
  }
}
```

| 자주 보는 step | 의미 |
|----------------|------|
| `check-rollover-ready` | rollover 조건 대기 중 |
| `set-indexing-complete` | rollover 후 write 차단 |
| `wait-for-shard-history-leases` | leases 만료 대기 (warm 진입 전) |
| `forcemerge` | force_merge 진행 중 |
| `mount-snapshot` | searchable snapshot mount |
| `complete` | 마지막 단계 도달 |
| `ERROR` | 진단 필요 |

### 에러 복구

```
POST /logs-*/_ilm/retry
```

`step_info.message`가 일시적 장애 (디스크, 네트워크) 때문이라면 retry로 진행. 영구 문제(설정 오류)면 정책 수정 후 retry.

### Move 강제

```
POST /_ilm/move/.ds-logs-app-2026.04.30-000001
{
  "current_step": { "phase": "hot",  "action": "rollover", "name": "check-rollover-ready" },
  "next_step":    { "phase": "warm", "action": "complete",  "name": "complete" }
}
```

긴급 시. 일반 운영에서는 사용하지 마라.

---

## 흔한 실수

| 실수 | 결과 | 처방 |
|------|------|------|
| `min_age`를 인덱스 생성 시점 기준으로 오해 | rollover 안 되면 영영 다음 phase로 안 감 | rollover action 포함 확인 |
| `forcemerge`를 hot phase에 | active 인덱스 머지 폭주 | warm 이후로 |
| `shrink` 후 `forcemerge` 순서 거꾸로 | shrink 실패 또는 효과 없음 | shrink → forcemerge |
| searchable_snapshot 후 hot/warm으로 되돌림 | 정책 거부 | 양방향 불가, 정책 재설계 |
| `delete_searchable_snapshot` 미지정 | snapshot이 영원히 남아 비용 누적 | 명시 (보통 true) |
| ILM 적용했는데 동작 안 함 | priority 낮음 또는 index template 미매칭 | template `priority`, `index_patterns` 확인 |

---

## 모니터링 체크 항목

```
# 정책 적용 인덱스 현황
GET /_ilm/status
GET /_cat/indices?v&h=index,health,docs.count,store.size,creation.date.string&s=creation.date

# 스텝 에러
GET /*/_ilm/explain?only_errors=true

# Snapshot repository 용량
GET /_snapshot/my_repo/_status
```

---

## 결정 가이드

```
보존 기간이 며칠?
├─ 1-30일 → hot + delete (warm 생략 가능)
├─ 30-90일 → hot + warm + delete
├─ 90일-1년 → hot + warm + cold(SS) + delete
└─ 1년+ → hot + warm + cold(SS) + frozen(SS) + delete

검색이 7일 후에도 빈번한가?
├─ Yes → warm phase 길게, replica 1
└─ No → 30일 후 cold(SS), replica 0

쓰기 비율 대비 읽기 비율?
├─ 쓰기 >> 읽기 → primary shard 늘리기, replica 줄이기
└─ 읽기 >> 쓰기 → replica 늘리기 (단 색인 throughput trade-off)
```

---

*관련 cheatsheets: troubleshooting_apis, version_history*
