# C2. Split Brain / Master 부재 문제

## 증상

- 클러스터가 두 개로 갈라져 각각 자기가 active master라고 주장 (split brain)
- 한쪽 ES 클러스터가 `master_not_discovered_exception` 반복:

```
{
  "type": "master_not_discovered_exception",
  "reason": "have discovered ...; discovery will continue using ..."
}
```

- 인덱싱 멈춤, 응답이 503
- 노드들이 마스터 선출에 실패하며 로그에 다음:

```
[WARN][o.e.c.c.ClusterFormationFailureHelper] master not discovered yet,
this node has not previously joined a bootstrapped cluster, and
[discovery.seed_hosts] is empty
```

또는

```
master not discovered or elected yet, an election requires
at least 2 nodes with ids from [...]
```

---

## 원인

### 7.x 이후의 마스터 선출

7.0부터 ES는 자동으로 quorum을 계산한다 (`discovery.zen.minimum_master_nodes` 폐지). 그러나 다음 실수가 여전히 흔하다:

#### 실수 1: 마스터 후보 노드 수가 짝수 또는 단일

```
2개 노드 모두 master eligible:
  네트워크 분리 시 두 노드가 각자 master 시도 → split brain
  또는 quorum 못 채워서 양쪽 다 멈춤

1개 master eligible 노드:
  그 노드가 죽으면 클러스터 즉시 멈춤 (단일 장애점)
```

#### 실수 2: dedicated master 부재

```
모든 노드가 data + master 역할 겸용:
  → 데이터 부하가 큰 노드의 GC 폭주 시 마스터 응답 실패
  → 마스터 변경 → cluster state 재배포 → 더 큰 부하 → 폭주 사이클
```

#### 실수 3: cluster bootstrap 잘못

```yaml
# elasticsearch.yml — 처음 클러스터 만들 때
cluster.initial_master_nodes: ["node-1"]
```

`cluster.initial_master_nodes`는 **클러스터 최초 생성 시 한 번만** 의미가 있다. 운영 중인 클러스터에 이 설정이 남아 있으면 새 노드가 합류할 때 혼동이 생긴다.

#### 실수 4: 네트워크 양쪽으로 갈라진 상태에서 자체 부트스트랩

```
3개 노드 클러스터 (n1, n2, n3):
  네트워크 파티션 → n1만 격리
  n1이 자체 부트스트랩으로 새 클러스터 시작 → split brain
```

8.x에서는 자동 부트스트랩이 차단되어 이 시나리오가 거의 사라졌지만, 잘못된 설정/수동 강제 시 가능.

---

## 진단

### 1) 현재 마스터 확인

```bash
GET /_cat/master?v

# 응답 예 (정상):
# id                     host       ip           node
# x9sQ3mE8Q5mZ...        es-mst-1   10.0.0.10    es-mst-1

# 비정상:
# (응답 없음 또는 5xx)  → 마스터 없음
# 두 곳에서 다른 master id 나옴 → split brain
```

### 2) 클러스터 멤버

```bash
GET /_cluster/health

# 응답:
# "number_of_nodes": 5
# "number_of_data_nodes": 3
# 정상이면 모든 노드 보임. 일부만 보이면 분리됨
```

### 3) 노드 역할

```bash
GET /_cat/nodes?v&h=name,node.role,master,heap.percent

# 응답 예:
# name        node.role  master  heap.percent
# es-mst-1    m          *        45              ← 마스터
# es-mst-2    m          -        40
# es-mst-3    m          -        42
# es-data-1   d          -        72
# es-data-2   d          -        70
# es-data-3   d          -        68
# 'm' = master eligible, 'd' = data
```

### 4) 마스터 선출 로그

```bash
# 모든 마스터 후보 노드의 로그에서:
grep "elected-as-master\|master not discovered" /var/log/elasticsearch/*.log

# 정상: 한 노드에서만 elected-as-master
# 비정상: 여러 노드에서 elected 또는 모든 노드에서 not discovered
```

### 5) Cluster bootstrap 설정 확인

```bash
# 모든 노드의 elasticsearch.yml
cluster.initial_master_nodes: [...]
# 운영 중인 클러스터에 이 설정이 있으면 안 됨
```

### 6) 네트워크/방화벽

```bash
# 노드 간 9300 포트 (transport) 통신
# 한쪽 노드에서:
nc -zv es-mst-2 9300
nc -zv es-mst-3 9300

# 양방향 모두 가능해야 함
```

---

## 해결

### 단계 1: split brain 발생 — 한쪽 정지

split brain이 실제로 발생했으면 **한쪽을 강제 정지**해야 한다. 두 클러스터를 합칠 수 있는 자동 절차는 없다.

```bash
# 데이터가 더 많거나 더 신뢰할 수 있는 쪽 선택
# 다른 쪽 노드들 모두 정지
sudo systemctl stop elasticsearch

# 정지된 쪽의 데이터 디렉터리 백업 (필요 시 수동 머지)
cp -r /var/lib/elasticsearch /backup/split-brain-side/
```

이후 그쪽 노드들을 빈 상태로 시작 또는 정상 클러스터에 합류.

### 단계 2: 마스터 후보 수를 3개 이상 홀수로

```yaml
# elasticsearch.yml — dedicated master 노드 3개
node.roles: [ master ]
node.name: es-mst-1
discovery.seed_hosts: [ "es-mst-1", "es-mst-2", "es-mst-3" ]
```

마스터 후보가 3개면:

```
정상:    3개 다 살아 있음 → quorum=2
1개 장애: 2개 살아 있음 → quorum=2 충족 → 계속 동작
2개 장애: 1개 살아 있음 → quorum 못 채움 → 안전하게 멈춤 (split brain 방지)
```

### 단계 3: dedicated master vs 겸용 노드 분리

소형 클러스터(노드 수 < 6) 추천 구성:

```yaml
# Master-eligible (3대, 작은 사양)
node.roles: [ master ]

# Data + ingest (필요한 만큼)
node.roles: [ data, data_content, ingest ]

# (대형) coordinating only
node.roles: [ ]   # 빈 배열 = coordinating
```

```
이점:
  - 데이터 노드 GC 폭주가 마스터에 영향 없음
  - 마스터 노드는 작은 사양으로 충분 (4 vCPU, 8GB heap)
  - cluster state 처리 안정
```

### 단계 4: 클러스터 부트스트랩 정리

```yaml
# elasticsearch.yml — 운영 중 클러스터에 신규 노드 추가 시
# cluster.initial_master_nodes 는 절대 넣지 말 것
# 대신 discovery.seed_hosts 로 마스터 후보 알려주기
discovery.seed_hosts: [ "es-mst-1:9300", "es-mst-2:9300", "es-mst-3:9300" ]
```

```
cluster.initial_master_nodes:
  - 클러스터 최초 부트스트랩 시 한 번만 사용
  - 부트스트랩 후 모든 노드 yml에서 제거
  - 살아있는 클러스터에 신규 노드 추가 시 절대 사용 X
```

### 단계 5: voting-only 노드 (선택)

마스터 5개 노드를 두면 quorum=3이라 안정성↑ 하지만 cluster state publish 비용↑.
voting-only 노드는 선출에는 참여하지만 마스터가 되지는 않음:

```yaml
# 5개 중 2개를 voting-only로
node.roles: [ master, voting_only ]
```

### 단계 6: 응급 — 마스터 강제 부트스트랩 (위험)

남은 마스터 후보가 quorum 미만일 때 (예: 3개 중 2개 유실, 1개만 남음):

```bash
# 모든 데이터를 잃을 위험. 백업 후 진행
elasticsearch-node remove-customs '*'
elasticsearch-node remove-settings 'cluster.initial_master_nodes'
elasticsearch-node unsafe-bootstrap
```

> **주의**: 데이터 손실 가능. Elastic 지원과 상의 후 진행 권장.

---

## 예방

### 1) 마스터 후보 수 = 3 (또는 5)

```
권장 구성:
  소형 클러스터: master 3 + data N + (coord 0~2)
  대형 클러스터: master 5 (그중 2개 voting-only) + data N + coord M
```

### 2) 마스터와 데이터 분리

```
이유:
  - 데이터 노드 GC pause가 cluster state heartbeat에 영향 없음
  - 마스터 노드는 작은 인스턴스로 충분
  - 비용 대비 안정성 큼
```

### 3) 모니터링 알람

```bash
# 정기 체크
GET /_cluster/health
# status: red, yellow → 알람
# number_of_nodes 가 예상치보다 적으면 알람

GET /_cat/master?v
# 응답 실패 = 마스터 없음 (즉시 알람)
```

### 4) 네트워크 안정성

```
- 노드 간 ping/지터 모니터링
- 9300 (transport) 방화벽 양방향 OK 확인
- 클라우드: 같은 가용 영역 내 또는 저지연 네트워크 보장
- discovery.seed_hosts 에 모든 마스터 후보 명시
```

### 5) 안티패턴

- 마스터 후보 1개 또는 2개
- 모든 노드를 data + master 겸용
- `cluster.initial_master_nodes`를 운영 중에 그대로 두기
- 마스터 노드를 GC 부담 큰 워크로드 (kibana, monitoring) 와 같은 머신에 배치
- 짝수 마스터 후보 (split brain risk 자체는 없지만 quorum 비효율)

**핵심: master eligible 3개를 dedicated로. 그게 99%의 split brain/master 문제를 막는다.**
