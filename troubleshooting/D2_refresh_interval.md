# D2. Refresh Interval — 1s 고집으로 세그먼트 폭증

## 증상

- 색인은 잘 들어가는데 검색 응답 시간이 점점 늘어남
- 인덱스의 세그먼트 수가 매우 큼 (수천~수만 개)
- 백그라운드 머지 스레드가 계속 바쁨
- I/O wait이 높음
- heap 사용량이 만성적으로 높음 (segment metadata)

```bash
GET /_cat/segments/events?v&s=size:asc

# 응답 예 (문제 있는 인덱스):
# index   shard  segment  docs.count  size
# events  0      _0       50          5kb
# events  0      _1       45          4kb
# events  0      _2       60          6kb
# ... (수천 줄)
```

또는:

```bash
GET /_cat/indices/events?v&h=index,segments.count,refresh.total

# segments.count = 12000  ← 정상은 인덱스당 < 100
```

---

## 원인

### Refresh / Segment의 라이프사이클

```
색인 흐름:

  POST /events/_doc { ... }
        ↓
   in-memory buffer (검색 불가)
        ↓ refresh (기본 1초)
   새 segment 생성 (검색 가능)
        ↓ background merge
   여러 segment를 더 큰 segment로 병합
```

각 refresh = 새 segment 1개. **refresh 간격이 짧을수록 segment가 자주 만들어진다.**

### 1초 refresh의 문제

```
색인량이 적을 때:
  1초마다 refresh → 1초마다 segment 1개
  하루 86,400개 segment 생성 (한 샤드에)

머지가 따라가야 하는 양:
  머지는 비례 비용 (segment 크기 + 개수)
  segment 수가 많아질수록 머지 비용 ↑
  CPU/IO 자원 잠식 → 색인 속도 저하

heap 영향:
  segment 1개당 metadata 수십~수백 KB
  10000 segment = 수 GB heap
```

### 1초가 항상 좋은 건 아니다

```
실시간 검색이 필요한 경우 (UI 실시간 결과):
  refresh = 1s → 검색 가능까지 1초 지연

배경 분석/로그/모니터링:
  refresh = 30s ~ 60s 도 충분
  → segment 수 30~60배 감소 → 머지/heap 부담 격감
```

---

## 진단

### 1) 인덱스의 refresh 설정

```bash
GET /events/_settings?include_defaults=true&filter_path=*.*.index.refresh_interval,*.defaults.index.refresh_interval

# 응답 예:
# "events": { "settings": { "index": { "refresh_interval": "1s" } } }
# 명시 없으면 기본 1초
```

### 2) Segment 수 / 분포

```bash
# 인덱스별 segment 카운트
GET /_cat/indices?v&h=index,docs.count,segments.count,store.size&s=segments.count:desc

# 응답 예 (위험):
# index            docs.count  segments.count  store.size
# events           5000000     8500            45gb
# events-old       1000000     200             8gb     ← 정상
```

```bash
# Segment 크기 분포
GET /_cat/segments/events?v&s=size:asc

# 작은 segment(< 100KB)가 많으면 문제
```

### 3) Refresh / Merge 통계

```bash
GET /_nodes/stats/indices/refresh,merge?human

# 핵심 지표:
# refresh.total       : 총 refresh 횟수
# refresh.total_time  : 총 refresh 시간
# merges.current      : 현재 진행 중인 머지
# merges.total_time   : 총 머지 시간
# merges.total_throttled_time : throttle 시간 (자원 부족)
```

```bash
# 더 자세히
GET /events/_stats/refresh,merge?human

# refresh.total_time / refresh.total = 평균 refresh 시간
# 평균이 100ms+면 너무 자주 / 너무 큰 segment
```

### 4) 머지 throttle 확인

```bash
GET /_cat/thread_pool/refresh,force_merge?v&h=node_name,name,active,queue,completed
```

### 5) 실제 검색이 얼마나 자주 필요한가?

```
기준 질문:
  "사용자가 색인 후 결과를 N초 안에 보아야 하는가?"
   N = 1초     → refresh 1s
   N = 10초    → refresh 10s (또는 5s)
   N = 30~60초 → refresh 30s
   배경 분석   → refresh 60s ~ 5분
```

---

## 해결

### 단계 1: refresh_interval 조정

```bash
# 인덱스별 (운영 중에도 변경 가능)
PUT /events/_settings
{
  "index.refresh_interval": "30s"
}

# 비활성화 (refresh 강제 호출 시에만)
PUT /events/_settings
{
  "index.refresh_interval": "-1"
}
# → 색인 후 검색에 보이려면 명시적 POST /events/_refresh
```

### 단계 2: 인덱스 템플릿에서 기본값 설정

```json
PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index.refresh_interval": "30s",
      "index.number_of_shards":   1,
      "index.number_of_replicas": 1
    }
  }
}
```

### 단계 3: 기존 segment 정리 — force_merge

```bash
# 정상 인덱스 1개 segment로 머지 (read-heavy 인덱스에 권장)
POST /events-2025.04.01/_forcemerge?max_num_segments=1

# 비동기 (대용량 인덱스)
POST /events-2025.04.01/_forcemerge?max_num_segments=1&wait_for_completion=false

# 진행 상태
GET /_tasks?actions=indices:admin/forcemerge*&detailed=true
```

> **주의**: force_merge는 비용이 크다 (모든 segment 다시 쓰기). 색인 중인 인덱스에는 권장하지 않음. 일반적으로 ILM `warm` 단계에서 자동 실행.

### 단계 4: 대량 적재 시 refresh 일시 정지

```bash
# 적재 시작 전
PUT /events/_settings
{
  "index.refresh_interval": "-1",
  "index.number_of_replicas": 0
}

# (대량 bulk 색인...)

# 적재 종료 후
PUT /events/_settings
{
  "index.refresh_interval": "30s",
  "index.number_of_replicas": 1
}
POST /events/_refresh
```

### 단계 5: ILM에서 단계별 refresh 차등화

```json
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_age": "1d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "1d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      }
    }
  }
}
```

수동으로 warm 단계 인덱스에 refresh 비활성:

```bash
PUT /logs-2025.04.20/_settings
{
  "index.refresh_interval": "-1"   // warm 단계에선 더 이상 색인 안 함
}
```

---

## 예방

### 1) Workload별 refresh 가이드

```
Workload                     권장 refresh_interval
────────────────────────────────────────────────
실시간 검색 (UI)              1s ~ 5s
사용자 행동 이벤트            5s ~ 10s
로그 (Kibana 모니터링)        10s ~ 30s
메트릭/시계열                 30s ~ 60s
배치 분석/감사로그            60s ~ 5m
대량 적재 도중                -1 (수동 refresh)
```

### 2) 모니터링

```bash
# 정기 체크
GET /_cat/indices?v&h=index,segments.count,docs.count&s=segments.count:desc

# 알람 임계치:
#   segments.count > 200/샤드     → warning
#   segments.count > 1000/샤드    → critical
```

### 3) wait_for / refresh 옵션 활용

색인 직후 검색이 필요한 시나리오에서, refresh_interval을 짧게 두는 대신 클라이언트가 명시:

```bash
# 이 색인 직후 한정해서 즉시 refresh
POST /events/_doc?refresh=true
{ "user_id": 100 }

# 더 효율적: refresh가 일어날 때까지 대기 (cluster 부하 ↓)
POST /events/_doc?refresh=wait_for
{ "user_id": 100 }
```

`wait_for`는 강제 refresh를 일으키지 않고, 다음 자동 refresh를 기다림 → 일반적인 색인은 refresh를 30s로 두면서도 특정 요청만 즉시 검색 가능.

### 4) 머지 정책 튜닝 (드물게)

```bash
# 기본 정책: TieredMergePolicy
# 일반적으로 변경 불필요
PUT /events/_settings
{
  "index.merge.policy.max_merged_segment": "5gb"
  // 너무 큰 segment 머지 방지
}
```

### 5) 안티패턴

- 모든 인덱스에 refresh_interval=1s 일률 적용
- refresh_interval 변경 후 force_merge 안 함 → 기존 작은 segment 그대로 남음
- 대량 적재 + replica 1+ + refresh 1s → 처리량 절반 이하
- `?refresh=true` 매 색인마다 호출 (강제 segment 생성)
- 운영 중 인덱스에 force_merge 자주 호출

**핵심: refresh = "색인 후 검색에 보이기까지 시간". 그 SLA에 맞는 가장 큰 값을 쓰라.**
