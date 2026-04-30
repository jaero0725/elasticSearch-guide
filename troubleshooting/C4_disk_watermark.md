# C4. Disk Watermark — 디스크 95% 도달, 인덱스 read-only 락

## 증상

```
{
  "type": "cluster_block_exception",
  "reason": "index [logs-2025.04] blocked by:
             [TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark,
              index has read-only-allow-delete block];"
}
```

또는 클러스터 로그에:

```
[WARN][o.e.c.r.a.DiskThresholdMonitor] flood stage disk watermark
[95%] exceeded on [es-data-2][...][/var/lib/elasticsearch] free: ...,
all indices on this node will be marked read-only
```

영향:

- 모든 색인이 `cluster_block_exception`으로 거부
- Kibana 모니터링 데이터도 못 들어와서 가시성 저하
- Logstash/Filebeat 큐가 폭증

---

## 원인

### Disk Watermark 3단계

ES는 디스크 사용량을 보고 자동으로 단계적 보호를 한다.

```
low watermark    (기본 85%)  : 새 샤드를 이 노드에 할당 안 함
high watermark   (기본 90%)  : 기존 샤드를 다른 노드로 재배치
flood stage      (기본 95%)  : 모든 인덱스 read-only-allow-delete 락
```

flood stage가 발동되면 **삭제는 가능하지만 색인은 불가**. 이 락은 디스크가 회복되어도 자동으로 해제되지 않는 경우가 있다 (8.x에서는 회복 시 자동 해제 시도, 그러나 케이스에 따라 수동 필요).

### 흔한 발생 시나리오

```
1. 로그가 갑자기 폭주 (배포 사고, 장애 알람 폭증)
2. ILM 정책 누락 → 오래된 인덱스 누적
3. searchable snapshot 전환 안 됨
4. 노드 1대 다운 → 남은 노드들에 샤드 몰림
5. 큰 reindex / restore 도중 디스크 압박
6. heap dump 또는 OOM 덤프 파일이 디스크 점유
```

---

## 진단

### 1) 클러스터 디스크 사용량

```bash
GET /_cat/allocation?v&h=node,shards,disk.indices,disk.used,disk.avail,disk.total,disk.percent

# 응답 예 (위험):
# node       shards  disk.indices  disk.used  disk.avail  disk.total  disk.percent
# es-data-1  120     1.8tb         1.85tb     150gb       2tb         92
# es-data-2  118     1.9tb         1.95tb     50gb        2tb         97   ← flood stage
# es-data-3  121     1.7tb         1.75tb     250gb       2tb         87
```

### 2) read-only 락 상태 확인

```bash
GET /_cluster/health?level=indices&filter_path=indices.*.status

# 인덱스별 settings로 직접 확인
GET /logs-*/_settings/index.blocks*?flat_settings=true

# 응답 예:
# "logs-2025.04" : {
#   "settings": { "index.blocks.read_only_allow_delete": "true" }   ← 락 걸림
# }
```

### 3) Watermark 설정값

```bash
GET /_cluster/settings?include_defaults=true&filter_path=*.cluster.routing.allocation.disk*,*.defaults.cluster.routing.allocation.disk*

# 기본값:
# cluster.routing.allocation.disk.threshold_enabled: true
# cluster.routing.allocation.disk.watermark.low:        "85%"
# cluster.routing.allocation.disk.watermark.high":      "90%"
# cluster.routing.allocation.disk.watermark.flood_stage:"95%"
# cluster.routing.allocation.disk.watermark.flood_stage.frozen: "95%"
```

### 4) 디스크에서 가장 큰 인덱스

```bash
GET /_cat/indices?v&h=index,docs.count,store.size,creation.date.string&s=store.size:desc&bytes=gb

# top 5에서 오래되거나 불필요한 인덱스 식별
```

### 5) OS 레벨 디스크

```bash
df -h /var/lib/elasticsearch
du -sh /var/lib/elasticsearch/*/*

# heap dump 등 ES 외 파일이 디스크 점유 중인지
ls -lh /var/log/elasticsearch/*.hprof 2>/dev/null
```

---

## 해결

### 단계 1: 즉각 — 공간 확보

#### 1.1 오래된 인덱스 삭제

```bash
# 가장 오래된 인덱스부터 (보존 정책 검토 후)
DELETE /logs-2024.01.*

# 시간 기반 인덱스 패턴
DELETE /logs-2024.01.0[1-9],/logs-2024.01.1[0-9],/logs-2024.01.2[0-9]
```

#### 1.2 close 후 데이터 보존하며 공간 확보

```bash
# 즉시 메모리는 회수되지만 디스크는 그대로 (수단 안 됨)
POST /logs-2024.01.*/_close

# 더 효율적: snapshot 후 삭제
PUT /_snapshot/s3_repo/snap-2024-01
{ "indices": "logs-2024.01.*" }

DELETE /logs-2024.01.*
```

#### 1.3 ES 외부 큰 파일 정리

```bash
# heap dump (잘못 남은 .hprof)
rm /var/log/elasticsearch/*.hprof

# 오래된 로그
find /var/log/elasticsearch -name "*.log.gz" -mtime +14 -delete
```

### 단계 2: read-only 락 해제

디스크 회복 후에도 락이 남아 있으면 명시적으로 해제:

```bash
# 클러스터 전체 모든 인덱스의 read-only 해제
PUT /*/_settings
{
  "index.blocks.read_only_allow_delete": null
}

# 특정 인덱스만
PUT /logs-2025.04/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```

> `null` 값이 설정 제거. `false` 도 동작하지만 명시적 제거 권장.

### 단계 3: 샤드 재배치 — 노드 간 균형

```bash
# 한 노드만 디스크 차고 다른 노드는 여유 있을 때 강제 재분배
POST /_cluster/reroute?retry_failed=true

# 자동 rebalance 확인
GET /_cluster/settings?include_defaults=true&filter_path=*.cluster.routing.rebalance*

# 일시적으로 rebalance 더 적극적으로
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.cluster_concurrent_rebalance": 4,
    "cluster.routing.allocation.node_concurrent_recoveries": 4
  }
}
```

### 단계 4: Watermark 임시 조정 (응급 시)

```bash
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low":         "90%",
    "cluster.routing.allocation.disk.watermark.high":        "93%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "97%"
  }
}
```

> **주의**: 임시 회피일 뿐. flood stage 95% → 97%로 늘리면 진짜 디스크 풀 가까워질 위험.

### 단계 5: 노드 추가 / 디스크 확장

#### 클라우드 (EBS, GP3 등)

```bash
# 디스크 확장 후 OS에서 파일시스템 확장
sudo growpart /dev/nvme1n1 1
sudo resize2fs /dev/nvme1n1p1   # ext4
sudo xfs_growfs /var/lib/elasticsearch  # xfs

# ES는 자동 인식
GET /_cat/allocation?v
# disk.total 이 늘어나면 OK
```

#### 노드 추가

새 노드 합류 후 자동 rebalance.

---

## 예방

### 1) ILM 정책 강제

```json
PUT /_ilm/policy/standard-logs
{
  "policy": {
    "phases": {
      "hot":  { "actions": { "rollover": { "max_age": "1d", "max_primary_shard_size": "50gb" } } },
      "warm": { "min_age": "7d", "actions": { "shrink": { "number_of_shards": 1 }, "forcemerge": {} } },
      "cold": { "min_age": "30d", "actions": { "searchable_snapshot": { "snapshot_repository": "s3_repo" } } },
      "delete": { "min_age": "90d", "actions": { "delete": { "delete_searchable_snapshot": true } } }
    }
  }
}
```

### 2) 모니터링 알람

```
임계치:
  disk.percent > 75%    → warning (주간 정기 점검)
  disk.percent > 85%    → low watermark 도달 (긴급 조치)
  disk.percent > 90%    → critical
  disk.percent > 95%    → flood stage (즉시 대응)
```

```bash
# 정기 체크 쿼리
GET /_cat/allocation?v&h=node,disk.percent,disk.avail
GET /_cluster/health
```

### 3) 디스크 사용량 예측

```
여유 기준:
  최소 free space:  단일 인덱스의 가장 큰 primary 샤드 크기 × 2
  ILM warm/cold:    별도 tier 노드로 분리
  snapshot 공간:    별도 repo (S3, GCS, MinIO)

성장률 추적:
  지난 30일 store.size 증가 추세 → 향후 30일 디스크 필요량 추정
```

### 4) Watermark 조정 (영구)

큰 디스크 (수 TB)에서는 5% = 수십~수백 GB. 절대값으로 설정하는 편이 안전:

```bash
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low":          "200gb",
    "cluster.routing.allocation.disk.watermark.high":         "100gb",
    "cluster.routing.allocation.disk.watermark.flood_stage":  "50gb"
  }
}
```

> "노드별 free 50GB 이하면 flood stage 락" 의미. % 보다 직관적이고 큰 디스크에 안전.

### 5) 스냅샷 자동화

```json
# SLM (Snapshot Lifecycle Management)
PUT /_slm/policy/daily-snapshots
{
  "schedule": "0 30 1 * * ?",      // 매일 01:30
  "name":     "<daily-{now/d}>",
  "repository": "s3_repo",
  "config": {
    "indices": ["logs-*", "metrics-*"]
  },
  "retention": {
    "expire_after": "30d",
    "min_count":    7,
    "max_count":    50
  }
}
```

### 6) 안티패턴

- ILM 없이 일별 인덱스 무한 누적
- 노드 1개의 디스크가 다른 노드의 2배 → 그 노드만 빨리 참
- 디스크 모니터링 없이 운영
- searchable snapshot 도입 안 한 cold tier
- 80% 차면 노드 추가 미루다가 95% 도달

**핵심: ILM + searchable snapshot + 모니터링 = 디스크 락 예방의 3단 콤보.**
