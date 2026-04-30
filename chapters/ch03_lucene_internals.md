# 3장. Lucene 내부 구조

Elasticsearch의 모든 색인·검색은 결국 **Apache Lucene**이 처리한다. 분산·복제 레이어를 제거하면 Elasticsearch는 "Lucene 인덱스를 여러 노드에 흩뿌리고, REST로 노출하는 것"이라고 단순화할 수 있다. 따라서 Lucene 내부를 이해하지 못하면 refresh 지연, 검색 속도 저하, 디스크 사용량 폭증 같은 현상의 근본 원인을 짚을 수 없다.

이 장은 **Elasticsearch 8.x = Lucene 9, 9.x = Lucene 10** 기준으로 한다.

---

## 3.1 Lucene이란

Lucene은 1999년 Doug Cutting이 시작한 **자바 기반 풀텍스트 검색 라이브러리**다. 데이터베이스가 아니다. 분산도, 복제도, REST도 제공하지 않는다. 그저 "디렉터리 한 개에 인덱스 자료구조를 만들고, 그 위에서 검색하는 라이브러리"다.

Lucene의 두 가지 본질적 특성:

1. **세그먼트(segment) 기반**: 인덱스는 여러 개의 세그먼트로 구성된다. 각 세그먼트는 자체 완결적인 미니 인덱스다.
2. **세그먼트는 immutable**: 한 번 디스크에 쓰인 세그먼트는 절대 수정되지 않는다. 변경/삭제는 새 세그먼트 추가 + 기존 세그먼트의 deleted bitset 마킹으로 처리된다.

이 두 성질에서 Elasticsearch의 모든 운영 특성이 나온다 — refresh, flush, merge, NRT, 그리고 색인 처리량의 한계까지.

```
한 Elasticsearch shard = 한 Lucene index = 여러 segment의 모음

  shard "logs-2026.04.30 / 0"
   ┌───────────────────────────────────────────────┐
   │  segment_0   segment_1   segment_2  ...        │
   │  ┌──────┐   ┌──────┐    ┌──────┐               │
   │  │ inv. │   │ inv. │    │ inv. │               │
   │  │ idx  │   │ idx  │    │ idx  │  ← 각 segment │
   │  │ docs │   │ docs │    │ docs │    내부 자료  │
   │  │ stor.│   │ stor.│    │ stor.│    구조       │
   │  └──────┘   └──────┘    └──────┘               │
   │                                                 │
   │   + commit point  (어떤 segment들이 활성인지)    │
   │   + deleted docs bitset (삭제 마킹)             │
   └───────────────────────────────────────────────┘
```

---

## 3.2 Inverted Index 구조

Lucene 검색의 핵심 자료구조. 1장에서 개념적으로 다뤘지만, 실제로 디스크에 어떻게 저장되는지 본다.

### 세 가지 컴포넌트

```
1. Term Dictionary (어휘 사전)
   "apple" → posting list 위치, frequency 정보
   "banana" → ...
   "cherry" → ...
   ▲
   │ 정렬됨, FST(Finite State Transducer)로 압축

2. Posting List (문서 목록)
   각 term이 등장하는 doc ID의 리스트
   "apple" → [doc1, doc5, doc12, doc88, ...]
   ▲
   │ Delta encoding + bit packing으로 압축
   │ (delta: doc IDs 차이만 저장 → 작은 숫자 → 더 압축됨)

3. Term Frequencies + Positions (스코어링용)
   각 term이 각 문서에서 몇 번, 어느 위치에 나타났는지
   ▲
   │ BM25, phrase query에 필요
```

### FST (Finite State Transducer)

Term Dictionary는 **수백만 개의 term**을 담을 수 있다. 이를 메모리에 효율적으로 올리기 위해 Lucene은 FST를 사용한다.

```
간단한 예: terms = ["cat", "car", "card", "care"]

       [c]
        │
       [a]
        │
       [r] ─── [t]      → "cat"
        │       (출력)
        │
       [r] ─── (출력: "car")
        │
        ├── [d] (출력: "card")
        │
        └── [e] (출력: "care")
```

FST는 **공통 prefix와 suffix를 공유**하므로 trie보다 작고, 문자열 → 값 매핑을 O(prefix 길이)로 조회할 수 있다. Lucene은 이 FST를 메모리에 유지하여 term 룩업을 디스크 접근 없이 처리한다.

### 검색 흐름 예시

쿼리: `"the quick brown fox"` 의 BM25 스코어링

```
① Analyzer가 쿼리를 토큰화: [quick, brown, fox]   (the는 stopword)
② 각 term의 posting list 조회 (FST 활용):
     quick → [doc1:tf=1:pos=2, doc7:tf=2:pos=5,12, ...]
     brown → [doc1:tf=1:pos=3, doc15:tf=1:pos=0]
     fox   → [doc1:tf=1:pos=4]
③ 모든 term을 포함하는 문서 찾기 (intersection):
     {doc1, doc7, doc15} ∩ {doc1, doc15} ∩ {doc1} = {doc1}
④ 각 문서에 BM25 점수 계산
⑤ top-K 반환
```

posting list가 압축된 채로 디스크에 있어도, intersection은 정렬된 순서로 한 번에 스캔하면서(SkipList) 처리된다. 따라서 검색 속도는 **가장 작은 posting list의 길이**에 거의 비례한다.

---

## 3.3 Doc Values: 컬럼 지향 보조 인덱스

역인덱스는 "term → docs"는 빠르지만 **"doc → field 값"**은 못 한다. 정렬, 집계, 스크립트는 후자를 필요로 한다.

이를 위해 Lucene은 **Doc Values**라는 별도 자료구조를 만든다.

### Inverted Index vs Doc Values

```
Inverted Index (검색용):
  term "apple" → [doc1, doc5, doc12]
  term "banana" → [doc3, doc7]

Doc Values (정렬/집계용 — 컬럼 지향):
  field "price":
    doc1 → 100
    doc2 → 250
    doc3 → 80
    doc4 → 320
    ...
  field "category":
    doc1 → "electronics"
    doc2 → "books"
    ...
```

Doc Values는 **컬럼 단위로 디스크에 저장**된다. ClickHouse의 컬럼형 저장과 같은 원리다. 한 필드의 값들이 연속 배치되어 있어 정렬/집계 시 sequential read로 처리된다.

### 메모리 모델

Doc Values는 **off-heap 메모리(file system cache)**에 올라간다. JVM heap이 아니다. 이것이 Elasticsearch가 heap을 시스템 메모리의 50% 이하(최대 32GB)로 권장하는 이유다 — 나머지는 OS가 file system cache로 사용해 doc values, posting list 등을 메모리에 올린다.

### 끄는 경우

Doc Values는 기본 활성이고 모든 필드에 자동 생성된다. 다음 경우에만 끈다.

```json
PUT my-index
{
  "mappings": {
    "properties": {
      "rarely_aggregated_field": {
        "type": "keyword",
        "doc_values": false   // ← 정렬/집계 안 할 거면 디스크 절약
      }
    }
  }
}
```

`text` 타입은 기본적으로 doc_values가 없다(대신 fielddata 사용). 집계가 필요한 텍스트는 `keyword`로 매핑하거나 multi-field로 둘 다 둔다.

---

## 3.4 Stored Fields, _source, _id

문서의 원본 JSON은 어디에 저장되는가?

### `_source`

기본적으로 Elasticsearch는 색인 시 받은 **원본 JSON 전체를 `_source`라는 stored field에 저장**한다. 검색 결과로 문서 내용을 반환하거나, reindex/update 시 사용된다.

```bash
GET my-index/_doc/1
{
  "_index": "my-index",
  "_id": "1",
  "_source": {                    ← 원본 JSON
    "title": "Hello",
    "body": "World"
  }
}
```

### Stored Fields

Stored fields는 **문서 단위로 저장되는 필드**다. _source가 그 대표적인 예다. 추가로 개별 필드도 `"store": true`로 stored field로 저장 가능하지만, _source가 있으면 보통 불필요하다.

Stored fields는 **압축된 청크 단위**로 저장된다 (LZ4 또는 ZSTD). 검색 결과 fetch 단계에서 이 청크가 디스크에서 읽힌다.

### `_source` 끄기 — 위험한 최적화

저장 공간을 줄이려고 `_source`를 끌 수 있다.

```json
PUT my-index
{
  "mappings": {
    "_source": { "enabled": false }
  }
}
```

하지만 이렇게 하면:

- **reindex 불가** (원본이 없음)
- **update API 불가**
- **partial update, upsert 불가**
- **highlighter가 일부 기능 못 함**

권장하지 않는다. 대신 `_source.excludes`로 큰 필드만 제외하거나, **Synthetic _source**(8.4+에서 시계열 인덱스에 도입됨)를 사용하라. Synthetic _source는 doc values에서 필요할 때 _source를 재구성하므로 디스크 절약 + 기능 유지.

### `_id`

문서 식별자. 명시하지 않으면 Elasticsearch가 자동 생성한다(Base64 UUID).

`_id`는 hash 함수의 입력이 되어 **샤드 라우팅**을 결정한다 (4장).

> **운영 팁**: `_id`를 직접 지정하면 같은 ID로 색인 시 **upsert**가 일어난다(기존 문서 삭제 마킹 + 새 버전 색인). 이는 더 비싸다. ID 자동 생성이 가능하면 그쪽이 빠르다.

---

## 3.5 Refresh / Flush / Merge 라이프사이클

이 절이 이 장의 핵심이다. 색인된 문서가 검색 가능해지고, 디스크에 영속화되고, 정리되는 전체 흐름.

### 전체 그림

```
   ┌──────────────────────────────────────────────────────────────┐
   │                  In-memory Indexing Buffer                    │
   │  새 문서가 도착하면 일단 여기에 쌓임                              │
   │  + Translog에 fsync (durability)                               │
   └───────────────────────┬──────────────────────────────────────┘
                           │
                           │  refresh (default: 1s 간격)
                           │
                           ▼
   ┌──────────────────────────────────────────────────────────────┐
   │              In-memory Segment (filesystem cache)             │
   │  새 segment가 만들어짐 (아직 fsync 안 됨)                        │
   │  검색 가능해짐 (NRT)                                            │
   └───────────────────────┬──────────────────────────────────────┘
                           │
                           │  flush (default: translog 512MB or 30분)
                           │
                           ▼
   ┌──────────────────────────────────────────────────────────────┐
   │                  Persisted Segments on Disk                   │
   │  fsync 완료. translog는 새로 시작.                              │
   │  Lucene commit point 갱신                                      │
   └───────────────────────┬──────────────────────────────────────┘
                           │
                           │  merge (백그라운드, 지속적)
                           │
                           ▼
   ┌──────────────────────────────────────────────────────────────┐
   │                Larger Merged Segments                          │
   │  10개 작은 segment → 1개 큰 segment                             │
   │  deleted docs 진짜로 제거됨                                      │
   └──────────────────────────────────────────────────────────────┘
```

### Refresh: NRT를 만드는 메커니즘

색인 요청이 들어오면 문서는 **인메모리 buffer**에 쌓인다. 이 시점에는 검색 불가. 1초마다(기본값) **refresh**가 일어난다.

Refresh가 하는 일:

1. 인메모리 buffer의 내용으로 **새 Lucene segment를 생성** (단, 메모리에)
2. 이 segment를 **filesystem cache에 쓴다** (OS 메모리, 아직 fsync 안 됨)
3. **새 IndexReader를 연다** → 검색에 새 segment가 노출됨
4. 인메모리 buffer 비움

```bash
# refresh_interval 조정
PUT my-index/_settings
{
  "index": { "refresh_interval": "30s" }    # 색인 처리량 우선
}

# 강제 refresh (테스트용)
POST my-index/_refresh
```

**중요**: Refresh는 **검색 가시성**을 만들 뿐, durability를 보장하지 않는다. fsync 안 된 segment는 OS 크래시 시 사라질 수 있다. durability는 translog가 담당한다.

### Translog: Durability 보장

Refresh가 1초 간격이라면, 그 사이에 색인된 문서는 어떻게 안 잃어버릴까? 답은 **translog**.

```
색인 요청
   │
   ▼
①  In-memory buffer에 추가
②  Translog에 append + fsync   ← 이 시점에 디스크 도달
③  Client에 ack (기본 index.translog.durability=request)
```

색인 응답이 오면 데이터는 이미 **translog에 fsync된 상태**다. 노드가 죽어도 재시작 시 translog를 replay하여 in-memory buffer를 복원한다.

#### Translog 설정

```yaml
# 두 가지 모드
index.translog.durability: request   # 기본. 매 요청마다 fsync (안전, 느림)
index.translog.durability: async     # 5초마다 fsync (빠름, 최대 5초 데이터 유실 가능)

index.translog.flush_threshold_size: 512mb   # 이 크기 도달 시 flush 트리거
index.translog.sync_interval: 5s             # async 모드에서의 fsync 주기
```

**대량 로그 적재**처럼 일부 데이터 유실이 허용되는 경우 `async`로 처리량 2~3배 증가 가능. 결제 같은 케이스는 절대 `request` 유지.

### Flush: Lucene Commit

Flush는 **현재까지의 모든 인메모리 segment를 fsync하고 새 commit point를 기록**하는 과정이다.

```
Flush가 하는 일:
  1. 인메모리/캐시에 있는 segment들을 fsync (디스크 영속화)
  2. 새 Lucene commit point 작성 (어떤 segment들이 활성인지 기록)
  3. 기존 translog 비우고 새 translog 시작
```

Flush 트리거:
- translog 크기가 `flush_threshold_size`(기본 512MB) 도달
- 30분 경과 (`index.translog.flush_threshold_period`)
- 명시적 `POST /_flush`

flush 후 translog는 비워지므로, **재시작 시 replay할 양이 줄어든다**. 즉, flush는 "복구 시간 단축"이 가장 큰 목적.

### Merge: TieredMergePolicy

Refresh가 1초마다 일어나면 segment가 빠르게 누적된다. segment가 많을수록:

- 검색 시 모든 segment를 순회해야 함 → 느려짐
- 파일 디스크립터 소비
- 메타데이터 메모리 부담

Lucene은 **백그라운드에서 작은 segment들을 큰 segment로 병합**한다. 기본 정책은 **TieredMergePolicy**.

```
TieredMergePolicy 동작:

Tier 0 (작은 segment들):
  [seg_a, seg_b, seg_c, seg_d, seg_e, seg_f, seg_g, seg_h, seg_i, seg_j]
   각 ~5MB                     ↓ 10개 모이면 머지
  [───────── seg_t1_x (50MB) ─────────]

Tier 1:
  [seg_t1_a, seg_t1_b, ..., seg_t1_j]    ← 이것도 10개 모이면 머지
   각 ~50MB                    ↓
  [────── seg_t2_x (500MB) ──────]

...
이렇게 지수적으로 큰 tier 형성
```

핵심 설정:

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `index.merge.policy.segments_per_tier` | 10 | tier당 segment 수. 10개 모이면 머지 |
| `index.merge.policy.max_merge_at_once` | 10 | 한 번에 머지할 segment 최대 |
| `index.merge.policy.max_merged_segment` | 5gb | 머지 결과 최대 크기. 이걸 넘는 머지는 하지 않음 |
| `index.merge.policy.floor_segment` | 2mb | 이보다 작은 segment는 모두 같은 크기로 취급 |

`max_merged_segment=5gb`가 핵심이다. 일반 머지는 5GB까지만 자동 진행되고, 그보다 큰 segment는 만들어지지 않는다. 더 큰 segment를 원하면 **Force Merge**를 명시적으로 호출해야 한다.

### Force Merge

색인이 끝난 인덱스(write가 더 없는 인덱스 — 시계열의 어제 데이터 등)에 한해, 강제로 segment 수를 줄여 검색 속도를 올린다.

```bash
# segment 1개로 강제 머지 (write가 끝난 인덱스에만!)
POST /logs-2026.04.29/_forcemerge?max_num_segments=1

# 주의: write가 진행 중인 인덱스에 force merge 하지 말 것
# - 거대한 segment(>5GB) 생성됨
# - 이후 새 문서가 들어와 deleted docs가 쌓이면 진짜 정리(merge)가 안 됨
# - 디스크 사용량 회수 불가
```

### Deleted Docs

세그먼트가 immutable이므로 **삭제는 즉시 일어나지 않는다**. `DELETE` 또는 같은 `_id`로 update 시:

```
1. 기존 segment의 doc_id에 대해 "deleted" 비트 마킹 (.liv 파일)
2. 새 버전(있으면)은 새 segment에 추가
3. 검색 시 deleted bit가 켜진 문서는 결과에서 제외 (필터링)
4. 머지 때 비로소 deleted docs가 진짜로 디스크에서 사라짐
```

따라서 **대량 삭제 후 즉시 디스크 공간 회수를 기대하면 안 된다**. 머지가 일어나야 회수된다.

---

## 3.6 색인 파이프라인 전체 흐름

```
[Client]
   │ POST /logs/_doc
   ▼
[Coordinating node]
   │ 샤드 라우팅: hash(_id) % num_primary_shards
   ▼
[Primary shard 노드]
   │
   │  ① in-memory indexing buffer에 문서 추가
   │     (역인덱스 자료구조 갱신, doc values 갱신)
   │
   │  ② translog에 append + fsync
   │     (durability 확보)
   │
   │  ③ replica로 복제 요청 전송
   │
   ├──────► [Replica shard 1]: 위 ①②와 동일
   ├──────► [Replica shard 2]: 위 ①②와 동일
   │
   │  ④ 모든 replica의 ack 받으면 client에 응답
   ▼
   ───── 응답 완료 ─────
   ─────  여기까지가 색인 응답  ─────
   ─────  아직 검색 불가, 아직 segment 아님 ─────


   백그라운드 진행:

   ⑤ Refresh (기본 1초마다)
      buffer → 새 in-memory segment → searcher 갱신
      → 이제 검색 가능!

   ⑥ Translog 누적, 512MB 또는 30분 경과 시 → Flush
      모든 segment fsync + commit point 갱신 + translog 초기화

   ⑦ Segment 누적 → Merge (TieredMergePolicy)
      작은 segment들 → 큰 segment로 합침
      deleted docs 진짜 제거
```

---

## 3.7 운영 시 주의점

### refresh_interval은 색인 처리량의 가장 큰 변수

```bash
# 대량 색인 시작 전: refresh 끄거나 길게
PUT logs-init/_settings
{ "index": { "refresh_interval": "-1" } }

# 대량 색인 종료 후: 복원
PUT logs-init/_settings
{ "index": { "refresh_interval": "1s" } }
POST logs-init/_refresh
POST logs-init/_forcemerge?max_num_segments=1
```

`refresh_interval=30s`로만 늘려도 색인 처리량이 2~5배 증가하는 경우가 흔하다. 검색 즉시성이 필요 없는 워크로드에는 우선 검토.

### Translog durability는 의식적으로 결정하라

기본 `request`는 안전하지만 IOPS 부하가 크다. 로그/메트릭처럼 일부 유실 허용 가능한 워크로드는 `async`.

### Force merge는 "쓰기 끝난" 인덱스에만

ILM의 warm phase에서 `forcemerge` 액션을 활용하라. 운영 중인 hot 인덱스에 수동 force merge는 금기.

### Heap 32GB 룰 (compressed oops)

JVM heap이 32GB를 넘으면 **compressed object pointer**가 비활성되며, 메모리 효율이 급락한다. **heap = min(시스템 메모리의 50%, 30GB)** 가 표준이다. 나머지 메모리는 OS file system cache로 doc values와 posting list를 캐싱하게 둔다.

```yaml
# jvm.options
-Xms30g
-Xmx30g
```

### Segment 수 모니터링

```bash
GET /_cat/segments/logs-2026.04.30?v=true&h=shard,segment,size,size.memory,docs.count,docs.deleted

# 샤드당 segment 수
GET /_cat/shards/logs-*?v=true&h=index,shard,prirep,docs,store,segments.count
```

세그먼트가 한 샤드에 100개를 넘어가면 머지가 따라잡지 못하는 신호다. 색인 throttling, refresh_interval, merge throttling을 점검하라.

---

## 3.8 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **Lucene segment** | Immutable. 변경은 새 segment 추가 + deleted bit |
| **Inverted Index** | Term dict (FST) + Posting List + Term frequencies/positions |
| **Doc Values** | 컬럼 지향 보조. 정렬/집계용. off-heap (filesystem cache) |
| **_source** | 원본 JSON. reindex/update의 전제 |
| **Refresh** | 1초마다. 검색 가시성. fsync 아님 |
| **Translog** | Durability. 매 요청 fsync(기본). 재시작 시 replay |
| **Flush** | Lucene commit + translog 초기화 |
| **Merge** | TieredMergePolicy. 백그라운드. 5GB가 자연 머지 상한 |
| **Force merge** | 쓰기 끝난 인덱스만. 운영 중 인덱스에 금기 |
| **Heap 32GB 룰** | compressed oops 한계. 시스템 메모리 50%, 최대 30GB |

---

*다음 장: 4장. Index, Shard, Replica — 라우팅 공식, 샤드 사이징, 할당 제어, oversharding 진단*
