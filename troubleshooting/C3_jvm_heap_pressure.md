# C3. JVM Heap Pressure — Heap 90%, GC 폭증, Circuit Breaker

## 증상

```
{
  "type": "circuit_breaking_exception",
  "reason": "[parent] Data too large, data for [<http_request>] would be
             [12.5gb], which is larger than the limit of [12.4gb]"
}
```

또는 다음 패턴:

- 노드 heap 사용률이 항상 85~95%
- Old GC pause가 길어짐 (`[gc][young][...] [gc-time=8.5s]`)
- 색인/검색 응답 시간이 들쭉날쭉 (GC 동안 멈춤)
- 결국 노드가 비정상 종료 (`OutOfMemoryError`)
- Hot threads에 GC 스레드가 자주 보임

---

## 원인

### Heap 압박의 주요 원인

```
1. Heap 크기 부족             → 워크로드에 비해 작게 설정
2. fielddata 로드             → text 필드 정렬/집계 (B3 참고)
3. 큰 집계 결과               → terms agg with size:100000
4. 매핑 폭증                  → cluster state metadata heap 점유 (A1, A3)
5. 너무 많은 샤드             → 샤드별 Lucene metadata heap (C1)
6. 큰 _source / scripts       → 한 번에 fetch 양 과다
7. JVM 옵션 잘못             → G1GC vs CMS, heap > 32GB compressed oops 손실
```

### Heap 32GB 이상의 함정

```
JVM의 compressed oops:
  heap < 32GB  → 객체 포인터 4바이트 (효율적)
  heap ≥ 32GB  → 객체 포인터 8바이트, 메모리 효율 급락

→ heap을 30~31GB로 설정하는 것이 64GB로 키우는 것보다 빠른 경우가 많음
→ ES 권장: heap 최대 30GB
```

### Circuit breaker

ES는 OOM을 막기 위해 여러 단계의 circuit breaker를 둔다:

```
parent       : 전체 (기본 95%)
fielddata    : fielddata 로드 (기본 40%)
request      : 단일 요청 (기본 60%)
in_flight    : transport 메시지 (기본 100%)
accounting   : 추적 정보 (기본 100%)
```

`tripped > 0`이면 이미 차단된 적이 있다는 뜻.

---

## 진단

### 1) Heap 사용률 확인

```bash
GET /_cat/nodes?v&h=name,heap.percent,heap.current,heap.max,ram.percent,cpu,load_1m

# 응답 예 (위험):
# name        heap.percent  heap.current  heap.max  ram.percent  cpu  load_1m
# es-data-1   92            27.6gb        30gb      88           75   3.8
# es-data-2   89            26.7gb        30gb      87           70   3.2
```

### 2) GC 통계

```bash
GET /_nodes/stats/jvm?human

# 핵심 지표:
# jvm.gc.collectors.old.collection_count
# jvm.gc.collectors.old.collection_time
# jvm.gc.collectors.young.collection_time
# 두 시점 차이로 GC 시간 비율 계산:
#   old_gc_time_per_min = (last - prev) / 시간
#   1분에 5초 이상이면 위험
```

### 3) Circuit breaker 상태

```bash
GET /_nodes/stats/breaker?human

# tripped > 0 이면 발동한 적 있음
# 어떤 breaker가 자주 trip 되는지가 원인 단서:
#   fielddata 자주 → text 필드 집계
#   request 자주 → 큰 단일 요청
#   parent 자주 → 누적 heap 압박
```

### 4) Heap 점유 분포

```bash
GET /_nodes/stats/indices/segments,fielddata,query_cache,request_cache?human

# 응답 예시:
# "segments": { "count": 12000, "memory_in_bytes": 1.2gb }
# "fielddata": { "memory_size_in_bytes": 8gb, "evictions": 100 }
# "query_cache": { "memory_size": 2gb }
# "request_cache": { "memory_size": 1gb }
```

### 5) 무거운 쿼리 추적

```bash
# 현재 도는 task
GET /_tasks?actions=*search*,*bulk*&detailed=true

# Slow log 활성화
PUT /_settings
{
  "index.search.slowlog.threshold.query.warn": "1s",
  "index.indexing.slowlog.threshold.index.warn": "1s"
}
```

### 6) Hot threads

```bash
GET /_nodes/hot_threads?threads=10&snapshots=10&interval=500ms

# GC, Lucene merge, search aggregation 등이 자주 보이면 패턴 식별
```

---

## 해결

### 단계 1: 즉각 — 큰 cache 비우기

```bash
# query cache, request cache 비우기
POST /_cache/clear

# fielddata 비우기 (가능하면)
POST /_cache/clear?fielddata=true

# 한 노드만
POST /_nodes/es-data-1/_cache/clear
```

### 단계 2: 무거운 쿼리 즉시 종료

```bash
# 도는 검색 task 확인
GET /_tasks?actions=*search*&detailed=true

# 의심되는 task 캔슬
POST /_tasks/<task_id>/_cancel
```

### 단계 3: Heap 크기 조정

```bash
# /etc/elasticsearch/jvm.options 또는 jvm.options.d/*.options
-Xms30g
-Xmx30g
# Xms == Xmx (동일하게 설정 필수)
# 30~31GB 까지가 효율적 (compressed oops)

# 8.x: 자동 heap sizing 활성화 가능
# config/jvm.options.d 에서 수동 설정이 없으면 ES가 자동으로 선택
```

> 컨테이너 환경에서 `cgroups`를 따라간다. K8s 메모리 limit의 50% 권장.

### 단계 4: 매핑/쿼리 개선

#### text 필드 fielddata 끄기 (→ B3)

```bash
PUT /products/_mapping
{
  "properties": {
    "description": { "type": "text", "fielddata": false }
  }
}
```

#### 큰 terms agg 분할

```bash
# 나쁨: 한 번에 100k bucket
"aggs": { "by_id": { "terms": { "field": "user_id", "size": 100000 } } }

# 좋음: composite aggregation으로 페이지네이션
"aggs": {
  "by_id": {
    "composite": {
      "size": 1000,
      "sources": [ { "user": { "terms": { "field": "user_id" } } } ]
    }
  }
}
```

#### `_source` 필드 줄이기

```bash
# 응답에 불필요한 본문 필드는 제외
GET /docs/_search
{
  "_source": { "includes": ["id", "title", "summary"], "excludes": ["body", "embedding"] },
  "size": 100
}
```

### 단계 5: Bulk 응답 / scroll 컨텍스트 정리

```bash
# 활성 scroll/PIT 확인
GET /_nodes/stats/indices/search?filter_path=**.open_contexts
# 활성 컨텍스트 너무 많으면 (수천 개) 메모리 점유

# 오래된 scroll 종료
DELETE /_search/scroll/_all
```

### 단계 6: Circuit breaker 한도 보수적으로

```bash
PUT /_cluster/settings
{
  "persistent": {
    "indices.breaker.fielddata.limit":  "30%",
    "indices.breaker.request.limit":    "40%",
    "indices.breaker.total.limit":      "85%"
  }
}
```

> 한도를 낮추면 큰 쿼리는 거부되지만 OOM은 막을 수 있다.

### 단계 7: 노드 추가 / 샤드 재분배

```bash
# 노드 추가 후 자동 재분배 또는 수동
POST /_cluster/reroute?retry_failed=true
```

---

## 예방

### 1) Heap 사이징 가이드

```
호스트 RAM의 50% 까지 (남은 50%는 Lucene mmap에)
단, 30~31GB 상한 (compressed oops)

예:
  64GB 머신 → heap 30GB, OS/Lucene 34GB
  128GB 머신 → heap 30GB, OS/Lucene 98GB (Lucene이 더 많이 활용)
  16GB 머신 → heap 8GB

데이터 노드 vs 마스터 노드:
  데이터: 30GB
  마스터: 8GB 충분 (cluster state만 다룸)
  coord:  16~30GB (요청 fan-out, agg merge)
```

### 2) JVM 옵션

```
-Xms30g
-Xmx30g
-XX:+UseG1GC                         # 8.x 기본
-XX:G1HeapRegionSize=8m              # heap 크기에 비례 (자동)
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30
```

### 3) 모니터링 알람

```
heap.percent > 85% (지속 5분)         → warning
heap.percent > 90% (지속 5분)         → critical
old GC time > 5초/분                  → warning
circuit breaker tripped 증가          → 매핑/쿼리 점검
```

```bash
# Prometheus exporter, Datadog, Elastic Monitoring 등 활용
GET /_cluster/stats
GET /_nodes/stats/jvm
```

### 4) 워크로드 패턴 점검

```
대형 집계 (size > 1000):           composite aggregation으로 분할
대량 fetch:                         search_after + PIT
text 정렬/집계:                     keyword 멀티필드 추가
큰 _source:                         필요한 필드만 fetch
모니터링/health check 호출 폭주:    Kibana/외부 모니터링 빈도 줄이기
```

### 5) 캐시 한도 명시

```yaml
# elasticsearch.yml
indices.queries.cache.size: "10%"      # 기본 10%
indices.fielddata.cache.size: "20%"    # 기본 무제한 → 명시 권장
```

### 6) 안티패턴

- Heap > 32GB (compressed oops 손실)
- Heap < 머신 RAM의 50% 미만 → Lucene mmap 비효율
- Xms ≠ Xmx (heap 동적 변경 비용)
- swap 활성화 (`bootstrap.memory_lock: true` 권장)
- 한 노드에 데이터 + 마스터 + Kibana 동거

**핵심: heap 크기는 상한 30GB. 운영 지표는 heap %, GC 시간, breaker tripped 3종 추적.**
