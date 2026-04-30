# 1장. Elasticsearch 개요와 설계 철학

이 가이드는 Elasticsearch 8.x(LTS 8.17 baseline)와 9.x를 기준으로 한다. 이 장에서는 Elasticsearch가 어떤 시스템이며, 왜 그런 형태로 설계되었는지를 설명한다. 단순한 기능 나열이 아니라, 이후 모든 챕터에서 반복적으로 등장할 **설계 결정의 근거**를 먼저 정리한다.

---

## 1.1 Elasticsearch는 어떤 시스템인가

Elasticsearch는 Apache Lucene을 기반으로 하는 **분산 검색·분석 엔진(Distributed Search and Analytics Engine)**이다. 공식 문서의 정의는 다음과 같다.

> "Elasticsearch is a distributed, RESTful search and analytics engine, capable of addressing a growing number of use cases. As the heart of the Elastic Stack, it centrally stores your data for lightning fast search, fine-tuned relevancy, and powerful analytics that scale with ease."
> — [Elastic 공식 문서](https://www.elastic.co/elasticsearch)

이 한 문장에 Elasticsearch의 정체성이 모두 들어 있다.

- **Distributed**: 처음부터 클러스터로 동작하도록 설계되었다. 단일 노드 모드는 개발용에 가까우며, 운영은 전제 조건이 다중 노드다.
- **RESTful**: 모든 조작이 HTTP + JSON으로 이루어진다. 드라이버 종속성이 없고, `curl` 한 줄로 시스템을 조작·진단할 수 있다.
- **Search and analytics**: 풀텍스트 검색뿐만 아니라 집계(aggregation), 시계열 분석, 벡터 검색까지 한 엔진에서 다룬다.
- **Lucene-based**: 실제 인덱스 자료구조(역인덱스, 세그먼트, 머지)는 Lucene이 담당한다. Elasticsearch는 Lucene 인덱스를 **분산·복제·라우팅**하는 레이어다.

이 마지막 항목이 핵심이다. **Elasticsearch ≈ Distributed Lucene + REST API + Cluster Management**라고 이해해도 큰 무리가 없다. 따라서 이 가이드는 3장에서 Lucene 내부를, 그 이후로는 Elasticsearch가 추가하는 분산 레이어를 다룬다.

> 9.x 시점부터 Elasticsearch는 **Lucene 10** 기반으로 동작한다. 8.x는 Lucene 9 계열이다. SIMD 최적화, 인덱싱 병렬성, BBQ(Better Binary Quantization) GA 등 9.x의 성능 향상은 대부분 Lucene 10에서 비롯된다. ([Elastic Blog: 9.0 announcement](https://www.elastic.co/blog/whats-new-elastic-search-9-0-0))

---

## 1.2 검색 엔진 vs RDBMS

Elasticsearch를 처음 접할 때 가장 흔한 오해가 "MySQL 비슷한데 검색이 빠른 것"이라는 인식이다. 두 시스템은 **인덱스 자료구조 자체가 다르며**, 그로부터 거의 모든 운영 특성이 갈린다.

### B-Tree (RDBMS) vs Inverted Index (Elasticsearch)

RDBMS의 인덱스는 일반적으로 **B-Tree** 또는 그 변형이다. B-Tree는 하나의 컬럼 값을 키로 받아 그에 해당하는 행 위치를 반환한다. `WHERE name = 'Alice'`처럼 **정확한 값 매치**에는 최적이지만, "name 컬럼 안에 'Alice'라는 토큰이 포함된 모든 문서"를 찾는 풀텍스트 검색에는 부적합하다.

Elasticsearch(Lucene)는 **역인덱스(Inverted Index)**를 쓴다. 역인덱스는 "term → 그 term이 등장한 문서들의 리스트"를 저장한다.

### 역인덱스 구조

```
원본 문서:
  doc1: "the quick brown fox"
  doc2: "the lazy dog"
  doc3: "quick brown dog runs"

분석(analyze): 소문자화, 토크나이즈, 불용어 제거 등

역인덱스(Inverted Index):
  ┌──────────┬──────────────────────────────┐
  │   term   │        posting list          │
  ├──────────┼──────────────────────────────┤
  │  brown   │  doc1, doc3                  │
  │  dog     │  doc2, doc3                  │
  │  fox     │  doc1                        │
  │  lazy    │  doc2                        │
  │  quick   │  doc1, doc3                  │
  │  runs    │  doc3                        │
  │  the     │  doc1, doc2                  │
  └──────────┴──────────────────────────────┘
            ▲
            │ 정렬된 term dictionary (FST로 효율 압축)

쿼리: "quick AND brown"
  → posting list 교집합: {doc1, doc3} ∩ {doc1, doc3} = {doc1, doc3}
  → O(posting list 길이의 합), 정확 매치보다 훨씬 큰 데이터셋도 ms 단위 응답
```

이 구조에서 다음 특성이 자연스럽게 따라 나온다.

- **풀텍스트 검색이 본업**이다. `LIKE '%foo%'`가 풀스캔인 RDBMS와 달리, Elasticsearch는 토큰 단위 검색이 곧 인덱스 룩업이다.
- **세그먼트는 immutable**하다. B-Tree처럼 in-place 업데이트가 아니라, 변경 시 새 세그먼트를 만들고 기존 세그먼트는 그대로 두며 백그라운드 머지로 정리한다 (3장 참고).
- **관계형 JOIN은 약점**이다. 역인덱스는 단일 인덱스 안에서만 효율적이다. 9.x에서 ES|QL JOIN이 GA되었지만, RDBMS의 임의 JOIN을 대체할 수 있는 기능은 아니다.

### Near-Real-Time (NRT)

Elasticsearch는 **NRT 검색**을 표방한다. 색인 직후 즉시 검색 가능한 것이 아니라, 기본 1초의 `refresh_interval`이 지나야 새 문서가 검색에 노출된다.

```
INSERT (RDBMS):           t=0에 즉시 SELECT 가능 (커밋 후)
Index (Elasticsearch):    t=0에 색인, t≈1s 후 검색 가능
```

이는 단점이 아니라 **의도된 설계**다. 매 색인마다 디스크 fsync와 인덱스 갱신을 하지 않고, 1초 단위로 묶어 처리하기에 색인 처리량이 극적으로 높아진다. NRT의 메커니즘(인메모리 버퍼 → refresh로 segment 생성 → flush로 영속화 → translog로 내구성 보장)은 3장에서 자세히 다룬다.

---

## 1.3 사용 사례

Elasticsearch는 단순 검색 엔진을 넘어 다양한 워크로드를 흡수해 왔다.

### Full-Text Search

가장 전통적인 용도다. 상품 검색, 문서 검색, 사이트 내 검색 등. BM25 스코어링, 형태소 분석(`nori`, `kuromoji`), 자동완성(suggest API)이 주요 기능이다.

### Observability (Logs / Metrics / Traces)

ELK 스택의 "L"이 Logstash인 것에서 알 수 있듯, Elasticsearch는 로그 분석의 사실상 표준 중 하나다. Beats/Elastic Agent로 수집한 로그를 시간 기반 인덱스(예: `logs-app-2026.04.30`)에 색인하고, Kibana에서 시각화한다. 9.x에서는 OpenTelemetry 기반 EDOT가 GA되어 메트릭/트레이스도 통합 처리한다.

### Security Analytics (SIEM)

Elastic Security는 Elasticsearch를 백엔드로 하는 SIEM 제품이다. 보안 이벤트를 색인하고, 룰 기반 탐지(EQL), 행위 분석, 위협 헌팅을 수행한다.

### Vector Search (Semantic Search)

8.x부터 dense_vector 필드와 kNN 검색이 추가되었다. 9.x에서는 BBQ(Better Binary Quantization)가 GA되어 메모리 사용량을 32배 줄이면서 SIMD로 8~30배 빠른 처리량을 제공한다. RAG(Retrieval-Augmented Generation), 추천, 이미지 검색 등에 활용된다. ([Elastic 9.0 BBQ](https://www.elastic.co/blog/whats-new-elastic-search-9-0-0))

---

## 1.4 Elastic Stack 구성

Elasticsearch는 단독으로 쓰이지 않는다. 함께 도는 컴포넌트의 역할을 알아두면 운영 의사결정이 쉬워진다.

```
┌────────────────────────────────────────────────────────────────────┐
│                          Elastic Stack                              │
└────────────────────────────────────────────────────────────────────┘

   ┌─────────┐   ┌─────────┐   ┌────────────┐
   │ Beats   │   │Logstash │   │Elastic Agent│   ← 데이터 수집(Shippers)
   │(Filebeat│   │ (ETL)   │   │ + Fleet    │
   │ Metric.)│   │         │   │ (중앙 관리) │
   └────┬────┘   └────┬────┘   └─────┬──────┘
        │             │              │
        └─────────────┼──────────────┘
                      ▼
        ┌─────────────────────────────┐
        │      Elasticsearch          │   ← 저장 + 검색 + 분석
        │  (분산 인덱스 + 클러스터링)   │
        └──────────────┬──────────────┘
                       │
                       ▼
                ┌──────────────┐
                │    Kibana    │   ← 시각화 + 관리 UI
                └──────────────┘
```

각 컴포넌트:

| 컴포넌트 | 역할 |
|----------|------|
| **Elasticsearch** | 데이터 저장, 검색, 집계. 모든 것의 중심. |
| **Kibana** | 시각화, 대시보드, 클러스터 관리(Stack Management), Dev Tools |
| **Logstash** | 무거운 ETL. grok 파싱, 외부 enrichment, 다중 출력 |
| **Beats** | 경량 데이터 수집기 (Filebeat, Metricbeat, Packetbeat 등) |
| **Elastic Agent** | 통합 수집기. Beats를 대체하는 차세대 에이전트. Fleet으로 중앙 관리 |
| **Fleet** | Elastic Agent의 중앙 정책 관리 서버 (Kibana 내장) |

운영에서 자주 헷갈리는 지점은 **Beats와 Elastic Agent의 관계**다. Elastic Agent는 Beats(filebeat, metricbeat 등)를 단일 바이너리로 묶고 Fleet으로 정책을 푸시할 수 있게 만든 것이다. 신규 도입이라면 Elastic Agent + Fleet 조합이 표준이다.

---

## 1.5 분산 시스템 핵심 원칙

Elasticsearch가 단일 Lucene 인덱스 위에 추가하는 분산 레이어의 핵심 개념을 정리한다. 자세한 내용은 2장(클러스터)과 4장(샤드)에서 다룬다.

### Shard: 데이터 분할 단위

하나의 **인덱스(index)**는 하나 이상의 **샤드(shard)**로 분할된다. 각 샤드는 독립적인 Lucene 인덱스이며, 클러스터 내 어느 노드에든 배치될 수 있다. 즉, **인덱스는 논리적 개념, 샤드는 물리적 단위**다.

```
Index "logs-2026.04.30" (논리)
   │
   ├── Shard 0  (Lucene 인덱스, Node A에 위치)
   ├── Shard 1  (Lucene 인덱스, Node B에 위치)
   ├── Shard 2  (Lucene 인덱스, Node C에 위치)
   └── Shard 3  (Lucene 인덱스, Node A에 위치)
```

문서가 어느 샤드로 갈지는 `shard_num = hash(_routing) % num_primary_shards` 공식으로 결정된다 (4장 상세).

### Replica: 복제본

각 primary shard는 0개 이상의 **replica shard**를 가질 수 있다. Replica는 두 가지 목적을 동시에 수행한다.

1. **고가용성**: primary가 있는 노드가 죽으면 replica가 promote되어 primary가 된다.
2. **읽기 처리량**: 검색 요청은 primary와 replica 어느 쪽에서도 처리할 수 있어, replica가 늘어나면 읽기 QPS가 선형 증가한다.

단, **primary와 replica는 같은 노드에 배치되지 않는다**. 노드 장애 시 둘 다 잃으면 의미가 없기 때문이다.

### Eventual Consistency in Search

Elasticsearch의 일관성 모델은 미묘하다.

- **쓰기 경로**: 색인 요청은 primary에 먼저 적용된 뒤, 모든 in-sync replica에 복제된다. 기본 `wait_for_active_shards=1` 설정에서 primary 확인만으로 응답이 돌아오지만, replica 복제까지 기다리도록 조정 가능하다.
- **읽기 경로**: 검색은 primary 또는 replica 중 하나로 라우팅된다. 따라서 방금 색인한 문서가 즉시 보이지 않을 수 있는 두 가지 이유가 있다.
  1. **Refresh 지연**: 색인 후 1초의 refresh_interval 전까지는 어떤 샤드에서도 검색되지 않는다.
  2. **Replication lag**: 매우 짧은 시간 동안 primary에는 반영됐지만 replica에는 아직 도달하지 않은 상태가 존재할 수 있다.

따라서 Elasticsearch는 **검색 경로에서 strong consistency를 보장하지 않는다**. 정확한 read-your-writes가 필요하면 `?refresh=wait_for` 또는 `?preference=_primary` 같은 옵션을 명시해야 한다.

---

## 1.6 9.x 버전 변경점 요약

> **Elasticsearch 9.x 주요 변경 (8.x 대비)**
>
> - **Lucene 10 기반**: 색인/검색 성능 전반 향상, SIMD 최적화 강화
> - **BBQ (Better Binary Quantization) GA**: 8.16에서 tech preview로 들어온 기능. 9.1부터 dense_vector 기본값. 메모리 32배 절감 + 8~30배 throughput
> - **ES|QL JOIN, LOOKUP, CCS GA**: 분산 환경에서 부분 결과(partial results)를 우선시하는 resilient 아키텍처
> - **Frozen tier 읽기 전용 인덱스의 read 동작 변경**: 9.x로 업그레이드 전, frozen 인덱스는 unfreeze 필요
> - **TLSv1.1 기본 비활성화**: 명시적 활성화 필요
> - **Enterprise Search 제거**: 9.0 이상에서 미지원
> - **업그레이드 경로**: 8.x → 9.x는 **반드시 8.19 경유**. Upgrade Assistant로 호환성 점검 필수
>
> 출처: [Elastic 9.0 release notes](https://www.elastic.co/guide/en/elastic-stack/9.0/release-notes-elasticsearch-9.0.0.html), [breaking changes](https://www.elastic.co/docs/release-notes/elasticsearch/breaking-changes)
>
> **이 가이드의 baseline은 8.17 LTS**다. 9.x 고유 변경점은 해당 챕터에서 박스로 별도 표기한다.

---

## 1.7 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **정체성** | Lucene 기반 분산 검색·분석 엔진. RESTful + JSON. |
| **인덱스 자료구조** | 역인덱스 + 컬럼형 doc values. B-Tree와 본질이 다르다. |
| **NRT** | refresh_interval(기본 1초) 후 검색 가능. 색인 효율을 위한 의도된 지연. |
| **분산 모델** | index = 논리, shard = 물리(Lucene 인덱스). replica = 복제본. |
| **일관성** | 검색 경로는 eventual. read-your-writes는 명시적 옵션 필요. |
| **사용 사례** | Full-text, Observability, SIEM, Vector Search |
| **버전 baseline** | 이 가이드는 8.17 LTS. 9.x 변경점은 박스로 별기. |

---

## 1.8 운영 시 주의점 (미리보기)

설계 철학에서 바로 도출되는 운영 원칙들. 이후 챕터에서 각각 자세히 다룬다.

- **샤드 수는 한번 정하면 바꾸기 어렵다**. 인덱스 생성 후 primary shard 수 변경은 reindex나 split/shrink로만 가능하다 (4장).
- **너무 많은 샤드는 클러스터를 죽인다**. cluster state 크기, 마스터 부하, 메모리 압박이 함께 증가한다 (oversharding, 4장).
- **bulk API로 배치 색인하라**. 단일 문서 색인은 RESTful이긴 하나 처리량이 매우 낮다.
- **마스터 전용 노드 3대를 분리하라** (운영 환경). 데이터 노드 부하가 마스터에 영향을 주면 클러스터가 통째로 흔들린다 (2장).
- **버전 업그레이드는 N → N+1 단위**. 8.x → 9.x는 8.19 경유 필수.

---

*다음 장: 2장. 클러스터 아키텍처 — Node 역할, Master election, Discovery, Coordinating의 동작*
