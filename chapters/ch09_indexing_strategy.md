# 9장. Indexing 전략

Elasticsearch에서 색인(indexing)은 단순히 "문서를 저장한다"가 아니다. 모든 색인 작업은 Lucene segment 생성, refresh, translog flush, merge 트리거를 동반하며, 잘못된 전략은 곧바로 indexing throughput 붕괴와 search latency 폭주로 이어진다. 이 장에서는 8.x / 9.x 기준 색인 파이프라인의 내부 동작과 운영 베스트 프랙티스를 정리한다.

---

## 9.1 단건 vs Bulk API

### 단건 INSERT의 비용

```
PUT /logs/_doc/1
{ "level": "INFO", ... }
```

이 요청 한 건마다:
- HTTP 라운드트립 1회
- 마스터 라우팅 + primary shard 라우팅 1회
- Primary indexing → replica forwarding 동기 대기
- Translog write + fsync (기본 `request` durability)
- (refresh 주기에 따라) in-memory buffer flush

**핵심**: Elasticsearch는 한 건 쓰든 1000건 쓰든 요청당 라우팅·acking 오버헤드가 거의 동일하다. 따라서 단건 색인은 처리량의 1/100 이하로 떨어진다.

### Bulk API 형식

```
POST /_bulk
{ "index" : { "_index" : "logs", "_id" : "1" } }
{ "field1" : "value1" }
{ "create" : { "_index" : "logs", "_id" : "2" } }
{ "field1" : "value2" }
{ "update" : { "_index" : "logs", "_id" : "1" } }
{ "doc" : { "field2" : "value2" } }
{ "delete" : { "_index" : "logs", "_id" : "3" } }
```

각 액션은 **두 줄 (delete는 한 줄)**, NDJSON 포맷, 마지막 줄 끝에도 개행이 와야 한다.

### 권장 페이로드 크기

공식 권장:

> "Try 5 MiB to 15 MiB. From there you can experiment to find the sweet spot."

- 너무 작으면(<1MiB): 라우팅·acking 오버헤드 비율 높음
- 너무 크면(>20MiB): coordinating node에서 메모리 압박, GC 유발, `http.max_content_length` (기본 100MiB) 한계
- **5–15MiB가 sweet spot**, 문서 수로는 보통 1,000–10,000건 (문서 크기에 따라 변동)

### 클라이언트 배칭 전략

```
[App] ─┐
       ├──→ [BulkProcessor]
       │     ├─ flushInterval: 1s
       │     ├─ bulkActions:   1000 docs
       │     └─ bulkSize:      10MiB
       │              ↓
[App] ─┘     [Bulk Request to ES]
```

자바 공식 클라이언트의 `BulkProcessor` (또는 새 클라이언트의 `BulkIngester`)는 위 세 조건 중 **하나라도 충족하면 flush**한다. 어플리케이션 레벨에서 직접 구현할 때도 동일한 패턴이 정석이다.

### 부분 실패 처리

Bulk 응답은 항목별 성공/실패가 섞여 올 수 있다.

```json
{
  "took": 30,
  "errors": true,
  "items": [
    { "index": { "_id": "1", "status": 201 } },
    { "index": { "_id": "2", "status": 429, "error": { "type": "es_rejected_execution_exception" } } }
  ]
}
```

**`errors: true` 면 반드시 항목별로 순회**하고, 429 (queue full)이나 503은 exponential backoff로 재시도해야 한다. 전체 응답 status가 200이라고 안심하면 데이터 유실로 직결된다.

---

## 9.2 Refresh interval과 NRT search trade-off

### Near Real Time의 정체

Lucene은 segment 단위로 검색한다. 새 문서를 색인하면 in-memory buffer에 쌓이고, **refresh** 시점에 새 segment가 만들어져 검색 가능해진다.

```
[색인]  [in-memory buffer]
   ↓             ↓ (1s 후 refresh)
   ↓     [새 segment - 검색 가능]
   ↓             ↓
[다음 refresh마다 segment 누적] ──→ background merge
```

**기본 `refresh_interval = 1s`**: 색인 후 최대 1초 내에 검색에 노출. 이것이 "Near Real Time"의 실체.

### Refresh 비용

매 refresh마다:
- 새 segment 파일 (FST, doc values, postings 등) 생성
- `IndexReader` 재오픈
- Page cache에 신규 파일 적재

소량 색인이 자주 발생하면 작은 segment가 폭증하고, merge가 그를 따라잡느라 CPU·I/O를 잡아먹는다.

### Trade-off

| `refresh_interval` | 검색 신선도 | indexing throughput | segment 수 |
|--------------------|-------------|---------------------|------------|
| `1s` (기본)        | 1초 내      | 기준                | 많음       |
| `30s`              | 30초 내     | +30~50%            | 줄어듦     |
| `-1` (비활성)      | 수동 refresh 시 | +최대 2~3배   | 최소화     |

### 운영 패턴

**시계열 / 로그 (검색 신선도 30s~1m 허용)**:
```
PUT /logs-2026.04.30/_settings
{ "refresh_interval": "30s" }
```

**대량 backfill / reindex**:
```
PUT /target/_settings
{ "refresh_interval": "-1", "number_of_replicas": 0 }
... (bulk indexing) ...
PUT /target/_settings
{ "refresh_interval": "1s", "number_of_replicas": 1 }
POST /target/_forcemerge?max_num_segments=1
```

색인 중 refresh를 끄고 replica를 0으로 두면 처리량이 극적으로 향상된다 (replica 색인을 나중에 한 번에 복제). 단, 색인 중 노드 장애 시 데이터 유실 가능성을 감수해야 한다.

---

## 9.3 Translog: durability와 sync_interval

### Translog의 역할

Lucene의 `commit` (디스크 fsync)은 비싸므로 매 색인마다 수행할 수 없다. 대신 Elasticsearch는 **translog**라는 WAL을 운영한다.

```
[색인 요청]
    ↓
[in-memory buffer] ──refresh──→ [새 segment (메모리 매핑)]
    ↓
[translog write]   ─── 장애 시 replay 소스
    ↓
[response to client]
```

장애 발생 시: 마지막 Lucene commit 이후의 translog를 replay해서 메모리 buffer를 복원.

**Flush** = Lucene commit + translog truncate. 기본 30분마다 또는 translog 크기가 `index.translog.flush_threshold_size` (기본 512MiB) 도달 시.

### Durability 설정

```yaml
# 기본값
index.translog.durability: request
index.translog.sync_interval: 5s
```

| 설정 | 동작 | 데이터 유실 가능 범위 |
|------|------|----------------------|
| `request` (기본) | 매 색인 요청마다 translog fsync. 응답 후 노드 장애에도 안전 | 0 (응답 받은 데이터) |
| `async`  | `sync_interval` 마다 fsync. 그 사이 색인은 "응답 OK"여도 유실 가능 | 최대 `sync_interval` (기본 5s) |

### 언제 async를 쓰는가

**`request` 유지가 원칙**. Async는 다음 조건 모두를 만족할 때만 고려:

- 로그/메트릭 등 **소량 유실이 비즈니스에 무해**
- 색인 throughput이 명백한 병목
- 디스크 fsync IOPS가 한계 (예: NFS, 저성능 EBS)

```
PUT /logs-*/_settings
{
  "index.translog.durability": "async",
  "index.translog.sync_interval": "30s"
}
```

이렇게 하면 throughput은 보통 20~50% 향상되지만, 노드 크래시 시 최대 30초 분량의 색인이 사라질 수 있다.

---

## 9.4 Index buffer size, flush

### Indexing buffer

```yaml
# elasticsearch.yml (노드 단위)
indices.memory.index_buffer_size: 10%   # heap의 10%, 기본
indices.memory.min_index_buffer_size: 48mb
```

이 buffer는 **노드 전체의 모든 active shard가 공유**한다. 한 노드에 active shard가 많으면 shard당 버퍼는 작아지고, 작은 segment가 자주 만들어진다.

> Heap 32GB × 10% = 3.2GB 버퍼. Active shard 100개면 shard당 32MB. → 32MB 차면 강제 flush → segment 작음.

### Flush 트리거

1. Translog 크기가 `flush_threshold_size` 초과 (기본 512MiB)
2. `flush_interval` 경과 (기본 없음, translog 크기로만 결정)
3. `_flush` API 명시적 호출
4. 인덱스 close 시

Flush가 너무 자주 발생하면(translog가 작은 경우) 디스크 I/O 폭증. 너무 드물면(translog가 큰 경우) 노드 재시작 시 replay 시간이 길어진다.

### Force flush (운영 명령)

```
POST /my-index/_flush
```

**일반 운영에서는 호출하지 마라.** Elasticsearch가 알아서 한다. 사용 케이스: snapshot 직전, 노드 재시작 직전 정도.

---

## 9.5 ILM: Index Lifecycle Management

### Phase 모델

```
[hot]  ──→  [warm]  ──→  [cold]  ──→  [frozen]  ──→  [delete]
 활발한      읽기 위주     드물게 읽음    거의 안 읽음    삭제
 색인+검색   검색만        검색 가능      searchable
                                        snapshot
```

| Phase | 노드 hardware tier | 일반적 보존 | Replica | 비고 |
|-------|---------------------|-------------|---------|------|
| hot   | NVMe SSD, 고RAM    | 1-7일       | 1+      | rollover로 진입 |
| warm  | SSD                 | 7-30일      | 1       | force_merge, shrink 가능 |
| cold  | HDD or 저비용 SSD   | 30-90일     | 0-1     | searchable snapshot 옵션 |
| frozen | object store mount | 90일~수년   | -       | 100% searchable snapshot, partial mount |
| delete | -                  | -           | -       | API로 삭제 |

### ILM 정책 예시

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_primary_shard_size": "50gb",
            "max_docs": 200000000
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink": { "number_of_shards": 1 },
          "allocate": { "include": { "data_tier": "data_warm" } },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
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
        "actions": { "delete": {} }
      }
    }
  }
}
```

`min_age`는 **rollover 시점부터 측정**된다 (해당 인덱스가 write 인덱스에서 빠진 시점). 색인 생성 시점이 아님에 주의.

### Frozen tier의 정체

Frozen은 단순한 cold가 아니다. 데이터를 object storage(S3 등)에 두고 노드는 캐시만 가진다. 검색은 가능하지만 latency는 cold보다 한 자릿수 느릴 수 있다 (수 초 단위). 보존 비용은 hot 대비 1/10 이하.

---

## 9.6 Data Stream: 시계열 표준

### Index alias의 한계

전통적 패턴:
```
logs-000001, logs-000002, ... → write alias: logs-write
                              → read alias:  logs-search
```

수동으로 rollover API를 호출하고, alias를 업데이트해야 했다.

### Data Stream

Data stream은 위 패턴을 **빌트인으로 자동화**한다. 사용자에게는 단일 이름(`logs`)으로 보이지만, 내부적으로 `.ds-logs-2026.04.30-000001` 같은 backing index들이 자동 관리된다.

```
              ┌──────────────────────────────────┐
   logs  ───→ │ .ds-logs-...-000001  (closed)    │
   (data      │ .ds-logs-...-000002  (closed)    │
    stream)   │ .ds-logs-...-000003  (write)  ←──┼── 모든 색인은 여기로
              └──────────────────────────────────┘
```

### 제약

- **모든 문서에 `@timestamp` 필드 필수**
- Append-only — `_id`로 update/delete 불가 (단, `_update_by_query`, `_delete_by_query`는 가능)
- 인덱스 템플릿 + ILM 정책과 한 세트로 동작

### 생성 흐름

```
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-policy",
      "number_of_shards": 1,
      "number_of_replicas": 1
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

POST /logs-app/_doc
{ "@timestamp": "2026-04-30T10:00:00Z", "level": "INFO", "message": "..." }
```

첫 색인 시점에 data stream과 backing index가 자동 생성된다.

---

## 9.7 Rollover: 자르는 기준

ILM rollover action 또는 수동 `_rollover` API 모두 동일한 조건을 사용한다.

| 조건 | 의미 | 권장 값 |
|------|------|---------|
| `max_age` | 인덱스 생성 후 경과 시간 | `1d` ~ `7d` |
| `max_primary_shard_size` | primary shard 1개의 크기 | `50gb` (권장) |
| `max_primary_shard_docs` | primary shard 1개의 문서 수 | 워크로드별 |
| `max_docs` | 인덱스 전체 문서 수 (deprecated 추세) | - |
| `max_size` | 인덱스 전체 크기 | shard 단위 권장으로 대체 |

**여러 조건 중 하나라도 충족하면 rollover.**

### 왜 50GB?

- Lucene segment 머지 비용은 segment 크기에 비례
- 너무 큰 shard는 force_merge·snapshot·복구 시간이 길어짐
- 50GB는 경험칙으로 검증된 sweet spot
- 8.x부터 `max_primary_shard_size`가 권장 1차 조건

### 단일 인덱스 < 단일 shard ≤ 50GB

```
1일치 데이터 = 100GB, primary shard 1
→ shard당 100GB. 너무 큼.

1일치 데이터 = 100GB, primary shard 2 + max_primary_shard_size: 50gb
→ shard당 50GB. OK.
```

shard 수와 rollover 기준을 같이 설계해야 한다.

---

## 9.8 Force merge: 언제 해야 하는가

### Merge의 자동화

Lucene은 background merge로 작은 segment를 큰 것으로 합친다. **운영자가 개입할 필요는 거의 없다.**

### Force merge의 효과

```
POST /logs-2026.04.20/_forcemerge?max_num_segments=1
```

- 모든 segment를 1개(또는 N개)로 강제 병합
- 삭제된 문서 (`_seq_no`로 마스크된) 영구 제거 → 디스크 회수
- Segment 수 감소 → 검색 시 segment 순회 비용 감소

### 절대 하지 말아야 할 때

**활발히 색인되는 인덱스에 force_merge 금지.**

이유:
- Merge가 진행되는 동안 새 segment가 계속 생김 → 끝없는 merge 반복
- 디스크가 일시적으로 2배로 사용됨 (원본 + 병합본 동시 존재)
- 거대한 단일 segment가 생기면 이후 다른 segment와의 merge가 사실상 불가 → 영원히 그 상태로 유지
- I/O·CPU 폭주로 검색 latency 폭주

### 해야 할 때

**read-only로 전환된 인덱스에 한해**:

```
# 1. write 중단 확인 (rollover 후 backing index)
# 2. read-only 전환
PUT /logs-2026.04.20/_settings
{ "index.blocks.write": true }

# 3. force merge
POST /logs-2026.04.20/_forcemerge?max_num_segments=1&wait_for_completion=false
```

ILM의 warm phase에서 자동으로 처리되도록 두는 것이 정석이다.

---

## 9.9 Bulk indexing 베스트 프랙티스 체크리스트

| 항목 | 권장 |
|------|------|
| Bulk 페이로드 크기 | 5–15 MiB |
| 클라이언트 배칭 | flush by size + count + interval 동시 적용 |
| Refresh interval | 운영 워크로드는 30s, backfill은 -1 |
| Replica | 대량 backfill 시 0으로, 완료 후 복구 |
| Translog durability | request (기본 유지). async는 의식적 선택 |
| Mapping | 명시 정의 + dynamic: strict 또는 false |
| `_id` 자동 생성 | 가능하면 ES가 생성하게 (auto-id 경로 최적화 있음) |
| Bulk 응답 | `errors: true` 항목별 처리 + 429 backoff 재시도 |
| Primary shard size | 10–50 GB 목표, max 50 GB |
| Shard 수 | (총 데이터 / 50GB) + replica 고려, 노드당 shard 수 절제 |
| Force merge | active 인덱스 금지, ILM warm에서 자동화 |
| Document 크기 | 큰 nested 배열·base64 binary 회피 |
| Update vs Insert | append-only가 가능하면 update 회피 |
| Pipeline (ingest) | CPU 비용 감안, Logstash/Beats로 외주 검토 |
| 모니터링 | `indexing_pressure`, `bulk` thread_pool rejected 카운터 |

---

## 9.10 Key Takeaways

| 항목 | 핵심 |
|------|------|
| **Bulk API** | 5–15MiB, NDJSON, 항목별 결과 검사 필수 |
| **Refresh interval** | NRT 비용. 신선도 요구에 맞게 늘려라 |
| **Translog** | `request` durability가 기본·정석. `async`는 의식적 선택 |
| **ILM** | hot→warm→cold→frozen→delete 표준 라이프사이클 |
| **Data Stream** | 시계열의 표준 진입점. `@timestamp` 필수 |
| **Rollover** | `max_primary_shard_size: 50gb` 우선 |
| **Force merge** | read-only 인덱스 한정. active 인덱스에 절대 금지 |
| **Backfill 시 settings** | `refresh_interval: -1`, `number_of_replicas: 0` |

---

*다음 장: 10장. 안티패턴*
