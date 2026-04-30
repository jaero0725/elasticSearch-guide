# D1. 단건 색인 — 처리량 저하

## 증상

- ES 클러스터 사양은 충분한데 색인 처리량(EPS)이 기대치의 1/10 수준
- 색인 latency 평균은 작은데 (10~30ms) 총 처리량이 안 늘어남
- ES 노드 CPU 사용률은 30~50% 수준 (idle한데 안 받아들임)
- 클라이언트 쪽에서 connection pool exhaustion / TLS handshake 폭증
- Refresh/translog 비용이 상대적으로 큼

---

## 원인

### 단건 INSERT의 진짜 비용

```
HTTP 요청 1건 = ES 한 번 색인 = 다음 비용 모두 발생:
  ① TCP/TLS handshake (재사용 안 하면 매번)
  ② HTTP 요청 파싱
  ③ 코디네이팅 노드 → primary 샤드 노드 라우팅
  ④ Lucene IndexWriter 호출
  ⑤ translog write + fsync (durability)
  ⑥ 응답 반환

이 오버헤드는 "1 doc"이든 "1000 doc"이든 거의 동일.
즉 단건 색인은 오버헤드가 페이로드보다 큼.
```

### 처리량 비교 (예시)

```
단건 색인:  애플리케이션이 1000 doc 보내려면
  → HTTP 1000번 + fsync 1000번 = 약 30 ~ 50초

Bulk (1 req, 1000 doc):
  → HTTP 1번 + fsync 1번 = 약 0.5 ~ 2초

처리량 차이: 20~100배
```

### 왜 자주 발생하나

- 마이그레이션 스크립트가 row-by-row로 INSERT
- ORM/SDK가 기본 single insert
- 메시지 큐 컨슈머가 메시지마다 즉시 색인
- "버퍼링은 복잡하니까 일단 단건으로" 라는 결정

---

## 진단

### 1) 색인 처리량 / latency

```bash
# 노드별 색인 통계
GET /_cat/nodes?v&h=name,indexing.index_total,indexing.index_time,indexing.index_current

# 더 자세히
GET /_nodes/stats/indices/indexing?human
```

### 2) thread pool 상태

```bash
GET /_cat/thread_pool/write?v&h=node_name,name,active,queue,rejected,completed

# 응답 예 (정상 부하):
# node_name   name   active  queue  rejected  completed
# es-data-1   write  0       0      0         150000
# es-data-2   write  0       0      0         152000

# 응답 예 (단건 색인 폭주):
# node_name   name   active  queue  rejected  completed
# es-data-1   write  16      200    1500      150000   ← rejected 발생
```

### 3) Translog 통계

```bash
GET /_cat/indices?v&h=index,segments.count,refresh.total_time,flush.total_time

# 또는 단일 인덱스 자세히
GET /events/_stats/translog,refresh,flush?human
```

### 4) 클라이언트 패턴 확인

```
- 애플리케이션 로그에서 ES 호출 빈도
- ES 슬로우 인덱싱 로그
- Connection pool 메트릭
```

```bash
# Indexing slow log
PUT /events/_settings
{
  "index.indexing.slowlog.threshold.index.warn":  "1s",
  "index.indexing.slowlog.threshold.index.info":  "500ms"
}
```

### 5) 단일 색인 vs Bulk 비율

```
공식 클라이언트 SDK 메트릭에서:
  total_requests / total_documents
  비율이 1:1에 가까우면 단건 색인 의심
  비율이 1:1000+ 면 bulk 사용 중
```

---

## 해결

### 단계 1: Bulk API로 묶기

```bash
POST /_bulk
{ "index": { "_index": "events", "_id": "1" } }
{ "user_id": 100, "action": "click", "ts": "2025-04-30T10:00:00Z" }
{ "index": { "_index": "events", "_id": "2" } }
{ "user_id": 101, "action": "view", "ts": "2025-04-30T10:00:01Z" }
{ "index": { "_index": "events", "_id": "3" } }
{ "user_id": 102, "action": "purchase", "ts": "2025-04-30T10:00:02Z" }
```

```python
# Python 의사코드
from elasticsearch.helpers import bulk

actions = []
for msg in stream:
    actions.append({
        "_op_type": "index",
        "_index":   "events",
        "_source":  msg
    })
    if len(actions) >= 1000:
        bulk(es, actions)
        actions = []
```

### 단계 2: Bulk 사이즈 결정

```
권장 시작점: 5~15MB per bulk (또는 1000~5000 docs)

너무 작음:  단건과 비슷한 오버헤드
너무 큼:    timeout, 메모리 부담, 부분 실패 시 재처리 비용 큼

확인 방법: bulk 응답 시간 측정
  500ms 미만이면 더 키워볼 수 있음
  2초 이상이면 줄이는 편이 안전
```

### 단계 3: 시간 기반 flush (배치가 차지 않아도)

```python
import time

actions = []
last_flush = time.time()

for msg in stream:
    actions.append(...)
    if len(actions) >= 1000 or (time.time() - last_flush) > 2:
        bulk(es, actions)
        actions = []
        last_flush = time.time()
```

### 단계 4: 클라이언트 connection 재사용

```python
# 나쁨: 매 요청마다 클라이언트 생성
def index_one(doc):
    es = Elasticsearch(...)
    es.index(...)

# 좋음: 싱글톤 / 풀
es = Elasticsearch(["..."], maxsize=25)   # connection pool 25
def index_one(doc):
    es.index(...)
```

### 단계 5: 동시성 활용

여러 워커가 병렬로 bulk 보내기. 단, ES write thread pool 한도(`thread_pool.write.queue_size = 1000`)를 의식.

```python
# parallel_bulk
from elasticsearch.helpers import parallel_bulk

for ok, info in parallel_bulk(es, actions, thread_count=4, chunk_size=1000):
    if not ok:
        print("err", info)
```

### 단계 6: refresh / replica 일시 조정 (대량 적재 시)

```bash
# 색인 시작 전
PUT /events/_settings
{
  "index.refresh_interval": "60s",
  "index.number_of_replicas": 0
}

# 색인 종료 후 복구
PUT /events/_settings
{
  "index.refresh_interval": "5s",
  "index.number_of_replicas": 1
}

# refresh 강제
POST /events/_refresh
```

```
효과: refresh가 색인 사이에 끼어들지 않음 → 처리량 5~10배
주의: 색인 도중 검색 결과가 즉각 반영되지 않음
```

### 단계 7: Async Bulk (Beats / Logstash)

Filebeat, Logstash, Fluentd 같은 수집기는 내부적으로 bulk + 큐를 자동 처리. 직접 단건 색인 코드를 짜지 말고 표준 수집기 사용 검토.

```yaml
# filebeat.yml
output.elasticsearch:
  bulk_max_size: 4096
  worker: 4
  compression_level: 3
```

---

## 예방

### 1) 클라이언트 표준 패턴

```
Java:           BulkProcessor (자동 flush)
Python:         elasticsearch-py helpers.bulk / streaming_bulk
Node.js:        client.helpers.bulk
Go:             esutil.BulkIndexer
```

대부분 다음을 자동 처리:
- 사이즈 또는 시간 기반 flush
- 재시도 / backoff
- 부분 실패 핸들링

### 2) 처리량 모니터링

```
KPI:
  events/sec (색인 처리량)
  bulk_size_avg (bulk 당 평균 문서 수)
  client_response_time

이상치:
  bulk_size_avg < 10 → 단건 또는 작은 bulk → 패턴 점검
  rejected > 0 → 너무 빨리 보냄 → 백프레셔 도입
```

### 3) write thread pool 모니터

```bash
GET /_cat/thread_pool/write?v&h=node_name,active,queue,rejected
# rejected > 0 발생 시:
#   - bulk 사이즈 키우기
#   - 클라이언트 동시성 줄이기
#   - 노드 추가
```

### 4) 운영 가이드

```
색인 처리량별 권장:
  < 100 EPS:    단건도 가능 (그래도 bulk 권장)
  100~1k EPS:  bulk 1~5초 간격, 100~500개씩
  1k~10k EPS:  bulk + 병렬 워커
  > 10k EPS:   Logstash/Filebeat + replica/refresh 튜닝
```

### 5) 안티패턴

- 단건 색인 + 매 호출마다 client 새로 생성
- bulk 사이즈가 너무 큼 (50MB+) → timeout/실패 위험
- bulk 응답 무시 (부분 실패 검출 안 함)
- refresh_interval=1s + 1 EPS 색인 → segment 폭증 (D2 참고)
- replica 2~3 + 대량 적재 → 처리량 절반 이하

**핵심: ES는 bulk 처리에 최적화되어 있다. "1 row 1 INSERT" 마인드는 RDB 시절 습관일 뿐.**
