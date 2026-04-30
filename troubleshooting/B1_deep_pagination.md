# B1. Deep Pagination — `from + size > 10000`

## 증상

```
{
  "type": "illegal_argument_exception",
  "reason": "Result window is too large, from + size must be less than or equal
             to: [10000] but was [20000]. See the scroll api for a more efficient
             way to request large data sets. This limit can be set by changing
             the [index.max_result_window] index level setting."
}
```

또는 한도까지는 안 닿아도:

- `from=9000, size=100` 같은 깊은 페이지에서 응답이 매우 느림 (수 초~분)
- 응답마다 노드 CPU 사용률 스파이크
- 같은 데이터를 페이지로 넘기는 사이 새 문서가 들어와서 결과가 일관되지 않음

---

## 원인

### `from + size`의 동작

```
N개 샤드에 데이터가 분산:

  쿼리: from=9000, size=10

  ① 코디네이팅 노드가 모든 샤드에 "from=0, size=9010" 요청
  ② 각 샤드가 9010개씩 정렬해서 반환  → N × 9010 문서를 코디네이터에 모음
  ③ 코디네이터가 다시 글로벌 정렬 후 9000~9009 슬라이스 반환

문제:
  - 매 페이지마다 처음부터 다시 정렬
  - 페이지가 깊을수록 N × (from+size) 문서를 메모리에 들고 있어야 함
  - heap 사용량 폭증 가능
```

기본 한도 `index.max_result_window: 10000`은 안전장치. 한도를 키우는 게 아니라 **검색 방식을 바꿔야** 한다.

---

## 진단

### 1) 한도 확인

```bash
GET /my-index/_settings?include_defaults=true&filter_path=*.*.index.max_result_window,*.defaults.index.max_result_window
```

### 2) 깊은 페이지 쿼리가 실제로 도는지

```bash
# Slow log 활성화
PUT /my-index/_settings
{
  "index.search.slowlog.threshold.query.warn":  "1s",
  "index.search.slowlog.threshold.fetch.warn": "1s"
}
```

slow log에 `from=9000` 같은 패턴이 자주 보이면 클라이언트 코드 확인.

### 3) 호출 통계

```bash
GET /_cat/indices/my-index?v&h=index,search.query_total,search.query_time
GET /_nodes/stats/indices/search
```

---

## 해결

### 케이스 A: 사용자 페이지네이션 (1, 2, 3, ... 페이지)

대부분의 UX는 100페이지 이상 안 보여줘도 된다. 한도를 그대로 두고 클라이언트에서 막는 게 정답.

```javascript
// 프론트
const MAX_PAGE = 100;  // (10페이지 * 100건 = 1000)
if (page * pageSize > 10000) showOnlyTopN();
```

### 케이스 B: 스크롤/내보내기 (모든 결과 순회) — `search_after` + PIT

8.x에서는 `search_after` + Point-In-Time(PIT)이 표준.

```bash
# 1) PIT 생성 (해당 시점의 일관된 스냅샷)
POST /my-index/_pit?keep_alive=5m

# 응답
# { "id": "46ToAwMDaWR4..." }
```

```bash
# 2) 첫 페이지 — sort 필수, tiebreaker 필요
POST /_search
{
  "size": 1000,
  "query": { "match_all": {} },
  "pit": {
    "id": "46ToAwMDaWR4...",
    "keep_alive": "5m"
  },
  "sort": [
    { "@timestamp":   "asc" },
    { "_shard_doc":   "asc" }   // tiebreaker (8.x 권장)
  ]
}
```

응답의 마지막 hit의 `sort` 값을 그대로 다음 요청의 `search_after`에 사용:

```bash
# 3) 다음 페이지
POST /_search
{
  "size": 1000,
  "query": { "match_all": {} },
  "pit": { "id": "46ToAwMDaWR4...", "keep_alive": "5m" },
  "sort": [
    { "@timestamp":   "asc" },
    { "_shard_doc":   "asc" }
  ],
  "search_after": ["2025-04-30T10:15:42.000Z", 12345]
}
```

```bash
# 4) 끝나면 PIT 종료
DELETE /_pit
{ "id": "46ToAwMDaWR4..." }
```

### `search_after`의 장점

```
from+size 방식:
  매 페이지마다 N × (from+size) 문서를 메모리에 → 깊은 페이지일수록 느림

search_after 방식:
  각 페이지가 "이 sort 값보다 큰 첫 size개" 만 가져옴
  → 깊이에 무관하게 일정한 비용
  → PIT 덕분에 페이지 도중 신규 문서가 결과에 끼어들지 않음
```

### 케이스 C: 레거시 `scroll` API

여전히 동작하지만 **신규 코드에서는 비권장** (8.x 이후). PIT + search_after가 표준.

```bash
# 옛날 방식 (참고용)
POST /my-index/_search?scroll=5m
{ "size": 1000, "query": { "match_all": {} } }

# 다음 페이지
POST /_search/scroll
{ "scroll": "5m", "scroll_id": "..." }

# 종료
DELETE /_search/scroll
{ "scroll_id": "..." }
```

scroll의 단점:
- 스크롤 컨텍스트가 무거워서 동시 스크롤이 많으면 메모리 압박
- 페이지 사이의 신규 데이터를 보지 못함 (PIT도 동일하지만, scroll은 양보 없이 자원 점유)

### 한도 임시 상향 (응급용)

```bash
PUT /my-index/_settings
{
  "index.max_result_window": 50000
}
```

> 권장하지 않음. 메모리 위험만 키운다.

---

## 예방

### 1) 페이지네이션 설계 시점에 결정

```
사용자 UI:
  - 페이지 1~100, 1페이지 20건  → 최대 from=2000+size=20  → 안전
  - 무한 스크롤              → search_after 권장

내보내기/배치:
  - search_after + PIT이 정답
```

### 2) PIT keep_alive 적절히

```
짧을수록 자원 절약, 길어지면 segment merge 지연 가능.
일반적으로:
  대화형 검색 페이지네이션: 1~5분
  배치 export:              30분 ~ 수 시간 (데이터 양에 따라)

활성 PIT 확인:
  GET /_nodes/stats/indices/search?filter_path=**.open_contexts
```

### 3) 쿼리 패턴 단순화

대량 데이터 export가 자주 필요하다면, ES 대신 별도 OLAP/DW로 보내는 것을 검토:

```
ES = 검색·실시간 분석 워크로드에 최적화
대량 ETL = ES → BigQuery / Snowflake / ClickHouse 로 sink
```

### 4) 운영 가이드

```
권장 한도:
  from + size ≤ 1000  : 일반 UI
  from + size ≤ 10000 : 가능하지만 비효율
  > 10000             : search_after + PIT 사용

체크 쿼리:
  slow log + from 패턴 모니터링
  활성 PIT 수 (메모리 사용량 추정)
```

**핵심: from+size는 페이지 ≤ 100건의 UI 전용. 그 이상은 search_after + PIT.**
