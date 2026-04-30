# C1. 작은 샤드가 수만 개 — Over-sharding

## 증상

- 클러스터 상태(`cluster state`) 갱신이 느림 (수 초~수 분)
- 마스터 노드의 heap 사용률이 80% 이상 만성적
- 새 노드 합류/재시작이 매우 오래 걸림 (샤드 할당)
- `_cat/shards` 응답이 거대 (수만 라인)
- `_cat/health` 에서 `unassigned_shards` 가 끊임없이 변동
- 다음과 같은 경고:

```
{ "type": "validation_exception",
  "reason": "Validation Failed: 1: this action would add [...] total shards,
             but this cluster currently has [1500]/[1000] maximum shards open;" }
```

또는

```
[shard_limit] cluster.max_shards_per_node will be exceeded
```

---

## 원인

### 샤드는 무료가 아니다

각 샤드는:

```
- Lucene 인덱스 1개 (segment metadata, term dictionary, ...)
- 노드 heap 메모리 (수십 MB ~ 수백 MB 고정)
- File handle (디스크 파일 수십~수백 개)
- Cluster state 항목 (마스터에 보관)
- 검색 시 N개 샤드 = N번의 컨텍스트 스위치
```

샤드 1만 개 클러스터는:

```
heap 점유:        샤드당 평균 50MB × 10000 = 500GB heap (전체 노드 합)
cluster state:    수십 MB → 마스터 노드 매번 publish
샤드 통계:        모니터링 호출 응답 시간 증가
검색 fan-out:     인덱스에 샤드 50개 = 50번의 작은 요청
```

### 흔한 잘못된 설계

```
잘못된 예 1: 일별 인덱스 + 큰 샤드 수
  logs-2025.04.30 (5 shards)
  logs-2025.04.29 (5 shards)
  logs-2025.04.28 (5 shards)
  ...
  1년 = 365 × 5 = 1825 샤드 (작은 인덱스에)

잘못된 예 2: replicas 과다
  primary 5 + replica 2 = 15 샤드/인덱스
  → 인덱스 100개면 1500 샤드

잘못된 예 3: 작은 인덱스 수만 개
  서비스별 × 환경별 × 일별 = 곱셈으로 폭증
```

### 권장 샤드 크기

```
권장:
  단일 샤드 크기:  10GB ~ 50GB (검색·로그)
                 50GB 이상 가능 (메트릭/시계열, 8.x)
  노드당 샤드 수:  heap GB × 20 이하 (예: 32GB heap → 640 샤드 이하)
  클러스터 총 샤드: 수만 개 이상이면 위험
```

---

## 진단

### 1) 전체 샤드 통계

```bash
# 클러스터 전체 샤드 수
GET /_cluster/health?level=cluster
# active_shards, unassigned_shards 확인

# 노드별 샤드 수
GET /_cat/allocation?v
# shards, disk.indices, disk.used 컬럼

# 인덱스별 + 샤드 크기
GET /_cat/shards?v&s=store:desc
# 작은 샤드(100MB 이하)가 많으면 over-sharding
```

### 2) 샤드 크기 분포

```bash
GET /_cat/indices?v&h=index,docs.count,store.size,pri,rep&s=store.size:desc

# 응답 예 (문제 있는 클러스터):
# index            docs.count  store.size  pri  rep
# logs-2025.04.30      1.2m       450mb     5    1   ← 5개 샤드 × 90mb 너무 작음
# logs-2025.04.29      1.1m       420mb     5    1
# ...
# 모든 인덱스가 sub-1GB primary
```

### 3) 마스터 노드 부하

```bash
GET /_nodes/stats/jvm,os?human

# 마스터에서 heap_used_percent 계속 70%+
# OS load average 가 노드 수와 무관하게 높음
```

### 4) cluster state 크기

```bash
GET /_cluster/state?human&filter_path=cluster_uuid

# 응답 시간이 1초 이상이면 비대 (수십 MB+)
GET /_cluster/state/metadata?filter_path=metadata.indices  # 가장 큰 부분
```

### 5) 한도 확인

```bash
GET /_cluster/settings?include_defaults=true&filter_path=*.cluster.max_shards_per_node,*.defaults.cluster.max_shards_per_node
# 기본값: 1000 (8.x)
```

---

## 해결

### 단계 1: 즉각 — 비활성 인덱스 close 또는 삭제

```bash
# 오래된 인덱스 close (메모리 절감)
POST /logs-2024.01.*/_close

# 또는 삭제
DELETE /logs-2024.01.*

# searchable snapshot로 cold tier 이동 (라이선스 필요)
POST /logs-2024.01.01/_searchable_snapshot
```

### 단계 2: 인덱스 합치기 — Shrink + ILM rollover 재설계

#### 2.1 ILM rollover로 일별 → 시간/크기 기반

```json
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",                          // 7일 단위
            "max_primary_shard_size": "50gb"          // 50GB 도달 시
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },        // 5 → 1 샤드
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}
```

#### 2.2 Shrink API로 샤드 수 축소

```bash
# 1) 단일 노드로 샤드 모음 + read-only
PUT /logs-2025.03.01/_settings
{
  "index.routing.allocation.require._name": "es-data-1",
  "index.blocks.write": true
}

# 2) shrink (5 shards → 1 shard)
POST /logs-2025.03.01/_shrink/logs-2025.03.01-shrunk
{
  "settings": {
    "index.number_of_shards": 1,
    "index.number_of_replicas": 1,
    "index.routing.allocation.require._name": null
  }
}

# 3) 별칭 전환
POST /_aliases
{
  "actions": [
    { "remove_index": { "index": "logs-2025.03.01" } },
    { "add": { "index": "logs-2025.03.01-shrunk", "alias": "logs-2025.03.01" } }
  ]
}
```

> shrink는 number_of_shards가 원본의 약수여야 한다 (5 → 5,1 가능, 5 → 2 불가).

### 단계 3: 한도 임시 상향

```bash
PUT /_cluster/settings
{
  "persistent": {
    "cluster.max_shards_per_node": 2000
  }
}
```

> 응급용. 결국 샤드 수를 줄여야 함.

### 단계 4: 신규 인덱스의 샤드 수 줄이기

```bash
# 인덱스 템플릿
PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index.number_of_shards":   1,    // 일반적으로 1개로 시작
      "index.number_of_replicas": 1
    }
  }
}
```

### 단계 5: Reindex로 여러 작은 인덱스를 합치기

```bash
POST /_reindex
{
  "source": {
    "index": "logs-2024.01.*"      // 1월 31개 인덱스
  },
  "dest": {
    "index": "logs-2024.01"        // 1개 인덱스로
  }
}
```

---

## 예방

### 1) 샤드 크기 가이드라인

```
검색·로그 워크로드:
  primary shard 크기 = 10GB ~ 50GB 목표
  너무 작으면 shrink, 너무 크면 split (또는 rollover로 새 인덱스)

메트릭/시계열:
  primary shard 크기 = 30GB ~ 80GB 가능
  TSDS는 더 큰 샤드도 잘 처리

인덱스당 샤드 수 결정:
  예상 일일 데이터 크기 / 권장 샤드 크기
  100GB/일 → primary 2개 (50GB 도달 시 rollover)
```

### 2) ILM 강제 사용

신규 데이터는 모두 ILM rollover로:

```
원본:    `logs-myapp` (data stream alias)
실제:    logs-myapp-2025.04.30-000001
         logs-myapp-2025.05.07-000002
         ...

크기 기반 rollover 가 시간 기반보다 우선 → 트래픽이 적은 날 인덱스가 작아도 OK
```

### 3) 노드별 샤드 한도

```bash
# 처음부터 작게 잡고 운영 (예방적)
PUT /_cluster/settings
{
  "persistent": {
    "cluster.max_shards_per_node": 800
  }
}
```

### 4) 모니터링

```
임계치 알람:
  cluster_state 크기 > 50MB
  활성 샤드 수 > 5000 (노드당 800 × 클러스터 노드 수)
  primary shard 평균 크기 < 1GB → over-sharding 신호
  unassigned shards > 0 (지속)
```

```bash
# 정기 체크 쿼리
GET /_cluster/health
GET /_cat/allocation?v
GET /_cat/indices?v&h=index,docs.count,store.size,pri,rep&s=store.size:asc
```

### 5) 안티패턴

- 일별 인덱스 + 1일 데이터가 1GB 미만 → ILM rollover로 합칠 것
- 쓰지도 않을 replica 2~3개 → 1개면 충분 (cold tier는 0)
- 서비스별 × 환경별 × 시간별 인덱스 폭증 → 차원 줄이기 또는 한 인덱스에 keyword 필터로
- "혹시 모르니 샤드 5개" → 인덱스가 작으면 1개로 충분

**핵심: 샤드는 비싸다. "데이터 크기 / 권장 크기"로 샤드 수를 역산하고, ILM rollover로 자동화하라.**
