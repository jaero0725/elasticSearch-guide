# 4장. Index, Shard, Replica

이 장은 Elasticsearch 운영의 가장 흔한 사고 원인 — **샤드 설계 실수** — 를 막기 위한 모든 개념을 정리한다. 라우팅이 어떻게 결정되는지, 샤드는 몇 개여야 하는지, 어떻게 노드 간 분산을 통제하는지, 그리고 일단 만들어진 인덱스를 사후에 reshape하는 도구들(shrink, split, clone, force merge)까지.

---

## 4.1 Index = 논리, Shard = 물리

이 구분이 모든 것의 출발이다.

- **Index**는 사용자가 보는 **논리 단위**다. `logs-2026.04.30`, `products`, `users` 같은 이름.
- **Shard**는 각 인덱스가 분할되어 디스크에 저장되는 **물리 단위**다. **각 샤드는 하나의 Lucene 인덱스**다 (3장 참고).

```
┌─────────────────────────────────────────────────────────┐
│  Index "logs-2026.04.30"                                 │
│  number_of_shards: 4                                     │
│  number_of_replicas: 1                                   │
│                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ Primary  │ │ Primary  │ │ Primary  │ │ Primary  │   │
│  │ Shard 0  │ │ Shard 1  │ │ Shard 2  │ │ Shard 3  │   │
│  │ (Lucene) │ │ (Lucene) │ │ (Lucene) │ │ (Lucene) │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
│                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ Replica  │ │ Replica  │ │ Replica  │ │ Replica  │   │
│  │ Shard 0  │ │ Shard 1  │ │ Shard 2  │ │ Shard 3  │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
└─────────────────────────────────────────────────────────┘
```

총 8개의 샤드(4 primary + 4 replica)가 클러스터의 노드들에 분산 배치된다.

### 인덱스 생성

```bash
PUT /logs-2026.04.30
{
  "settings": {
    "number_of_shards": 4,         # primary shard 수 — 생성 후 변경 어려움
    "number_of_replicas": 1        # replica 수 — 동적 변경 가능
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "message":    { "type": "text" },
      "level":      { "type": "keyword" }
    }
  }
}
```

**핵심**: `number_of_shards`는 생성 후 변경할 수 없다. 변경이 필요하면 reindex, split, shrink 같은 별도 작업이 필요하다(4.6 참고). `number_of_replicas`는 동적 변경 가능하다.

---

## 4.2 Primary vs Replica

### 역할 차이

| | Primary | Replica |
|--|---------|---------|
| **색인 처리** | 모든 색인은 primary 먼저 | primary로부터 동기 복제 받음 |
| **검색 처리** | 가능 | 가능 (부하 분산 대상) |
| **장애 대응** | 죽으면 replica가 promote | primary 장애 시 promote |
| **수 변경** | 생성 후 어려움 | 동적 변경 |

### Replica의 두 가지 역할

```
1. 고가용성 (HA):
   primary가 있는 노드가 죽으면
   → 마스터가 그 primary의 replica 중 하나를 새 primary로 promote
   → 클러스터 GREEN으로 복귀

2. 검색 처리량 (Throughput):
   검색 요청은 primary와 replica 어디든 갈 수 있음
   → replica가 늘면 읽기 QPS 선형 증가
   → 단, 색인 처리량은 떨어짐 (모든 replica로 복제해야 하므로)
```

### Trade-off

```
replicas=0:  색인 처리량 최대, 데이터 안전성 0 (노드 장애 = 데이터 유실)
replicas=1:  표준. 1대 노드 장애까지 견딤
replicas=2+: 더 강한 가용성, 더 많은 디스크, 색인 처리량 감소

검색 부하가 높고 노드가 충분하면 replicas 늘려서 분산.
색인 부하가 높고 가용성만 충족하면 replicas=1 유지.
```

### 동적 변경

```bash
# replica 수 변경 (즉시 반영, 새 replica 할당 시작)
PUT /logs-2026.04.30/_settings
{
  "index": { "number_of_replicas": 2 }
}

# 대량 색인 동안 replica 0으로 두고, 끝나면 복원하는 패턴
PUT /logs-init/_settings
{ "index": { "number_of_replicas": 0 } }
# ... bulk index ...
PUT /logs-init/_settings
{ "index": { "number_of_replicas": 1 } }
```

> **중요**: primary와 그 replica는 **반드시 서로 다른 노드**에 배치된다. 노드 장애 시 둘 다 잃지 않기 위해서. 노드 수가 부족해 이 제약을 만족할 수 없으면 일부 replica는 `unassigned` 상태로 남고 클러스터는 YELLOW가 된다.

---

## 4.3 샤드 라우팅

### 기본 공식

문서가 어떤 샤드로 갈지 결정하는 공식.

```
routing_factor = num_routing_shards / num_primary_shards
shard_num      = (hash(_routing) % num_routing_shards) / routing_factor
```

`_routing`의 기본값은 문서의 `_id`. 따라서:

```
shard_num = (hash(_id) % num_routing_shards) / routing_factor
```

이 공식은 [공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html)에 명시되어 있다.

### 단순한 경우 (num_routing_shards = num_primary_shards)

```
num_primary_shards = 4
num_routing_shards = 4
routing_factor     = 1

shard_num = hash(_id) % 4

   _id="abc123" → hash=0x7fa3... → 0x7fa3 % 4 = 3 → Shard 3
   _id="xyz789" → hash=0x12cd... → 0x12cd % 4 = 1 → Shard 1
```

### 왜 `num_routing_shards`가 분리되어 있는가

`num_routing_shards`(기본값은 num_primary_shards보다 큰 값으로 자동 설정)는 **나중에 인덱스를 split할 수 있게 하기 위한 여유분**이다.

예: `num_primary_shards=4`, `num_routing_shards=128` 으로 인덱스를 만들면:
- 4 → 8 → 16 → 32 → 64 → 128 까지 split 가능
- routing_factor만 조정하면 기존 문서의 hash 결과가 변하지 않음
- 즉, **resplit 시 데이터 재배치 없이 메타데이터만 갱신**

```
초기 (4 primary):
   shard 0 ← hash % 128 ∈ [0, 31]
   shard 1 ← hash % 128 ∈ [32, 63]
   shard 2 ← hash % 128 ∈ [64, 95]
   shard 3 ← hash % 128 ∈ [96, 127]

split → 8 primary:
   shard 0 ← [0, 15]    shard 1 ← [16, 31]    ← 기존 shard 0이 split됨
   shard 2 ← [32, 47]   shard 3 ← [48, 63]
   ...
```

### Custom Routing

같은 사용자의 모든 문서를 같은 샤드에 모으고 싶다면 `_routing`을 명시한다.

```bash
# 색인 시 routing 지정
POST /events/_doc?routing=user_42
{
  "user_id": "user_42",
  "event": "login"
}

# 검색 시에도 같은 routing 지정 → 한 샤드만 조회 (broadcast 회피)
GET /events/_search?routing=user_42
{
  "query": { "term": { "user_id": "user_42" } }
}
```

장점: 검색이 모든 샤드로 broadcast되지 않고 1개 샤드에서 처리. 처리량 급증.

함정: **샤드 간 데이터 쏠림(skew)** 위험. 특정 user의 데이터가 너무 많으면 그 샤드가 거대해진다. 또한 routing이 잘못되면 데이터가 검색되지 않는다.

---

## 4.4 샤드 사이징: 가장 흔한 사고의 원인

Elasticsearch 운영에서 가장 자주 잘못되는 결정이 **샤드 수**다. 너무 적으면 단일 샤드가 거대해져 검색이 느리고 recovery가 오래 걸리며, 너무 많으면 cluster state와 메타데이터 오버헤드가 클러스터를 죽인다.

### 공식 가이드라인

[공식 문서 "Size your shards"](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards)는 명확하다.

> "Aim for shard sizes between 10GB and 50GB"
> "Aim for 200 million documents or fewer per shard"

```
권장 범위:
  ┌──────────────────────────┐
  │ 10GB ──────────────── 50GB │  ← 샤드 크기 (저장 데이터)
  │ ~수백만 ─────── 200M docs  │  ← 샤드당 문서 수
  └──────────────────────────┘
```

이 범위 안에서 운영하라는 것이 디폴트 가이드. 검색 응답 시간이 ms 단위인 워크로드는 더 작게(~수 GB), 색인만 하고 검색은 가끔인 cold tier는 더 크게(50~150GB) 갈 수 있다.

### Heap당 샤드 수 한도

또 다른 중요한 룰:

> **JVM heap 1GB당 샤드 20개 이하**

heap=30GB 노드라면 600개 샤드까지가 한계. 이를 넘기면 metadata만으로 GC가 잦아지고 클러스터가 불안정해진다.

기본 cluster-level 한도는 노드당 1000 샤드 (frozen은 3000):

```bash
# 현재 한도 확인
GET _cluster/settings?include_defaults=true&filter_path=*.cluster.max_shards_per_node
```

### Oversharding의 증상

```
증상:
  - 매 요청 처리에 cluster state lookup 비용
  - master가 cluster state publish에 수 초~수십 초 소요
  - GC pause 증가, heap pressure
  - "yellow" 상태가 계속됨 (replica 할당 실패)
  - 디스크는 비어있는데도 새 인덱스 생성 거부

진단:
  GET _cat/shards?v=true | wc -l               # 총 샤드 수
  GET _cat/nodes?v=true&h=name,heap.percent,shards
  GET _cluster/stats?filter_path=indices.shards
```

### 적정 샤드 수 계산법

**시계열 데이터(로그)** 의 경우:

```
[샤드 수 산정 공식]

일일 색인 데이터 = D GB/day
보존 기간       = R days
샤드당 권장     = 30GB (10~50GB의 중간)
replica         = 1 (기본)

총 인덱스 수      = R                      (1일 1 인덱스 패턴)
인덱스당 샤드 수   = ceil(D / 30)
인덱스당 총 샤드  = 인덱스당 샤드 × (1 + replica)
클러스터 총 샤드  = 인덱스당 총 샤드 × R

예: D=200GB/day, R=30days
   인덱스당 샤드 = ceil(200/30) = 7
   인덱스당 총 샤드 = 7 × 2 = 14
   클러스터 총 샤드 = 14 × 30 = 420
```

**일반 인덱스 (검색용)** 의 경우 ILM 없이 단일 인덱스 운영:

```
예상 최종 데이터 크기 / 30GB → primary shard 수
50GB라면 → 2 shards
500GB라면 → 17 shards (반올림 16 또는 20)
```

### 일일 1 인덱스 vs Rollover

매일 새 인덱스를 만드는 패턴(`logs-2026.04.30`)은 단순하지만 **데이터량이 들쭉날쭉하면 샤드 크기가 불균일**해진다. 트래픽이 적은 날은 샤드가 1GB, 많은 날은 80GB가 될 수 있다.

대안: **Rollover**. 인덱스가 특정 크기/문서수에 도달하면 새 인덱스로 자동 전환.

```bash
# Index template + alias 설정
POST /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "lifecycle.name": "logs-policy",
      "lifecycle.rollover_alias": "logs"
    }
  }
}

# ILM 정책에서 rollover 조건 명시
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",   # ← 권장
            "max_age": "1d"
          }
        }
      }
    }
  }
}
```

`max_primary_shard_size: 50gb`로 두면 샤드가 항상 ~50GB에 가깝게 유지된다. 공식 문서가 추천하는 패턴.

---

## 4.5 Allocation, Awareness, Filtering

샤드를 **어느 노드에** 둘지 통제하는 메커니즘들.

### Cluster-level Allocation

기본적으로 Elasticsearch는 자동으로 샤드를 분산한다. 디스크 사용량 기반 watermark가 있다.

```yaml
cluster.routing.allocation.disk.watermark.low:   85%   # 새 샤드 할당 거부
cluster.routing.allocation.disk.watermark.high:  90%   # 기존 샤드 다른 노드로 이동
cluster.routing.allocation.disk.watermark.flood_stage: 95%  # 인덱스를 read-only로 전환
```

`flood_stage`(95%)에 도달하면 해당 노드의 모든 인덱스가 자동으로 `read-only-allow-delete`로 전환된다. **운영 사고의 단골 시나리오**다 — 디스크가 차서 색인이 멈춤. 디스크 확보 후 다음 명령으로 해제.

```bash
PUT /_all/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```

### Allocation Awareness (랙/AZ 분산)

복제본이 **같은 랙이나 같은 AZ에 몰리는 것을 방지**한다. 랙 전체가 죽어도 데이터를 잃지 않도록.

```yaml
# 각 노드의 yaml에 랙 정보 부여
node.attr.rack_id: rack_1     # 또는 rack_2, rack_3, ...

# 클러스터 설정
cluster.routing.allocation.awareness.attributes: rack_id
```

이렇게 하면 primary와 replica가 **다른 rack_id**의 노드에 배치된다. AWS면 `node.attr.zone: ap-northeast-2a` 같은 형식.

**Forced awareness**: 특정 zone이 통째로 죽었을 때 다른 zone에 모든 샤드를 몰아넣지 않도록 강제. 트래픽이 살아있는 zone에 폭증하지 않게 한다.

```yaml
cluster.routing.allocation.awareness.attributes: zone
cluster.routing.allocation.awareness.force.zone.values: zone_a, zone_b, zone_c
```

### Index-level Allocation Filtering (Hot/Warm/Cold)

특정 인덱스를 특정 노드 그룹에만 둘 수 있다. Hot/Warm/Cold 아키텍처의 기반.

```yaml
# 노드 yaml에 box_type 부여
node.attr.box_type: hot       # 또는 warm, cold
```

```bash
# 인덱스를 hot 노드에만 두기
PUT /logs-2026.04.30/_settings
{
  "index.routing.allocation.require.box_type": "hot"
}

# warm으로 이동
PUT /logs-2026.04.29/_settings
{
  "index.routing.allocation.require.box_type": "warm"
}
```

세 가지 키워드:
- `require`: 모든 값을 만족해야 함
- `include`: 하나라도 만족하면 OK
- `exclude`: 그 값을 가진 노드는 피함

ILM이 이 작업을 자동화한다.

> **9.x note**: `node.roles: [data_hot]` 같은 표준 data tier 역할을 쓰면 `box_type` 같은 custom attribute 없이도 자동 라우팅된다 (data tier shard filtering). 신규 클러스터는 이쪽이 표준.

---

## 4.6 Force Merge / Shrink / Split / Clone

생성된 인덱스를 사후에 reshape하는 도구들.

### Force Merge

3장에서 다뤘다. 색인이 끝난(write 없는) 인덱스의 segment를 줄여 검색 속도를 올린다.

```bash
POST /logs-2026.04.29/_forcemerge?max_num_segments=1
```

### Shrink: 샤드 수 줄이기

primary 샤드 수를 **줄이는** 작업. 예: 8 shards → 4 shards.

전제 조건:
1. 인덱스가 **read-only** 상태
2. 같은 노드에 모든 primary 샤드의 **사본**이 있어야 함 (allocation으로 한 노드에 모음)
3. 클러스터 health가 GREEN
4. **새 primary 수는 기존의 약수**여야 함 (8 → 4, 2, 1)

```bash
# 1. read-only로 전환 + 한 노드로 모음
PUT /my-index/_settings
{
  "settings": {
    "index.routing.allocation.require._name": "node-1",
    "index.blocks.write": true
  }
}

# 2. shrink
POST /my-index/_shrink/my-index-shrunk
{
  "settings": {
    "index.number_of_shards": 1,
    "index.number_of_replicas": 1,
    "index.routing.allocation.require._name": null,
    "index.blocks.write": null
  }
}
```

내부적으로는 **hard link**를 활용해 데이터 복사 없이 새 인덱스를 만든다. 매우 빠르다.

### Split: 샤드 수 늘리기

primary 샤드 수를 **늘리는** 작업. `num_routing_shards`가 충분히 커야 가능.

```bash
PUT /my-index/_settings
{
  "settings": { "index.blocks.write": true }
}

POST /my-index/_split/my-index-split
{
  "settings": {
    "index.number_of_shards": 8     # 기존의 배수만 가능 (1→2, 2→4, 4→8...)
  }
}
```

### Clone

같은 설정으로 인덱스 복제. 백업이나 실험용.

```bash
PUT /my-index/_settings
{ "settings": { "index.blocks.write": true } }

POST /my-index/_clone/my-index-clone
```

### 결정 트리

```
샤드 수 변경이 필요하다 →
   증가: Split (또는 Reindex)
   감소: Shrink (또는 Reindex)

원본 인덱스 설정/매핑을 변경하고 싶다 →
   Reindex (가장 자유롭지만 가장 느림 — 모든 데이터 복사)

색인이 끝났는데 segment가 많다 →
   Force merge to 1 segment

같은 인덱스의 사본이 필요하다 →
   Clone
```

---

## 4.7 운영 팁: Oversharding 진단과 대응

### 진단 체크리스트

```bash
# 1. 클러스터 총 샤드 수
GET _cluster/health?level=cluster
# → "active_shards", "active_primary_shards" 확인

# 2. 노드별 샤드 수와 heap
GET _cat/nodes?v=true&h=name,role,heap.percent,heap.max,shards

# 3. 인덱스별 샤드 크기 분포
GET _cat/shards?v=true&h=index,shard,prirep,docs,store&s=store:desc

# 4. cluster state 크기 (간접)
GET _cluster/state?filter_path=metadata.indices&human  # 응답 크기로 짐작
```

### 위험 신호

| 지표 | 위험 임계 |
|------|-----------|
| 노드당 샤드 수 | heap GB × 20 초과 |
| 클러스터 총 샤드 | 50,000 초과 |
| 평균 샤드 크기 | 1GB 미만 (oversharding) 또는 100GB 초과 (undersharding) |
| Cluster state publish 시간 | 수 초 이상 |
| Pending tasks | 지속적으로 0보다 큼 |

### 대응 전략

**Oversharding (샤드가 너무 많고 작음)**:
1. **Shrink**로 작은 인덱스들의 primary 수 감소
2. **Reindex**로 여러 작은 인덱스를 하나로 합침 (월간 인덱스 → 분기 인덱스)
3. ILM의 rollover 조건을 `max_primary_shard_size: 50gb`로 변경
4. 미래 인덱스부터는 더 적은 `number_of_shards`로 생성

**Undersharding (샤드가 너무 크고 적음)**:
1. **Split**으로 샤드 수 증가
2. 미래 인덱스는 더 많은 `number_of_shards`로 생성
3. 검색이 느리면 replica를 늘려 병렬 처리 활용

---

## 4.8 시각적 이해: 라우팅 흐름

```
Client: PUT /logs/_doc/abc123 { ... }
   │
   ▼
┌────────────────────────┐
│ Coordinating Node      │
│                        │
│  1. cluster state에서  │
│     "logs" 인덱스 메타  │
│     - num_primary=4   │
│     - routing_shards=128│
│                        │
│  2. shard 계산:         │
│     hash("abc123")     │
│     = 0x7fa3b2c1       │
│     0x7fa3b2c1 % 128   │
│     = 113              │
│     113 / (128/4)      │
│     = 113 / 32 = 3     │
│     → Shard 3          │
│                        │
│  3. Shard 3의 primary  │
│     위치 룩업: Node-C   │
└────────────┬───────────┘
             │
             ▼
   ┌──────────────────────┐
   │   Node-C (Shard 3)    │
   │   Primary             │
   │                       │
   │   1. Lucene 색인      │
   │   2. translog fsync   │
   │   3. replica로 복제   │
   └──────────┬───────────┘
              │
       ┌──────┴──────┐
       ▼             ▼
   ┌──────┐      ┌──────┐
   │Node-A│      │Node-D│
   │Replica│     │Replica│
   │of S3 │      │of S3 │
   └──────┘      └──────┘
       │             │
       └──────┬──────┘
              ▼
       모두 ack 완료
              │
              ▼
       Client에 응답
```

### Oversharding의 영향

```
정상 (적정 샤드):
  3 노드, 30GB heap each
  600 샤드 분산 → 노드당 200개
  cluster state: ~30MB
  publish 시간: <100ms

Oversharding:
  3 노드, 30GB heap each
  60,000 샤드 분산 → 노드당 20,000개  ← 한도 1000 초과!
  cluster state: ~3GB
  publish 시간: 30초+
  → 매 변경마다 30초 클러스터 일시정지
  → 새 노드 합류 시 cluster state 다운로드만 수 분
  → GC pause 빈발
  → 결국 클러스터 GREEN 회복 불가
```

---

## 4.9 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **Index** | 논리 단위. 사용자가 보는 이름. |
| **Shard** | 물리 단위 = 1 Lucene 인덱스. `number_of_shards`는 생성 후 변경 어려움 |
| **Primary vs Replica** | primary가 색인 우선 + 동기 복제. replica는 HA + 검색 분산 |
| **라우팅 공식** | `shard = (hash(_routing) % num_routing_shards) / routing_factor` |
| **Custom routing** | 같은 사용자 데이터를 한 샤드에 모음 → broadcast 회피, skew 위험 |
| **샤드 사이즈** | 10~50GB, < 200M docs/shard. heap 1GB당 샤드 20개 이하 |
| **Allocation awareness** | 랙/AZ 분산 강제 |
| **Filtering** | hot/warm/cold 노드 분리. ILM이 자동화 |
| **Reshape 도구** | Shrink(↓), Split(↑), Clone, Force merge |
| **Oversharding** | cluster state 비대 → publish 지연 → 클러스터 불안정 |

---

## 4.10 운영 시 주의점

- **인덱스 생성 전 `number_of_shards` 결정**. 기본값(1)은 작은 인덱스에는 OK지만, 대용량 인덱스에는 거의 항상 부적합.
- **ILM rollover에 `max_primary_shard_size: 50gb`** 사용. 시간 기반보다 안전.
- **replica를 동적으로 조정하라**. 대량 색인 중에는 0, 평소에는 1.
- **Disk watermark 모니터링**. 90% 넘으면 경보, 95% 넘으면 인덱스 read-only.
- **Force merge는 ILM의 warm phase에서만**. 운영 중인 hot 인덱스에 직접 호출 금지.
- **샤드 수 결정 시 `_id` 분포 확인**. 사용자 ID 기반 routing이면 skew 가능성 평가.
- **샤드 한도 cluster.max_shards_per_node 변경은 신중히**. 기본 1000은 안전망 — 늘리기 전에 oversharding 원인부터 해결.

```bash
# 자주 쓰는 운영 명령
GET _cluster/health
GET _cat/indices?v=true&s=store.size:desc
GET _cat/shards?v=true&s=store:desc | head -20
GET _cat/nodes?v=true&h=name,heap.percent,shards,disk.used_percent

# 특정 샤드의 할당 실패 이유
GET /_cluster/allocation/explain
{
  "index": "my-index",
  "shard": 0,
  "primary": false
}
```

---

*다음 장(이후 챕터): Mapping과 Analyzer, 검색 DSL과 Aggregation, ILM과 Snapshot, 그리고 운영 트러블슈팅.*
