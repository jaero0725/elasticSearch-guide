# 11장. 모니터링과 트러블슈팅

운영 중인 Elasticsearch 클러스터는 살아있는 시스템이다. CPU·heap·디스크·네트워크가 끊임없이 상호작용하고, 한 노드의 문제가 다른 노드의 증상으로 전파된다. 이 장에서는 진단의 핵심 API들과 증상별 트러블슈팅 플로우를 정리한다.

---

## 11.1 Cluster health: green / yellow / red

### 의미

```
GET /_cluster/health
```

```json
{
  "cluster_name": "prod",
  "status": "yellow",
  "number_of_nodes": 6,
  "active_primary_shards": 120,
  "active_shards": 235,
  "unassigned_shards": 5,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 0,
  "task_max_waiting_in_queue_millis": 0
}
```

| 상태 | 의미 | 즉시 행동 |
|------|------|-----------|
| **green** | 모든 primary + replica 할당 완료 | 없음 |
| **yellow** | 모든 primary 할당, 일부 replica 미할당 | 5분 내 자동복구 안 되면 진단 |
| **red** | **일부 primary 미할당** | 즉시 대응. 데이터 가용 불가 |

### 노드 추가/재시작 직후의 yellow

기본 `index.unassigned.node_left.delayed_timeout` = 1m. 노드가 1분 내 복귀하면 shard를 그대로 두지만, 그 이상이면 다른 노드로 재할당을 시작한다. 재할당은 데이터 크기에 따라 분~시간 단위.

### Red의 의미

**Primary가 미할당이면 그 인덱스의 일부 데이터는 검색·색인 불가**. 단, 다른 인덱스는 정상 작동한다. red ≠ "클러스터 다운".

### 인덱스 단위 health

```
GET /_cluster/health?level=indices
GET /_cluster/health?level=shards
```

문제 인덱스를 빠르게 좁힐 수 있다.

---

## 11.2 _cat APIs: 빠른 인스펙션

`_cat` API는 사람이 읽기 쉬운 표 형식. 운영자가 가장 자주 치는 명령들.

### Nodes

```
GET /_cat/nodes?v&h=name,node.role,heap.percent,ram.percent,cpu,load_1m,disk.used_percent
```

```
name     node.role  heap.percent  ram.percent  cpu  load_1m  disk.used_percent
es-01    cdfhirstw  68            92           45   2.10     67.3
es-02    cdfhirstw  72            93           48   2.34     68.1
es-03    cdfhirstw  85            93           62   3.01     71.8   ← heap 임계
```

`node.role` 글자: `m`=master eligible, `d`=data, `i`=ingest, `c`=coordinating, `h`=hot, `w`=warm, `s`=cold, `f`=frozen 등.

### Indices

```
GET /_cat/indices?v&h=index,health,status,pri,rep,docs.count,store.size,pri.store.size&s=store.size:desc
```

```
index               health  pri  rep  docs.count  store.size  pri.store.size
logs-2026.04.30-1   green   3    1    50000000    120gb       60gb
logs-2026.04.29-1   green   3    1    48000000    115gb       57gb
```

### Shards

```
GET /_cat/shards?v&h=index,shard,prirep,state,docs,store,node
```

`state`:
- `STARTED`: 정상
- `RELOCATING`: 노드 간 이동 중
- `INITIALIZING`: 복구·할당 중
- `UNASSIGNED`: 미할당 (원인 진단 필요)

### Allocation

```
GET /_cat/allocation?v
```

```
shards  disk.indices  disk.used  disk.avail  disk.total  disk.percent  node
120     1.2tb         1.5tb      0.5tb       2tb         75            es-01
118     1.18tb        1.49tb     0.51tb      2tb         74            es-02
```

노드 간 균형 확인 + 디스크 watermark 임박 여부.

### Thread pool

```
GET /_cat/thread_pool/write,search,get?v&h=node_name,name,active,queue,rejected
```

```
node_name  name    active  queue  rejected
es-01      write   8       0      0
es-01      search  12      0      0
es-02      write   8       50     1234       ← 색인 거부 발생
```

`rejected`가 0이 아니면 클라이언트 측에서 재시도 백오프 + 부하 조정 필요.

---

## 11.3 _nodes/stats: 깊이 있는 메트릭

### 핵심 카테고리

```
GET /_nodes/stats/jvm,fs,indices,thread_pool,os
```

| 카테고리 | 보는 것 |
|----------|---------|
| `jvm`    | heap_used_percent, GC count·time, pool별 사용량 |
| `fs`     | total/free/available, IO stats |
| `indices`| docs, store size, indexing/search stats, segments, fielddata, query_cache |
| `thread_pool` | active, queue, rejected, completed |
| `os`     | cpu, load_average, mem, swap |

### 자주 쓰는 부분

```
GET /_nodes/stats/jvm
```

```json
"jvm": {
  "mem": {
    "heap_used_percent": 78,
    "heap_max_in_bytes": 33285996544,
    "pools": {
      "old": { "used_in_bytes": 21000000000, "max_in_bytes": 25000000000 }
    }
  },
  "gc": {
    "collectors": {
      "young": { "collection_count": 12345, "collection_time_in_millis": 567000 },
      "old":   { "collection_count": 23,    "collection_time_in_millis": 89000  }
    }
  }
}
```

### 색인 압박 (indexing_pressure)

8.x에 추가된 핵심 보호 메커니즘.

```
GET /_nodes/stats/indexing_pressure
```

```json
"indexing_pressure": {
  "memory": {
    "current": { "combined_coordinating_and_primary_in_bytes": 12345 },
    "total": {
      "rejections": {
        "coordinating": 0,
        "primary":      5,
        "replica":      0
      }
    }
  }
}
```

`rejections`가 늘면 클라이언트가 부하를 줄이거나 노드 추가 필요.

---

## 11.4 Hot threads API

"이 노드 CPU가 90%다" 라는 알람이 왔을 때 가장 먼저 쳐야 할 명령.

```
GET /_nodes/hot_threads?threads=10&interval=500ms
```

CPU를 가장 많이 쓰는 thread 10개의 stack trace. 흔한 패턴:

- `Lucene Merge Thread`: 머지 폭주 → force_merge 또는 작은 segment 폭증 의심
- `[search]` thread가 query parsing/aggregation에 머무름: 비싼 쿼리
- `[write]` thread가 mapping update에 머무름: dynamic mapping 폭증
- `GC` 관련 thread: heap 압박

```
::: {es-01}{...}
   Hot threads at 2026-04-30T10:00:00Z, interval=500ms, busiestThreads=10:

   86.5% (432.5ms out of 500ms) cpu usage by thread 'elasticsearch[es-01][search][T#7]'
     10/10 snapshots sharing following 32 elements
       org.apache.lucene.search.IndexSearcher.searchAfter(...)
       ...
```

여러 노드를 비교하려면 `?ignore_idle_threads=false`도.

---

## 11.5 Pending tasks

```
GET /_cluster/pending_tasks
```

Cluster state 변경(매핑 추가, 인덱스 생성, shard 할당)이 master에서 큐잉된 항목. 평소 비어 있어야 한다.

```json
{
  "tasks": [
    {
      "insert_order": 101,
      "priority": "URGENT",
      "source": "shard-started ...",
      "time_in_queue_millis": 8123
    }
  ]
}
```

쌓여 있다면:
- Master 노드 부하·heap 부족
- Cluster state 비대화 (shard 너무 많음)
- 매핑/인덱스 폭발적 생성

---

## 11.6 Slow log: 검색·색인 느린 쿼리

### 활성화

```
PUT /my-index/_settings
{
  "index.search.slowlog.threshold.query.warn":  "10s",
  "index.search.slowlog.threshold.query.info":  "5s",
  "index.search.slowlog.threshold.fetch.warn":  "1s",
  "index.indexing.slowlog.threshold.index.warn":"10s",
  "index.indexing.slowlog.threshold.index.info":"5s"
}
```

8.x부터 slowlog는 JSON 형식으로 `logs/<cluster>_index_search_slowlog.json`에 기록.

### Query phase vs Fetch phase

검색은 두 단계:
- **Query**: 각 shard에서 top N의 `_id` + score 수집
- **Fetch**: `_id`로 실제 `_source` 가져오기

Slow log를 둘 다 봐야 한다. Fetch가 느리면 큰 `_source`, `script_fields`, `highlight`가 의심.

---

## 11.7 JVM heap 진단

### Heap 압박의 신호

```
GET /_nodes/stats/jvm
```

다음 중 하나라도 해당하면 위험:

- `heap_used_percent > 75%` 지속
- Old generation collection이 자주 발생 (분당 수 회 이상)
- Old GC time이 누적 시간의 5% 초과
- `circuit_breaker` triggered 카운터 증가

### Circuit breaker

```
GET /_nodes/stats/breaker
```

```
"breakers": {
  "request":    { "limit_size_in_bytes": ..., "tripped": 0 },
  "fielddata":  { "limit_size_in_bytes": ..., "tripped": 12 },
  "in_flight_requests": { "tripped": 0 },
  "parent":     { "limit_size_in_bytes": ..., "tripped": 3 }
}
```

`tripped > 0`이면 **OOM 직전에 ES가 요청을 거부**한 것. fielddata circuit breaker가 자주 발동하면 text 필드의 fielddata 사용 또는 keyword/doc_values 미사용 의심.

### Heap 늘리기 전에

8.x default는 노드 RAM에 따라 자동 결정 (보통 RAM/2, 31GB 상한). 늘리기 전에:
1. 진짜 메모리가 부족한가? (cardinality 폭증, fielddata, 큰 query) — hot threads, dump
2. Cluster state가 큰가? (shard 폭증)
3. 검색 쿼리가 비싼가? (deep agg, deep paging)
4. mapping의 fields 수 (`_field_caps`)

해결 후에도 부족하면 RAM 추가. **31GB는 절대 넘기지 마라** (10.11 참고).

---

## 11.8 Disk watermarks

```yaml
# 기본값
cluster.routing.allocation.disk.watermark.low:        85%
cluster.routing.allocation.disk.watermark.high:       90%
cluster.routing.allocation.disk.watermark.flood_stage:95%
```

| Watermark | 동작 |
|-----------|------|
| **low (85%)**       | 새 shard를 이 노드에 할당하지 않음 |
| **high (90%)**      | 기존 shard를 다른 노드로 재할당 시도 |
| **flood_stage (95%)**| **모든 인덱스를 read-only로 전환** (`index.blocks.read_only_allow_delete: true`) |

### Flood stage 발생 시

```
PUT /my-index/_settings
{ "index.blocks.read_only_allow_delete": null }
```

디스크 확보 후에만. 디스크가 여전히 95% 위면 다시 발동한다. 영구 해결은:
- 오래된 인덱스 삭제 (ILM delete)
- 노드 추가
- 디스크 확장
- ILM cold/frozen tier 이전

### 절대값 설정

```
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "200gb",
    "cluster.routing.allocation.disk.watermark.high": "100gb",
    "cluster.routing.allocation.disk.watermark.flood_stage": "50gb"
  }
}
```

큰 디스크에서는 5%가 너무 큰 절대 공간이라 절대값(`gb`)이 더 합리적.

---

## 11.9 Shard 미할당 진단

### 핵심 명령

```
GET /_cluster/allocation/explain
```

응답에 **왜 할당이 안 되었는지** 명시된다.

```json
{
  "index": "logs-2026.04.30",
  "shard": 0,
  "primary": false,
  "current_state": "unassigned",
  "unassigned_info": {
    "reason": "NODE_LEFT",
    "at": "2026-04-30T09:55:00Z"
  },
  "can_allocate": "no",
  "allocate_explanation": "...",
  "node_allocation_decisions": [
    {
      "node_name": "es-03",
      "node_decision": "no",
      "deciders": [
        {
          "decider": "disk_threshold",
          "decision": "NO",
          "explanation": "the node is above the high watermark..."
        }
      ]
    }
  ]
}
```

### 흔한 원인

| `reason` | 의미 | 처방 |
|----------|------|------|
| `INDEX_CREATED` | 신규 인덱스, 아직 할당 못 받음 | 시간 기다림 또는 disk/awareness 확인 |
| `NODE_LEFT` | 노드 이탈 | 노드 복귀 또는 `cluster.routing.allocation.enable: all` |
| `ALLOCATION_FAILED` | 5회 재시도 모두 실패 | `POST /_cluster/reroute?retry_failed=true` |
| `CLUSTER_RECOVERED` | 마스터 변경/재시작 후 | 자동 복구 진행 중 |
| `REROUTE_CANCELLED` | 재할당 취소됨 | 진행 중 작업 종료 후 자동 |

### 강제 재시도

```
POST /_cluster/reroute?retry_failed=true
```

5회 한도를 초과한 shard들의 재시도 카운터를 리셋.

### 위험한 명령

```
POST /_cluster/reroute
{ "commands": [{ "allocate_empty_primary": { ... } }] }
```

**빈 primary를 강제 할당**한다. 데이터가 사라진다. 절대 마지막 수단.

---

## 11.10 증상별 진단 플로우

### "검색이 갑자기 느려졌다"

```
1. _cluster/health      → red? yellow? unassigned shard?
2. _cat/nodes?v         → 어떤 노드가 heap·CPU 높은가
3. _nodes/hot_threads   → 무슨 일을 하고 있나
4. _cat/thread_pool/search?v → queue 쌓임? rejected?
5. slow log             → 어떤 쿼리가 느린가
6. _cat/indices?v       → segments.count 비정상?
7. _nodes/stats/breaker → fielddata trip?
```

자주 나오는 원인: 비싼 쿼리(deep agg, leading wildcard), segments 폭증, GC 압박, 다른 인덱스의 색인 부하 간섭.

### "색인이 느려졌다 / 429 폭주"

```
1. _cat/thread_pool/write?v → queue/rejected
2. _nodes/stats/indexing_pressure → 거부 카운트
3. _cat/nodes?v → heap, CPU, disk
4. _cat/allocation?v → 디스크 watermark 임박?
5. _cluster/pending_tasks → master 큐 쌓임 (mapping 폭증?)
6. hot_threads          → merge 폭주? mapping update?
7. _cat/segments?v      → segments 수 비정상?
```

원인: bulk 페이로드 너무 큼/작음, refresh_interval 너무 짧음, mapping explosion, merge 폭주, 디스크 fsync 병목.

### "OOM이 발생했다"

```
1. heap dump 분석 (운영 시 자동 dump 활성화)
2. _nodes/stats/breaker → 어떤 breaker?
3. _cat/fielddata?v     → text 필드의 fielddata 의존?
4. cluster state 크기 (?human&filter_path=...)
5. mapping의 fields 수, dynamic 정책
6. 비싼 query/aggregation 패턴 식별
```

처방: shard·인덱스 수 줄이기, dynamic: strict, fielddata 제거, query 패턴 수정, 노드 RAM 증설.

---

## 11.11 Kibana Stack Monitoring

코드로 진단하기 전, Stack Monitoring UI는 **시각화된 시계열로 같은 정보를 더 빠르게 보여준다**.

| 화면 | 핵심 정보 |
|------|-----------|
| Overview      | 클러스터 status, 노드 수, 인덱스 수, shard 수 |
| Nodes         | CPU, heap, disk, indexing/search rate, latency |
| Indices       | 인덱스별 색인·검색·storage 추이 |
| Logs          | (Filebeat 연동 시) 노드 로그 통합 검색 |

### 자가 모니터링 vs 별도 클러스터

운영 클러스터의 모니터링 데이터는 **별도의 monitoring 클러스터**로 보내는 것이 정석. 운영 클러스터가 죽으면 모니터링도 같이 죽기 때문이다 (Metricbeat 또는 Elastic Agent로 전송).

---

## 11.12 진단 쿼리 모음

```
# 1. 클러스터 전체 상태 한눈에
GET /_cluster/health?level=indices

# 2. 미할당 shard 원인
GET /_cluster/allocation/explain

# 3. 노드 전체 핵심 메트릭
GET /_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent

# 4. 색인별 크기·문서 수
GET /_cat/indices?v&s=store.size:desc

# 5. shard 분포·상태
GET /_cat/shards?v&s=state,index

# 6. thread pool 거부 추적
GET /_cat/thread_pool?v&h=node_name,name,active,queue,rejected

# 7. JVM heap·GC 상세
GET /_nodes/stats/jvm

# 8. 색인 압박 (8.x)
GET /_nodes/stats/indexing_pressure

# 9. Circuit breaker
GET /_nodes/stats/breaker

# 10. CPU 점유 thread stack
GET /_nodes/hot_threads?threads=10

# 11. Pending master tasks
GET /_cluster/pending_tasks

# 12. Recovery 상황
GET /_cat/recovery?v&active_only=true

# 13. Segments 수
GET /_cat/segments?v&s=index

# 14. Field cache 사용량
GET /_cat/fielddata?v

# 15. Disk usage 임박 여부
GET /_cat/allocation?v
```

---

## 11.13 Key Takeaways

| 영역 | 핵심 명령 | 보는 것 |
|------|-----------|---------|
| 전체 상태 | `_cluster/health` | green/yellow/red |
| 빠른 인스펙션 | `_cat/nodes`, `_cat/indices`, `_cat/shards` | 노드/인덱스/shard 상태 |
| 깊은 메트릭 | `_nodes/stats` | heap, GC, indexing pressure |
| CPU 진단 | `_nodes/hot_threads` | 무슨 thread가 바쁜가 |
| 미할당 진단 | `_cluster/allocation/explain` | 왜 할당 안 되는가 |
| 디스크 | watermark 85/90/95% | flood_stage = read-only |
| Heap | <75% 유지, GC time 추적 | 31GB 상한 |
| Slow log | 검색·색인 별도 임계 | query/fetch phase 구분 |

---

*다음 장: cheatsheets로 이동*
