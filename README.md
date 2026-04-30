# Elasticsearch 아키텍처 & 운영 학습 가이드

> 공식 문서 (elastic.co/docs) 기반 학습 자료
> 개념 이해 → 실무 예제 → 빠른 참조 3단 구성
> 검증 버전: Elasticsearch **8.x / 9.x** (LTS 8.17 기준, 일부 9.x 변경 사항 표기)

---

## 폴더 구조

```
elasticSearch-guide/
├── chapters/        ← 개념 학습 (1~11장)
├── examples/        ← 실무 도메인 예제 (검색, 로그, 시계열 등)
├── troubleshooting/ ← 실수 케이스 스터디 (12개 케이스)
└── cheatsheets/     ← 빠른 참조 (Query DSL, Mapping, ILM 등)
```

---

## 추천 학습 경로

### 🟢 초심자: "Elasticsearch가 뭔지 감 잡기"

```
chapters/ch01_elasticsearch_overview.md   (왜 쓰는가, 검색 엔진 vs DB)
  ↓
chapters/ch03_lucene_internals.md         (Inverted Index = 검색의 기본)
  ↓
chapters/ch05_mapping_and_field_types.md  (필드 타입과 매핑)
  ↓
examples/01_product_search.md             (실제 예제로 이해)
```

### 🟡 실무자: "검색/색인 파이프라인 만들기"

```
chapters/ch06_text_analysis.md            (Analyzer, Tokenizer, Filter)
  ↓
chapters/ch07_query_dsl.md                (Query DSL 문법과 패턴)
  ↓
chapters/ch09_indexing_strategy.md        (Bulk, Refresh, Flush)
  ↓
examples/02_observability_logs.md         (ILM + Data Stream)
```

### 🟠 고급: "운영, 장애 대응과 최적화"

```
chapters/ch04_index_and_shards.md
  ↓
chapters/ch10_anti_patterns.md
  ↓
chapters/ch11_monitoring_troubleshooting.md
  ↓
cheatsheets/troubleshooting_apis.md
```

---

## 📚 chapters — 개념 학습

### 1부: 아키텍처 기초

- [**1장. Elasticsearch 개요와 설계 철학**](chapters/ch01_elasticsearch_overview.md)
  - 검색 엔진 vs RDBMS, Lucene 기반 분산 시스템, 사용 케이스
  - **시각화: 역인덱스 vs B-Tree, ELK 스택 구성도**
- [**2장. 클러스터 아키텍처**](chapters/ch02_cluster_architecture.md)
  - Node 역할 (Master, Data, Ingest, Coordinating, ML)
  - Discovery, Cluster State, Master 선출
- [**3장. Lucene 내부 구조**](chapters/ch03_lucene_internals.md)
  - Segment, Inverted Index, Doc Values, Stored Fields
  - **시각화: Refresh / Flush / Merge 라이프사이클**

### 2부: 인덱스 & 데이터 모델

- [**4장. Index, Shard, Replica**](chapters/ch04_index_and_shards.md) — 분산의 단위
  - Primary/Replica, Shard 라우팅, Allocation
  - **시각화: 잘못된 샤드 사이징이 일으키는 문제**
- [**5장. Mapping과 Field Types**](chapters/ch05_mapping_and_field_types.md)
  - text vs keyword, Dynamic mapping, Multi-fields, Runtime fields
  - **시각화: text/keyword 잘못 쓴 결과**
- [**6장. Text Analysis (Analyzer)**](chapters/ch06_text_analysis.md)
  - Char Filter → Tokenizer → Token Filter
  - Standard, Whitespace, Korean (Nori) Analyzer

### 3부: 검색과 집계

- [**7장. Query DSL**](chapters/ch07_query_dsl.md)
  - bool, match, term, range, function_score
  - **시각화: Query Context vs Filter Context, 점수 계산**
- [**8장. Aggregations**](chapters/ch08_aggregations.md)
  - Metric, Bucket, Pipeline 집계
  - Composite, Date histogram, Cardinality

### 4부: 운영 실무

- [**9장. Indexing 전략**](chapters/ch09_indexing_strategy.md)
  - Bulk API, Refresh interval, Translog, ILM, Data Stream
- [**10장. 안티패턴**](chapters/ch10_anti_patterns.md) — 피해야 할 패턴
- [**11장. 모니터링과 트러블슈팅**](chapters/ch11_monitoring_troubleshooting.md)
  - Hot threads, Pending tasks, Cluster health, JVM heap

---

## 🛠️ examples — 실무 도메인 예제

| # | 도메인 | 핵심 개념 |
|---|--------|---------|
| [01](examples/01_product_search.md) | 🛒 **E-commerce 상품 검색** | Multi-field, Nori, 동의어, 부스팅 |
| [02](examples/02_observability_logs.md) | 📊 **로그/Observability** | Data Stream, ILM, Hot-Warm-Cold |
| [03](examples/03_metrics_timeseries.md) | 📈 **시계열 메트릭** | TSDS, downsampling, runtime field |
| [04](examples/04_security_siem.md) | 🛡️ **보안/SIEM** | EQL, threat hunting, alerting |
| [05](examples/05_vector_search.md) | 🔎 **시맨틱/벡터 검색** | dense_vector, kNN, hybrid retrieval |

---

## 🔥 troubleshooting — 케이스 스터디

### A. 매핑 실수
| 케이스 | 핵심 증상 |
|--------|---------|
| [A1. Mapping explosion](troubleshooting/A1_mapping_explosion.md) | total_fields_limit, 메모리 폭주 |
| [A2. text/keyword 혼동](troubleshooting/A2_text_vs_keyword.md) | 정확 매칭 실패, 정렬/집계 불가 |
| [A3. Dynamic mapping 오용](troubleshooting/A3_dynamic_mapping_runaway.md) | 운영 중 매핑 무한 증식 |

### B. 검색/쿼리 실수
| 케이스 | 핵심 증상 |
|--------|---------|
| [B1. Deep pagination](troubleshooting/B1_deep_pagination.md) | from+size > 10000, OOM |
| [B2. Wildcard/Regex](troubleshooting/B2_wildcard_regex.md) | 쿼리 수십 초, CPU 폭증 |
| [B3. Fielddata on text](troubleshooting/B3_fielddata_on_text.md) | Heap OOM, 노드 다운 |

### C. 운영 실수
| 케이스 | 핵심 증상 |
|--------|---------|
| [C1. Oversharding](troubleshooting/C1_oversharding.md) | 클러스터 상태 빨강, 메타데이터 폭증 |
| [C2. Split brain / 마스터 선출](troubleshooting/C2_split_brain.md) | 데이터 분기, 인덱스 손상 |
| [C3. JVM Heap 압박](troubleshooting/C3_jvm_heap_pressure.md) | GC 폭증, circuit breaker |
| [C4. Disk watermark](troubleshooting/C4_disk_watermark.md) | read-only 인덱스, 쓰기 중단 |

### D. 인덱싱 실수
| 케이스 | 핵심 증상 |
|--------|---------|
| [D1. 단건 INSERT](troubleshooting/D1_single_doc_indexing.md) | 처리량 저하, refresh 폭주 |
| [D2. Refresh interval 1s 고집](troubleshooting/D2_refresh_interval.md) | 인덱싱 처리량 50% 손실 |

---

## ⚡ cheatsheets — 빠른 참조

| 파일 | 내용 |
|------|------|
| [query_dsl_quickref.md](cheatsheets/query_dsl_quickref.md) | Query DSL 자주 쓰는 패턴 모음 |
| [mapping_field_types.md](cheatsheets/mapping_field_types.md) | 필드 타입 선택 플로우차트 |
| [analyzer_selection.md](cheatsheets/analyzer_selection.md) | Analyzer 선택, 한국어 처리 |
| [ilm_policy_templates.md](cheatsheets/ilm_policy_templates.md) | ILM 정책 템플릿 (로그/메트릭) |
| [troubleshooting_apis.md](cheatsheets/troubleshooting_apis.md) | _cat / _cluster / _nodes API 모음 |
| [version_history.md](cheatsheets/version_history.md) | 버전별 주요 변경사항 (7.x ~ 9.x) |

---

## 핵심 원칙 요약

### 설계 원칙

```
1. text와 keyword를 구분하라
   → 검색은 text, 정확매칭/정렬/집계는 keyword
   → 둘 다 필요하면 multi-field 사용

2. Dynamic mapping을 운영에서 풀어두지 마라
   → "strict" 또는 "false"로 명시적 매핑 사용
   → 매핑은 한번 정해지면 변경 불가 (re-index 필요)

3. 샤드는 적게, 크게 (10~50GB 권장)
   → 작은 샤드 수만 개 = oversharding = 메타데이터 폭주
   → 너무 큰 샤드 = recovery 느림, hot spot

4. Replica는 가용성과 검색 처리량을 위한 것
   → 색인 처리량을 위한 것이 아님

5. ILM/Data Stream으로 인덱스 lifecycle 관리
   → 시계열은 무조건 Data Stream
```

### 색인 원칙

```
1. Bulk API로 배치 색인 (5~15MB 페이로드)
   → 단건 색인은 처리량의 적
2. Refresh interval은 워크로드에 맞게
   → 실시간 검색이 필요 없으면 30s~60s
   → 대용량 적재 시 -1 (수동) 후 일괄 refresh
3. Translog durability
   → 장애 허용도에 따라 request / async 선택
```

### 쿼리 원칙

```
1. Filter context를 적극 활용
   → 점수 계산 불필요한 조건은 filter (캐시됨)
2. Deep pagination 금지
   → search_after 또는 PIT(Point In Time) 사용
3. Wildcard, regex는 prefix/suffix만
   → leading wildcard("*foo")는 사실상 풀스캔
4. text 필드에 fielddata 켜지 마라
   → 정렬/집계가 필요하면 keyword 멀티필드 추가
5. 큰 결과는 _source 필요한 것만 (`_source` filtering)
```

---

## 참고 자료

- **공식 Get Started**: [elastic.co/docs/get-started](https://www.elastic.co/docs/get-started)
- **Elasticsearch Guide**: [elastic.co/guide/en/elasticsearch/reference/current](https://www.elastic.co/guide/en/elasticsearch/reference/current)
- **Apache Lucene**: [lucene.apache.org](https://lucene.apache.org/)
- **Elastic Blog (검색 관련 베스트 프랙티스)**: [elastic.co/blog](https://www.elastic.co/blog)
- **한국어 분석기 (Nori)**: [analysis-nori](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori.html)
