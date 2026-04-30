# Elasticsearch 트러블슈팅 케이스

실무에서 자주 마주치는 12가지 사고 케이스. 각 문서는 **증상 → 원인 → 진단 쿼리 → 해결 → 예방** 흐름으로 구성된다.

대상 버전: Elasticsearch 8.x / 9.x

---

## 카테고리별 인덱스

### A. Mapping 실수

| # | 케이스 | 핵심 |
|---|--------|------|
| [A1](A1_mapping_explosion.md) | 매핑 폭증 (total_fields > 1000) | cluster state 비대 → 마스터 노드 부하. `flattened`/`runtime`으로 가변 키 격리 |
| [A2](A2_text_vs_keyword.md) | text/keyword 혼동 | text 정렬·집계 불가, fielddata 강제 ON 위험. 멀티필드(.keyword) 표준 패턴 |
| [A3](A3_dynamic_mapping_runaway.md) | dynamic mapping 무한 증식 | 페이로드 키가 데이터일 때. `dynamic: strict` + `flattened` 또는 key/value nested |

### B. Query 실수

| # | 케이스 | 핵심 |
|---|--------|------|
| [B1](B1_deep_pagination.md) | from+size > 10000 | `Result window is too large`. `search_after` + PIT으로 마이그레이션 |
| [B2](B2_wildcard_regex.md) | leading wildcard / 복잡 regex | CPU 폭증. `wildcard` 타입, edge_ngram, prefix 쿼리, `allow_expensive_queries` 차단 |
| [B3](B3_fielddata_on_text.md) | text 필드 fielddata → heap OOM | `fielddata: true`는 거의 항상 잘못된 답. 멀티필드(.keyword) 또는 runtime field |

### C. 운영 실수

| # | 케이스 | 핵심 |
|---|--------|------|
| [C1](C1_oversharding.md) | 샤드 수만 개 → 메타 폭증 | 샤드는 비싸다. ILM rollover + shrink로 권장 크기 (10~50GB) 맞추기 |
| [C2](C2_split_brain.md) | master quorum 잘못 설정 | dedicated master 3대 권장. `cluster.initial_master_nodes` 운영 중에 두지 말 것 |
| [C3](C3_jvm_heap_pressure.md) | heap 90%+, GC 폭증, circuit breaker | heap 30GB 상한, 멀티필드/composite agg/_source 슬리밍 |
| [C4](C4_disk_watermark.md) | 95% flood_stage → read-only 락 | ILM + searchable snapshot + 모니터링 3종. watermark 절대값 설정 권장 |

### D. 인덱싱 실수

| # | 케이스 | 핵심 |
|---|--------|------|
| [D1](D1_single_doc_indexing.md) | 단건 색인 → 처리량 저하 | Bulk API 5~15MB로 묶기. 표준 클라이언트의 BulkProcessor/streaming_bulk 활용 |
| [D2](D2_refresh_interval.md) | refresh_interval=1s 고집 → segment 폭증 | 워크로드별 refresh 차등화. 로그/메트릭은 30~60s |

---

## 우선순위 (장애 영향도)

```
즉시 다운 위험       : C2 (split brain), C3 (heap OOM), C4 (디스크 락)
서비스 일부 마비     : A1, B1, B3
점진적 성능 저하     : A2, A3, B2, C1, D1, D2
```

---

## 공통 진단 명령어

각 케이스에서 반복되는 핵심 진단 API:

```bash
# 클러스터 헬스
GET /_cluster/health
GET /_cat/health?v

# 노드 상태 (heap, disk, CPU)
GET /_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.percent
GET /_cat/allocation?v

# 인덱스 / 샤드
GET /_cat/indices?v&s=store.size:desc
GET /_cat/shards?v&s=store:desc
GET /_cat/segments?v&s=size:asc

# Thread pool
GET /_cat/thread_pool/write,search?v&h=node_name,name,active,queue,rejected

# JVM / GC
GET /_nodes/stats/jvm?human
GET /_nodes/stats/breaker

# 진행 중 task
GET /_tasks?detailed=true

# Hot threads
GET /_nodes/hot_threads
```

---

## 사고 대응 플로우

```
1. 알람 또는 사용자 신고 수신
       ↓
2. _cluster/health, _cat/nodes 로 1차 상태 파악
       ↓
3. 증상 매칭 (이 README의 카테고리)
       ↓
4. 해당 케이스의 "진단" 섹션 쿼리 실행
       ↓
5. "해결" 섹션의 단계 1 (즉각 완화) 우선 적용
       ↓
6. 복구 후 "해결" 단계 2~ (근본 해결) 적용
       ↓
7. "예방" 섹션 체크리스트로 재발 방지
```

운영 중인 시스템이면 단계 1 → 즉각 완화부터 적용해 사용자 영향을 줄이고, 단계 2 이후는 변경관리 절차 거쳐 적용 권장.
