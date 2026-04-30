# 2장. 클러스터 아키텍처

Elasticsearch는 단일 노드로도 동작하지만, 운영은 항상 다중 노드 클러스터다. 이 장은 클러스터를 구성하는 **노드 역할**, **클러스터 상태**, **마스터 선출**, **요청 라우팅**의 흐름을 다룬다. 이후 모든 트러블슈팅과 용량 산정은 이 장의 모델 위에서 이루어진다.

---

## 2.1 Node 역할 (Node Roles)

Elasticsearch 노드는 단순한 "프로세스"가 아니다. 각 노드는 `node.roles` 설정으로 결정되는 **하나 이상의 역할**을 갖는다. 8.x부터는 이 역할 분리가 매우 중요해졌다.

### 주요 역할 목록

| 역할 | 약어 | 설명 |
|------|------|------|
| `master` | `m` | 클러스터 상태 관리, 인덱스 생성/삭제, 샤드 할당 결정 |
| `voting_only` | `v` | 마스터 선출 투표만 참여, 마스터로 선출되지 않음 |
| `data` | `d` | 모든 데이터 티어 포함 (legacy) |
| `data_content` | `s` | 콘텐츠(검색 중심) 데이터 — 시계열이 아닌 일반 인덱스 |
| `data_hot` | `h` | 최근 시계열 데이터, 빈번한 색인/검색, SSD |
| `data_warm` | `w` | 색인 종료된 데이터, 가끔 검색, 보통 HDD |
| `data_cold` | `c` | 거의 검색되지 않는 오래된 데이터, searchable snapshot |
| `data_frozen` | `f` | 스냅샷 저장소에 부분 마운트, 매우 저렴, 매우 느림 |
| `ingest` | `i` | ingest pipeline 실행 (문서 전처리) |
| `ml` | `l` | Machine Learning 작업 |
| `transform` | `t` | Transform 작업 (인덱스 → pivoted/latest 인덱스) |
| `remote_cluster_client` | `r` | CCR/CCS용 원격 클러스터 연결 |
| (no roles) | `-` | **Coordinating-only 노드** — 요청 라우팅만 담당 |

설정 예:

```yaml
# elasticsearch.yml — dedicated master 노드
node.roles: [ master ]

# dedicated hot data 노드
node.roles: [ data_hot, data_content, ingest ]

# coordinating-only 노드 (모든 역할 제거)
node.roles: [ ]
```

### Voting-Only 노드의 용도

`voting_only`는 **마스터 후보지만 마스터가 될 수는 없는 노드**다. 왜 이런 게 필요한가?

쿼럼은 홀수가 권장된다(분할 뇌(split-brain) 방지). 하지만 dedicated master 3대는 비용이 높다. 2개의 dedicated master + 1개의 voting_only(보통 데이터 노드 중 하나)로 비용을 줄이면서 쿼럼은 3을 유지할 수 있다. 단, 공식 문서는 **운영 환경에서는 진짜 dedicated master 3대 권장**이라고 명시한다. ([Voting configurations](https://www.elastic.co/docs/deploy-manage/distributed-architecture/discovery-cluster-formation/modules-discovery-voting))

### Data Tier (Hot/Warm/Cold/Frozen)

시계열 데이터(로그, 메트릭) 운영의 표준 패턴. 비용/성능 곡선을 따라 데이터를 단계적으로 이전한다.

```
   비용/성능
      ▲
      │
 Hot  │  ████████  SSD, 빈번한 색인+검색, primary+replica 풀 보존
      │
 Warm │  ██████    SSD/HDD, 색인 종료, 검색은 가끔, replica 축소 가능
      │
 Cold │  ████      HDD, searchable snapshot, primary만
      │
Frozen│  ██        Object storage 부분 마운트, 매우 느린 검색
      │
      └────────────────────────────────────────────► 데이터 나이
```

ILM(Index Lifecycle Management)이 이 이전을 자동화한다 (별도 챕터 가능 영역). frozen tier는 데이터를 로컬에 두지 않고 S3 등 스냅샷 저장소에서 직접 부분 마운트하므로, **node당 frozen 샤드 한도는 3000**(non-frozen은 1000)으로 다르게 설정되어 있다.

> **9.x 변경점**: 9.0부터 frozen 인덱스의 일부 read 동작이 변경되어, 8.x에서 frozen 처리된 인덱스는 9.x 업그레이드 전 unfreeze 필요. ([Breaking changes](https://www.elastic.co/docs/release-notes/elasticsearch/breaking-changes))

---

## 2.2 Cluster State와 Master Node

### Cluster State란?

Cluster state는 클러스터의 **모든 메타데이터**를 담은 단일 자료구조다. 다음을 포함한다.

- 인덱스 메타데이터 (settings, mappings, aliases)
- 샤드 위치 (어떤 샤드가 어느 노드에 있는지)
- 노드 목록과 역할
- 라우팅 테이블
- 보안 정보, ILM 정책, ingest pipeline 등

이 cluster state는 **마스터 노드가 단일 진실 공급자(single source of truth)**다. 변경(인덱스 생성, 샤드 이동 등)은 마스터에서 일어나고, 마스터가 모든 노드에 새 버전을 발행(publish)한다.

```
          [master]
              │ ① 변경 결정
              │ ② 새 cluster state version 생성
              │ ③ 모든 노드에 publish
              │
        ┌─────┼─────┬─────┐
        ▼     ▼     ▼     ▼
      data1 data2 data3  ...   ← 각 노드는 자기 부분에 적용
                                  (예: 자기에게 할당된 샤드를 시작)
```

### Cluster State 크기가 중요한 이유

cluster state는 **모든 노드가 메모리에 통째로** 들고 있어야 한다. 인덱스/샤드가 많아질수록 state 크기가 선형 증가한다.

- 인덱스 1개당 state 메타데이터: mapping 크기에 따라 KB~수 MB
- 수천 개 인덱스 + 수만 개 샤드 → state가 수백 MB
- 변경 발생 시 매번 모든 노드로 publish → 네트워크 부하
- 마스터가 publish ack 기다리며 블록 → 클러스터 전체 멈춤

이것이 **oversharding이 위험한 가장 큰 이유** 중 하나다 (4장 상세).

---

## 2.3 Discovery: 노드들이 서로를 찾는 법

Elasticsearch 클러스터는 자동 클러스터링이 아니다. 각 노드는 **다른 마스터 후보 노드의 주소**를 알아야 시작할 수 있다.

### 7.x 이후 변경: Zen → 새 Discovery

Elasticsearch 7.0에서 기존 Zen Discovery가 폐기되고 **새 Discovery + Cluster Coordination**이 도입되었다. 가장 큰 변화는 **`minimum_master_nodes` 설정 제거**와 **`cluster.initial_master_nodes` 도입**이다.

### 핵심 설정

```yaml
# elasticsearch.yml

# (1) 클러스터 식별자 — 같은 cluster.name을 가진 노드만 합류
cluster.name: prod-logs

# (2) seed_hosts — 처음 만날 마스터 후보 노드들의 주소
# 모든 마스터 후보의 주소를 나열 (DNS 가능, 동적 추가 OK)
discovery.seed_hosts:
  - master-1.es.internal
  - master-2.es.internal
  - master-3.es.internal

# (3) cluster.initial_master_nodes — "최초 부트스트랩" 시에만 사용
# 빈 클러스터를 처음 띄울 때, 어느 노드들이 첫 마스터 선출에 투표할지
# *** 클러스터가 한 번 형성된 후에는 절대 변경/추가하지 말 것 ***
cluster.initial_master_nodes:
  - master-1
  - master-2
  - master-3
```

### `cluster.initial_master_nodes`의 함정

이것은 **최초 부트스트랩 한 번만** 의미가 있다. 하지만 흔한 운영 사고로:

- 새 노드를 추가하면서 기존 yaml의 `initial_master_nodes`를 그대로 복사 → 새 노드가 이미 형성된 클러스터를 무시하고 별도 클러스터로 부트스트랩
- 두 개의 분리된 클러스터가 동시에 마스터를 선출 → split-brain

해결책:

> **클러스터가 한 번 형성된 직후 모든 노드의 yaml에서 `cluster.initial_master_nodes` 제거**.

`seed_hosts`만 두고, 그 외 노드 추가는 동적으로 처리한다.

---

## 2.4 Master Election: Raft-like Quorum

Elasticsearch의 마스터 선출 알고리즘은 Raft에서 영감을 받은 자체 합의 프로토콜이다. ([공식 문서](https://www.elastic.co/docs/deploy-manage/distributed-architecture/discovery-cluster-formation/modules-discovery-quorums))

### Voting Configuration

선출에 참여하는 마스터 후보 노드의 부분집합을 **voting configuration**이라 한다. 이 집합의 **과반수(strict majority, > 50%)**가 응답해야 결정이 성립한다.

```
voting configuration: { master-1, master-2, master-3 }
quorum = floor(N/2) + 1
       = floor(3/2) + 1 = 2   → 2표면 선출 성립
        ┌─────────┐
        │ master-1│ ✓ vote
        │ master-2│ ✓ vote   → 2/3 = 66% > 50% → 선출 성공
        │ master-3│ ✗ down
        └─────────┘

N=3 → quorum 2 (1대 장애 허용)
N=5 → quorum 3 (2대 장애 허용)
N=2는 안티패턴 (quorum 2 → 한 대 장애도 못 견딤)
```

### 왜 홀수인가

| 구성 | 노드 수 | quorum | 견딜 수 있는 장애 |
|------|---------|--------|-------------------|
| 1 master | 1 | 1 | 0 (SPOF) |
| 2 masters | 2 | 2 | 0 (1대만 죽어도 쿼럼 미달) |
| **3 masters** | **3** | **2** | **1** ← 권장 |
| 4 masters | 4 | 3 | 1 (3보다 나을 게 없는데 비용↑) |
| 5 masters | 5 | 3 | 2 (대규모 클러스터) |

**결론: dedicated master는 3대(또는 5대). 2대나 4대는 의미 없다.**

### Auto-reconfiguration

7.x 이후 voting configuration은 **자동 조정**된다. 마스터 후보 노드가 추가/제거되면 클러스터가 알아서 voting config를 업데이트한다. 단, 이 자동화는 클러스터가 정상 상태일 때만 동작하므로, 장애 상황에서 voting config가 stale한 상태로 남으면 수동 개입이 필요하다 (`POST _cluster/voting_config_exclusions`).

---

## 2.5 요청 라우팅: Coordinating Node의 역할

클라이언트 요청은 **어느 노드에 보내도 된다**. 받은 노드가 자동으로 **coordinating node** 역할을 수행한다.

### Search 요청 흐름

```
   Client
     │ ① GET /logs-*/_search { query: ... }
     ▼
┌─────────────┐
│ Node-X      │  ← coordinating 역할
│ (어떤 노드든)│
└──────┬──────┘
       │
       │ ② 인덱스 → 샤드 매핑 룩업 (cluster state)
       │    "logs-2026.04.30"의 4개 primary 중 어디에서 검색할지
       │    (primary 또는 replica 중 부하 분산해서 선택)
       │
       ├────────────┬────────────┬────────────┐
       ▼            ▼            ▼            ▼
   Shard 0      Shard 1      Shard 2      Shard 3
   (Node-A)     (Node-B)     (Node-C,R)   (Node-A)
       │            │            │            │
       │ Query Phase: 각 샤드에서 top-K 문서 ID + 점수 반환
       │            │            │            │
       └────────────┴──────┬─────┴────────────┘
                           ▼
                    Node-X (coordinating)
                    ③ 결과 머지 → 글로벌 top-K 결정
                           │
                           │ Fetch Phase: 선택된 문서의 _source를 해당 샤드에서 수집
                           │
                           ▼
                       Client에 응답
```

검색은 **Query Then Fetch**의 2-phase 모델로 동작한다.

1. **Query Phase**: 각 샤드에서 (size + from)개의 문서 ID/점수만 반환
2. **Fetch Phase**: coordinating이 글로벌 top-K를 정한 뒤, 해당 문서의 실제 내용만 가져옴

이 구조 때문에 **deep pagination(`from=10000`)**이 매우 비싸진다. 모든 샤드가 from+size만큼을 반환해야 하기 때문이다. → `search_after` 또는 `point in time` 사용 권장.

### Index 요청 흐름

```
   Client
     │ ① POST /logs/_doc { ... }
     ▼
┌─────────────┐
│ Node-X      │  coordinating 역할
└──────┬──────┘
       │ ② shard_num = hash(_id) % num_primary_shards
       │    예: shard_num = 2
       │
       ▼
   Shard 2 primary (예: Node-C)
       │ ③ 색인 (in-memory buffer + translog)
       │
       ├──────────► Replica 1 (Node-D)  ← 동기 복제
       └──────────► Replica 2 (Node-E)  ← 동기 복제
       │
       │ ④ primary가 모든 in-sync replica의 ack 수신 후
       │
       ▼
   coordinating으로 응답 → Client
```

색인은 **primary 우선 + 모든 in-sync replica 동기 복제**다. 따라서 색인 응답 시점에 데이터는 모든 활성 사본에 도달한 상태다(Lucene flush와는 다른 차원의 보장 — 3장 참고).

---

## 2.6 운영 팁

### 1. Dedicated Master 3대 권장

**규모와 무관하게**, 운영 환경에서는 dedicated master 3대를 분리하는 것이 표준이다.

```yaml
# master 노드 — 데이터/검색 부하 격리
node.roles: [ master ]

# 메모리는 작아도 됨 (4~8GB heap), CPU와 디스크 I/O는 크게 필요 없음
# 단, 네트워크는 모든 데이터 노드와 통신해야 하므로 안정성 필요
```

dedicated master를 두지 않으면, 데이터 노드의 GC pause나 디스크 I/O 폭증이 마스터 election timeout을 유발하여 **클러스터 전체가 잠시 멈출 수 있다**.

### 2. Coordinating-only 노드는 언제 필요한가

대부분의 환경에서는 **불필요**하다. 데이터 노드가 coordinating 역할을 겸할 수 있기 때문이다. 다음 경우에만 분리를 검토하라.

- **거대한 집계 쿼리**가 잦아, 머지 단계의 메모리/CPU가 데이터 노드의 색인을 방해
- **클라이언트 연결 수가 매우 많음** (수천 개 keep-alive)
- **검색 트래픽과 색인 트래픽 격리** 필요

```yaml
# coordinating-only
node.roles: [ ]
```

이 노드는 데이터를 보유하지 않고 cluster state도 메모리에 가지지만, 샤드 할당은 받지 않는다. heap 32GB 미만, CPU 코어 많음(병렬 머지)이 일반적이다.

### 3. 클러스터 구성 예시

```
                ┌─────────────────────────────────────┐
                │   Load Balancer (HTTP 9200)         │
                └──────────────┬──────────────────────┘
                               │
        ┌──────────────────────┴────────────────────────┐
        │                                                │
   ┌────▼────┐  ┌────────┐  ┌────────┐                  │
   │coord-1  │  │coord-2 │  │coord-3 │   (선택사항)       │
   │ (no role)│  │        │  │        │                  │
   └─────────┘  └────────┘  └────────┘                  │
                                                          │
   ┌─────────┐  ┌─────────┐  ┌─────────┐                 │
   │master-1 │  │master-2 │  │master-3 │                 │
   │ [m]     │  │ [m]     │  │ [m]     │   ← 쿼럼 3       │
   └─────────┘  └─────────┘  └─────────┘                 │
                                                          │
   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
   │ data-h-1 │ │ data-h-2 │ │ data-w-1 │ │ data-w-2 │  │
   │ [data_   │ │ [data_   │ │ [data_   │ │ [data_   │  │
   │  hot,    │ │  hot,    │ │  warm]   │ │  warm]   │  │
   │  content]│ │  content]│ │          │ │          │  │
   └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
                                                          │
                          ... (필요한 만큼 확장) ...
```

---

## 2.7 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **Node roles** | `node.roles`로 명시. master/data_hot/warm/cold/frozen/ingest/ml 분리 |
| **Cluster state** | 마스터가 단일 진실 공급. 모든 노드가 메모리에 보유 |
| **Discovery** | `seed_hosts`로 노드 발견, `initial_master_nodes`는 최초 부트스트랩만 |
| **Master election** | Raft-like, voting config 과반수(>50%). dedicated master 3대 표준 |
| **Coordinating** | 어느 노드든 받을 수 있음. Query→Fetch 2-phase 검색 |
| **Hot/Warm/Cold/Frozen** | 비용/성능 단계 — ILM으로 자동 이전 |
| **Voting-only** | 마스터 후보지만 선출 안됨. 비용 절감용 옵션 |

---

## 2.8 운영 시 주의점

- **`cluster.initial_master_nodes`는 부트스트랩 한 번만**. 이후 즉시 yaml에서 제거.
- **dedicated master 3대 + 데이터 노드 N대**가 운영의 출발점. 2대나 4대는 의미 없음.
- **마스터 노드 디스크 모니터링**: master는 cluster state 디스크 persist를 위해 IO를 사용. 디스크 풀 시 마스터 election 실패.
- **Cluster state 크기 모니터링**: `GET _cluster/state?filter_path=metadata` 응답 크기를 추적. 100MB 넘어가면 oversharding 의심.
- **`GET _cat/nodes?v=true&h=name,node.role,heap.percent,master`** 로 클러스터 토폴로지 상시 확인.

```
# 클러스터 헬스 확인
GET /_cluster/health

# 노드별 역할/리소스
GET /_cat/nodes?v=true&h=name,node.role,heap.percent,disk.used_percent,master

# 누가 마스터인가
GET /_cat/master?v=true
```

---

*다음 장: 3장. Lucene 내부 구조 — 역인덱스, doc values, segment 라이프사이클의 동작 메커니즘*
