# 10장. 안티패턴

Elasticsearch는 강력한 도구지만, 관계형 DB 또는 OLTP의 직관을 그대로 가져오면 클러스터가 무너진다. 이 장은 운영 현장에서 반복적으로 관찰되는 안티패턴 12가지와 그 대안을 정리한다. 각 항목은 "왜 위험한가 → 어떤 증상이 나오는가 → 어떻게 고치는가" 순으로 읽으면 좋다.

---

## 10.1 Mapping을 dynamic: true로 두기

### 위험

기본값(`dynamic: true`)에서 새 필드는 자동으로 mapping에 추가된다. 예쁘게 작동하다가 다음과 같은 일이 일어난다.

```
첫 문서:  { "user_id": 12345 }            → user_id: long
다음 문서: { "user_id": "u-abc-12345" }   → 색인 실패 (mapper_parsing_exception)
```

또는 더 나쁘게:

```
앱 v1:    { "ts": "2026-04-30" }          → ts: date
앱 v2:    { "ts": 1714435200000 }         → 같은 필드, 다른 의미. 정렬 깨짐.
```

### 증상

- "mapper_parsing_exception: failed to parse field [...]"
- Mapping explosion (`Limit of total fields [1000] has been exceeded`)
- Heap에 FieldMapper 객체 폭증 → cluster state 비대화

### 대안

```json
PUT /my-index
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "user_id":   { "type": "keyword" },
      "@timestamp":{ "type": "date" },
      "message":   { "type": "text" }
    }
  }
}
```

- `strict`: 정의되지 않은 필드 → 거부 (개발 단계 권장)
- `false`: 정의되지 않은 필드 → 저장은 하되 색인은 안함 (조용한 데이터 유입)
- `runtime`: 런타임 필드로 등록 (스키마 진화 유연성)

명시적 매핑 + 인덱스 템플릿 + 컴포넌트 템플릿 조합이 정석.

→ 상세: [A3 dynamic mapping runaway](../troubleshooting/A3_dynamic_mapping_runaway.md)

---

## 10.2 text + keyword 혼동

### 위험

`text`와 `keyword`는 동일한 문자열에 대해 완전히 다른 일을 한다.

| 타입 | 분석 | 정렬 | 집계 | term query 정확 일치 |
|------|------|------|------|---------------------|
| `text`    | 토큰화·소문자화 | 불가(*) | 불가(*) | 불가 |
| `keyword` | 그대로 저장 | 가능 | 가능 | 가능 |

(*) `fielddata: true`로 강제 가능하나 heap 폭주 위험으로 권장하지 않음.

```
mapping: { "status": { "type": "text" } }
색인:    { "status": "ACTIVE" }
검색:    { "term": { "status": "ACTIVE" } }   → 미스 (text는 "active"로 분석됨)
정렬:    sort: ["status"]                       → "Fielddata is disabled..." 에러
```

### 대안: multi-field

```json
{
  "status": {
    "type": "text",
    "fields": {
      "keyword": { "type": "keyword", "ignore_above": 256 }
    }
  }
}
```

- 전문 검색: `status` (text 경로)
- 정확 일치 / 정렬 / 집계: `status.keyword`
- `ignore_above`: 길이 초과 시 색인 skip (cardinality 폭증 방지)

> 8.x dynamic template 기본값은 string 필드를 자동으로 `text + .keyword` multi-field로 매핑한다. 이를 신뢰하지 말고 **명시 정의**하는 것이 안전하다.

→ 상세: [A2 text vs keyword](../troubleshooting/A2_text_vs_keyword.md)

---

## 10.3 nested 남발

### 위험

`nested` 타입은 배열 내 객체의 필드 간 관계를 보존하기 위해 **배열 요소마다 hidden Lucene 문서**를 만든다.

```
{ "tags": [
    { "key": "env",  "value": "prod" },
    { "key": "team", "value": "search" }
  ]
}
```

이 문서 1건은 실제로는 **메인 문서 1 + nested 문서 2 = 3개의 Lucene 문서**가 된다.

### 증상

- 인덱스 크기와 segment 수가 예상보다 2~10배 큼
- `index.mapping.nested_objects.limit` (기본 10000) 초과 에러
- Aggregation 시 `nested` agg 누락 → 결과가 평탄화돼 잘못 나옴
- Join 비용으로 검색 latency 증가

### 대안

**옵션 A: flatten**

```json
{ "tags.env": "prod", "tags.team": "search" }
```

key가 고정 집합이라면 단순 평탄화가 가장 빠르다.

**옵션 B: flattened 타입**

```json
{ "tags": { "type": "flattened" } }
```

- 모든 sub-key를 하나의 keyword 필드처럼 취급
- 임의 sub-key 검색 가능
- 단, 분석/정렬/집계가 keyword 수준으로 제한됨

**옵션 C: nested 유지 (불가피한 경우)**

배열 요소 간 관계 검증이 진짜 필요할 때만. `index.mapping.nested_fields.limit`, `nested_objects.limit` 모니터링.

---

## 10.4 Deep pagination (from + size > 10000)

### 위험

```
GET /my-index/_search
{ "from": 100000, "size": 10 }
```

분산 검색에서 페이지 N을 반환하려면 **모든 shard가 from + size 만큼의 top 결과를 코디네이터로 보낸다**. 코디네이터는 그것을 다시 합치고 정렬한다.

```
shard 1 → top 100010 docs ─┐
shard 2 → top 100010 docs ─┼──→ coordinator merge → return last 10
shard 3 → top 100010 docs ─┘
```

`from + size > 10000`은 기본 `index.max_result_window`로 차단된다. 그 한계를 늘리지 마라.

### 대안

**search_after**: 정렬 키 + tiebreaker로 cursor 기반 페이징.

```json
{
  "size": 10,
  "sort": [
    { "@timestamp": "desc" },
    { "_id": "asc" }
  ],
  "search_after": [1714435200000, "doc-id-from-prev-page"]
}
```

**Point in Time (PIT)**: 일관된 스냅샷에서 search_after 사용.

```
POST /logs/_pit?keep_alive=1m
→ { "id": "..." }

GET /_search
{ "pit": { "id": "...", "keep_alive": "1m" }, "sort": [...], "search_after": [...] }
```

대량 export는 `_search/scroll` (deprecated 추세) 또는 PIT + search_after.

→ 상세: [B1 deep pagination](../troubleshooting/B1_deep_pagination.md)

---

## 10.5 Leading wildcard ("*foo")

### 위험

```
{ "wildcard": { "url": "*example.com*" } }
```

선행 와일드카드는 inverted index의 정렬을 활용할 수 없으므로 **모든 term을 스캔**해야 한다. 카디널리티가 높으면 사실상 풀스캔.

`index.allow_expensive_queries`가 false면 거부된다.

### 대안

- **Reverse keyword 트릭**: 별도 필드에 문자열을 reverse해서 저장 → 선행 와일드카드를 후행으로 변환
- **n-gram / edge_ngram analyzer**: 색인 시 부분 문자열을 미리 토큰화. 디스크 비용 증가.
- **wildcard 타입 (8.x)**: 부분 문자열 매칭에 최적화된 전용 타입
- **데이터 모델 재설계**: URL이라면 host, path를 분리해 keyword로

```json
{
  "url": { "type": "wildcard" }
}
```

`wildcard` 타입은 일반 `keyword`보다 부분 일치에 빠르다 (n-gram보다 디스크 효율적).

→ 상세: [B2 wildcard / regex](../troubleshooting/B2_wildcard_regex.md)

---

## 10.6 _all 필드 의존

### 위험

`_all`은 모든 필드를 합친 가상 필드였다. **6.0에서 deprecated, 7.x에서 제거됨.** 8.x에서는 존재하지 않는다.

```
GET /_search
{ "query": { "match": { "_all": "elasticsearch" } } }   → 작동 안 함
```

### 대안

**copy_to**: 명시적으로 합칠 필드를 지정.

```json
{
  "mappings": {
    "properties": {
      "title":   { "type": "text", "copy_to": "all_text" },
      "body":    { "type": "text", "copy_to": "all_text" },
      "all_text":{ "type": "text" }
    }
  }
}
```

**multi_match**: 검색 시 여러 필드 동시 매칭.

```json
{ "multi_match": { "query": "elasticsearch", "fields": ["title^2", "body"] } }
```

Boost(`^N`)도 지원해서 `_all`보다 유연하다.

---

## 10.7 단건 INSERT

9장의 반복:

- 단건 PUT/POST는 bulk 대비 처리량 1/100 이하
- 라우팅·acking 오버헤드가 요청당 동일
- thread_pool `write` queue가 빠르게 차서 429 폭주

대안: **`_bulk` API + 클라이언트 측 BulkProcessor**. 5–15 MiB / 1초 / 1000건 중 하나라도 차면 flush.

→ 상세: [D1 single-doc indexing](../troubleshooting/D1_single_doc_indexing.md)

---

## 10.8 작은 shard 수만 개 (oversharding)

### 위험

```
1000개 인덱스 × 5 shard × 1 replica = 10,000 shard
```

각 shard는:
- Lucene `IndexReader` 인스턴스
- File handle 수십~수백 개
- Heap에 segment 메타데이터, FST, BKD tree, doc values cache
- Cluster state 항목

> "Aim to keep the average shard size between a few GB and a few tens of GB. Avoid having a very large number of small shards."

너무 많은 shard는 cluster state 폭증 → master 압박 → 색인·검색 둔화.

### 증상

- "this action would add ... too many shards" warning
- Master node CPU 폭주, cluster state 갱신 지연
- 노드 재시작 후 복구 시간이 분 단위 → 시간 단위로 늘어남
- Heap의 상당 부분이 segment 메타데이터로 점유

### 가이드라인

- Heap GB당 shard 수 **20개 이하** 목표
- Primary shard 크기 **10–50 GB**
- 시계열은 일/주 단위 인덱스 + ILM rollover로 shard 수 통제
- 작은 인덱스가 많다면 → 같은 mapping이라면 **1 shard**로 통합 검토

```
PUT /my-index/_settings
{ "index.number_of_shards": 1 }   # 생성 시점에만 가능. 이후는 reindex 필요.
```

→ 상세: [C1 oversharding](../troubleshooting/C1_oversharding.md)

---

## 10.9 Replica로 색인 처리량 늘리려는 시도

### 오해

"Replica가 많으면 부하가 분산되니 색인도 빨라지지 않나?"

### 현실

Replica는 **검색 처리량**을 늘린다. 색인은 정확히 **반대 방향**으로 영향을 준다.

```
[primary indexing]
    ↓
[replica forwarding] ──→ 모든 replica가 동일 작업 반복
    ↓
[primary는 wait_for_active_shards 만족 시 응답 → in-sync replica로 복제 시도]
```

색인은 `wait_for_active_shards`(기본값 `1`, primary만 활성이면 진행)를 만족하면 응답을 반환하고, 이후 모든 in-sync replica로 복제가 시도된다. "quorum 대기"는 5.x 이전 동작이며 현재는 기본 `1`이다.

Replica 수가 늘면:
- 네트워크 트래픽 N배
- 디스크 쓰기 N배 (replica 노드들의 합산)
- in-sync 유지 비용 증가 → 색인 latency 증가

### 결론

색인 처리량 ≠ replica 수. 처리량이 부족하면:
1. **Primary shard 수**를 늘려라 (인덱스 생성 시)
2. 노드 추가
3. Bulk 페이로드 튜닝
4. `refresh_interval` 늘리기

검색 부하가 진짜 문제일 때만 replica 추가.

---

## 10.10 force_merge를 활성 인덱스에

9.8 반복. 핵심 증상:

- 디스크 사용량이 일시적으로 2배 → flood_stage watermark 도달 → 모든 인덱스 read-only 전환
- Merge가 끝없이 반복됨 (새 segment가 계속 생기므로)
- 거대 단일 segment가 생기면 삭제된 문서가 영원히 회수되지 않음
- I/O 폭주로 검색 latency 수십~수백 배

**원칙**: ILM warm phase에 자동으로 처리하라. 수동 `_forcemerge`는 read-only 전환된 인덱스에만.

---

## 10.11 Heap을 32GB 초과로 설정

### 위험

JVM은 **compressed ordinary object pointers (compressed oops)**를 사용해서 64비트 환경에서도 32비트 포인터를 쓴다. 이 한계는 약 32 GB heap이다 (정확히는 ~26-30GB, JVM 옵션에 따라).

```
heap 31GB → compressed oops → 모든 객체 포인터 4 byte
heap 33GB → compressed oops 비활성 → 포인터 8 byte
        → "heap을 키웠는데 사용 가능한 객체 수는 더 적어짐"
```

> "Set Xms and Xmx to no more than the threshold for compressed oops; the exact threshold varies but is near 32 GB."

### 운영 권장

- **Heap = total RAM의 50% 또는 31GB 중 작은 값**
- 나머지 50%는 OS file system cache (Lucene이 mmap)
- 64GB RAM 노드 → heap 31GB
- 128GB RAM 노드 → heap 31GB (남는 RAM은 모두 page cache)

```
# jvm.options
-Xms31g
-Xmx31g
```

`-XX:+PrintFlagsFinal | grep UseCompressedOops`로 활성 여부 확인.

→ 상세: [C3 JVM heap pressure](../troubleshooting/C3_jvm_heap_pressure.md)

---

## 10.12 Update 남발

### 위험

Lucene segment는 **immutable**이다. `_update` API는 내부적으로:

```
1. 기존 문서 read
2. 변경 적용
3. 기존 문서를 deleted로 마킹 (tombstone)
4. 새 버전을 새 segment에 색인
```

자주 update하는 워크로드는:
- Segment에 deleted document가 누적 → 디스크·메모리 낭비
- Merge가 더 자주 발생 → I/O 폭증
- `_id` 라우팅을 위한 GET 비용
- Optimistic concurrency control 충돌 (`version_conflict_engine_exception`)

### 증상

```
GET /_cat/indices?v&h=index,docs.count,docs.deleted
my-index   100000   850000  ← deleted가 active의 8배
```

### 대안

**Append-only 모델**:

```
이벤트 시점마다 새 문서 INSERT
{ "@timestamp": "...", "user_id": 1, "status": "ACTIVE" }
{ "@timestamp": "...", "user_id": 1, "status": "INACTIVE" }
```

조회는 `top_hits` agg + sort by timestamp desc로 최신값.

**bulk update 대신 reindex**:

대량 변경은 새 인덱스로 reindex하고 alias 스왑이 거의 항상 더 빠르고 안전하다.

```
POST /_reindex
{ "source": { "index": "old" }, "dest": { "index": "new" } }

POST /_aliases
{ "actions": [
  { "remove": { "index": "old", "alias": "current" } },
  { "add":    { "index": "new", "alias": "current" } }
]}
```

---

## 10.13 Key Takeaways

| 안티패턴 | 핵심 증상 | 대안 |
|----------|-----------|------|
| `dynamic: true` 방치 | mapping explosion, 파싱 실패 | `dynamic: strict` + 명시 매핑 |
| text/keyword 혼동 | sort/agg 실패, term miss | multi-field |
| nested 남발 | doc 수 폭증, 머지 비용 | flatten 또는 flattened |
| deep pagination | 코디네이터 OOM | search_after + PIT |
| leading wildcard | term 풀스캔 | wildcard 타입, 데이터 분리 |
| `_all` 의존 | 8.x에서 미존재 | copy_to + multi_match |
| 단건 INSERT | 처리량 1/100 | bulk 5-15 MiB |
| oversharding | cluster state 비대화 | shard 10-50 GB, heap당 20개 |
| replica로 색인 가속 | 색인 더 느려짐 | primary shard 수 ↑ |
| 활성 인덱스에 force_merge | 디스크 2배, 머지 폭주 | ILM warm 자동화 |
| heap > 32GB | compressed oops 비활성 | 31GB 상한 |
| Update 남발 | deleted doc 폭증 | append-only, reindex |

---

*다음 장: 11장. 모니터링과 트러블슈팅*
