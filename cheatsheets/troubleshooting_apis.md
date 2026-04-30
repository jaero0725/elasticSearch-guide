# 트러블슈팅 API 모음

증상 → API → 해석 순으로 빠르게 찾기. 8.x / 9.x 기준.

---

## 1단계: 한눈에 보기

```
GET /_cluster/health
GET /_cat/nodes?v&h=name,node.role,heap.percent,ram.percent,cpu,load_1m,disk.used_percent
GET /_cat/indices?v&h=index,health,pri,rep,docs.count,store.size&s=store.size:desc
GET /_cat/shards?v&s=state,index | grep -v STARTED
```

이 4줄로 90%의 1차 진단이 가능하다.

---

## 클러스터 상태

| API | 보는 것 |
|-----|---------|
| `GET /_cluster/health` | green/yellow/red, 미할당 shard 수 |
| `GET /_cluster/health?level=indices` | 인덱스별 상태 |
| `GET /_cluster/health?level=shards` | shard별 상태 |
| `GET /_cluster/state?filter_path=...` | cluster state 일부 (전체는 무거움) |
| `GET /_cluster/stats` | 노드·인덱스·shard 합계 |
| `GET /_cluster/pending_tasks` | master 큐 적재 |
| `GET /_cluster/settings` | 동적·정적 설정 |

### Cluster state 크기 진단

```
GET /_cluster/state?filter_path=metadata.indices.*.mappings&human
```

비대화 신호: shard 수 폭증, 인덱스당 mapping 필드 폭증.

---

## 노드

| API | 보는 것 |
|-----|---------|
| `GET /_cat/nodes?v` | 노드 목록 + 핵심 메트릭 |
| `GET /_nodes/stats` | 모든 카테고리 (heavy) |
| `GET /_nodes/stats/jvm` | heap, GC, pool |
| `GET /_nodes/stats/fs` | 디스크 사용·IOPS |
| `GET /_nodes/stats/indices` | 색인·검색 통계 |
| `GET /_nodes/stats/thread_pool` | active/queue/rejected |
| `GET /_nodes/stats/breaker` | circuit breaker trip |
| `GET /_nodes/stats/indexing_pressure` | 색인 압박 (8.x) |
| `GET /_nodes/hot_threads` | CPU top thread stack |

### Hot threads 활용

```
GET /_nodes/hot_threads?threads=10&interval=500ms&snapshots=10
```

- `threads`: 노드당 보고할 thread 수
- `interval`: 측정 시간 (기본 500ms)
- `snapshots`: 측정 횟수

전형 패턴:

| Stack에서 보이는 함수 | 의심 |
|----------------------|------|
| `Lucene Merge Thread` | 머지 폭주, 작은 segment 많음 |
| `BulkRequest` 처리 + mapping update | dynamic mapping 폭증 |
| `aggregation collector` | 비싼 집계 (deep terms agg, cardinality) |
| `GC ConcurrentMarkSweep` 등 | heap 압박 |
| `IndexShard.refresh` | refresh 너무 잦음 |

---

## 인덱스 / Shard

| API | 보는 것 |
|-----|---------|
| `GET /_cat/indices?v` | 모든 인덱스 |
| `GET /_cat/shards?v` | 모든 shard |
| `GET /_cat/segments?v` | segment 수·크기 |
| `GET /_cat/allocation?v` | 노드별 shard·디스크 |
| `GET /_cat/recovery?v&active_only` | 복구 진행 중 |
| `GET /_cat/fielddata?v` | fielddata 사용량 |
| `GET /<idx>/_stats` | 인덱스 단위 상세 |
| `GET /<idx>/_segments` | segment 상세 |
| `GET /<idx>/_settings` | 설정 |
| `GET /<idx>/_mapping` | 매핑 |
| `GET /<idx>/_recovery` | 복구 상태 |

### Shard 미할당 진단 (핵심)

```
GET /_cluster/allocation/explain
```

특정 shard를 지정:
```json
GET /_cluster/allocation/explain
{
  "index": "logs-2026.04.30",
  "shard": 0,
  "primary": false
}
```

응답의 `node_allocation_decisions[].deciders[]`에서 거부 이유를 본다.

| Decider | 거부 사유 |
|---------|-----------|
| `disk_threshold` | 디스크 high/flood watermark |
| `same_shard` | primary와 같은 노드에 replica 안 됨 |
| `awareness` | rack/zone 분산 정책 |
| `filter` | include/exclude 설정 |
| `data_tier` | 노드 tier가 인덱스 tier_preference와 불일치 |
| `max_retry` | 5회 실패 (retry_failed 필요) |

### 강제 재시도

```
POST /_cluster/reroute?retry_failed=true
```

---

## ILM

```
GET /_ilm/status                              # ILM 자체 상태 (RUNNING/STOPPED)
GET /<idx>/_ilm/explain                       # 인덱스별 phase/step
GET /<idx>/_ilm/explain?only_errors=true      # 에러 인덱스만
POST /<idx>/_ilm/retry                        # ERROR 스텝 재시도
POST /_ilm/start                              # ILM 시작
POST /_ilm/stop                               # ILM 정지 (운영 작업 중)
```

---

## Snapshot

```
GET /_snapshot                                # 등록된 repository
GET /_snapshot/_status                        # 진행 중인 snapshot
GET /_snapshot/<repo>/_all                    # 모든 snapshot 목록
GET /_snapshot/<repo>/<name>                  # 단일 snapshot 메타
GET /_snapshot/<repo>/<name>/_status          # 진행 상세
DELETE /_snapshot/<repo>/<name>               # 삭제
```

---

## 증상별 진단 플로우

### "검색이 갑자기 느려졌다"

```
1. GET /_cluster/health
   → red/yellow면 미할당 진단으로

2. GET /_cat/nodes?v&h=name,heap.percent,cpu,load_1m
   → heap >75% 또는 CPU 폭주 노드 식별

3. GET /_nodes/hot_threads?threads=5
   → 무슨 thread가 점유?
   - search thread → 비싼 쿼리
   - merge thread  → segment 폭증
   - GC thread     → heap 압박

4. GET /_cat/thread_pool/search?v
   → queue·rejected 카운트

5. Slow log 활성 후 재현
   PUT /<idx>/_settings { "index.search.slowlog.threshold.query.warn": "5s" }

6. GET /<idx>/_segments → segments 수 확인 (수만 개면 비정상)
```

### "색인이 느려지고 429가 난다"

```
1. GET /_cat/thread_pool/write?v&h=node_name,active,queue,rejected
   → rejected 어디서 나는지

2. GET /_nodes/stats/indexing_pressure
   → memory.total.rejections 추세

3. GET /_cat/nodes?v&h=name,heap.percent,disk.used_percent
   → 디스크 watermark 임박? heap?

4. GET /_cluster/pending_tasks
   → master 큐 쌓이면 mapping update 폭주

5. GET /_cat/segments?v
   → 작은 segment 폭증

6. GET /<idx>/_settings
   → refresh_interval, translog durability 확인
```

### "디스크가 가득 찼다 (flood_stage)"

```
1. GET /_cat/allocation?v
   → 어느 노드가 한계인지

2. GET /_cat/indices?v&s=store.size:desc | head -20
   → 가장 큰 인덱스 식별

3. (오래된 인덱스 삭제 또는 ILM cold/frozen으로 이전)
   DELETE /old-index
   또는 ILM 정책 조정

4. read-only 차단 해제 (디스크 확보 후)
   PUT /_all/_settings
   { "index.blocks.read_only_allow_delete": null }

5. 영구 해결: 노드/디스크 추가, ILM delete phase 단축
```

### "OOM 또는 빈번한 GC"

```
1. GET /_nodes/stats/jvm
   → heap_used_percent, GC count·time

2. GET /_nodes/stats/breaker
   → 어떤 breaker가 trip?
   - fielddata → text 필드 fielddata 의존
   - request → 큰 aggregation
   - in_flight_requests → 큰 bulk 요청
   - parent → 전체 한계

3. GET /_cat/fielddata?v
   → 어느 필드가 fielddata를 점유?

4. (heap dump 분석. 운영에서는 -XX:+HeapDumpOnOutOfMemoryError 설정)

5. 처방:
   - dynamic: strict로 mapping 폭증 차단
   - 비싼 쿼리 식별 (slow log)
   - shard 수 줄이기
   - heap 31GB 상한 준수 (그 이상은 RAM·노드 추가)
```

### "노드 하나가 응답 안 함"

```
1. GET /_cluster/health
   → 자동 재할당 진행 중인지

2. GET /_cat/nodes?v
   → 그 노드가 보이는지

3. (해당 노드에서)
   - jstack <pid>     # JVM stack
   - top, vmstat      # OS 자원
   - dmesg | tail     # OOM kill 흔적
   - ES 로그 tail

4. 일시 격리: cluster.routing.allocation.exclude._name
   PUT /_cluster/settings
   { "transient": { "cluster.routing.allocation.exclude._name": "es-03" } }

5. 노드 재시작 또는 교체
```

---

## 운영 자주 쓰는 명령

### 인덱스 settings 동적 변경

```json
PUT /<idx>/_settings
{
  "index.refresh_interval": "30s",
  "index.number_of_replicas": 1,
  "index.translog.durability": "request"
}
```

### Cluster settings 동적 변경

```json
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low":  "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}
```

`persistent` (재시작 후 유지) vs `transient` (재시작 시 사라짐). 8.x부터 transient는 deprecated.

### 작업 일시 정지

```
POST /_cluster/reroute?dry_run     # 시뮬레이션
POST /_ilm/stop                    # ILM 일시 정지 (대규모 작업 전)
POST /_ilm/start

# Allocation 일시 정지
PUT /_cluster/settings
{ "persistent": { "cluster.routing.allocation.enable": "none" } }
# 작업 후
PUT /_cluster/settings
{ "persistent": { "cluster.routing.allocation.enable": null } }
```

### 인덱스 close/open (매핑 일부 변경 시)

```
POST /<idx>/_close
PUT  /<idx>/_settings { ... }
POST /<idx>/_open
```

### Reindex (background)

```
POST /_reindex?wait_for_completion=false
{ "source": { "index": "old" }, "dest": { "index": "new" } }

GET /_tasks?detailed=true&actions=*reindex
```

---

## 데이터 점검

```
# 문서 수 빠른 카운트
GET /<idx>/_count

# 가장 큰 문서
GET /<idx>/_search
{ "size": 1, "sort": [{ "_id": "asc" }], "_source": false }

# 매핑 필드 capabilities (필드 일관성 확인)
GET /<idx>/_field_caps?fields=*

# stats 요약
GET /<idx>/_stats?level=shards
```

---

## 빠른 참조

| 증상 | 첫 명령 |
|------|---------|
| Red 클러스터 | `_cluster/allocation/explain` |
| Yellow 지속 | `_cat/recovery`, `_cluster/allocation/explain` |
| 검색 느림 | `_nodes/hot_threads` |
| 색인 느림 | `_cat/thread_pool/write` |
| 디스크 풀 | `_cat/allocation`, `_cat/indices?s=store.size:desc` |
| Heap 압박 | `_nodes/stats/jvm`, `_nodes/stats/breaker` |
| 미할당 shard | `_cluster/allocation/explain` |
| ILM 멈춤 | `_ilm/explain?only_errors=true` |
| Master 부하 | `_cluster/pending_tasks` |
| Mapping 폭증 | `_cluster/state?filter_path=metadata.indices.*.mappings` |

---

*관련 cheatsheets: ilm_policy_templates, version_history*
