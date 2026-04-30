# Elasticsearch 실무 예제

5개의 도메인별 실무 시나리오. 각 예제는 **요구사항 → 매핑 설계 → 색인 → 검색 쿼리 → 운영 팁** 흐름으로 구성된다.

대상 버전: Elasticsearch 8.x / 9.x

---

## 도메인별 한눈에 보기

| # | 파일 | 도메인 | 핵심 기술 |
|---|------|--------|----------|
| 1 | [01_product_search.md](01_product_search.md) | E-commerce 상품 검색 | Nori, multi-field, edge_ngram 자동완성, function_score, 패싯 |
| 2 | [02_observability_logs.md](02_observability_logs.md) | 로그 / Observability | ECS, Data Stream, ILM (hot-warm-cold), Filebeat, `match_only_text`, `flattened` |
| 3 | [03_metrics_timeseries.md](03_metrics_timeseries.md) | 시계열 메트릭 | TSDS (`index.mode: time_series`), dimension/metric, Downsampling, runtime field, `serial_diff` |
| 4 | [04_security_siem.md](04_security_siem.md) | 보안 / SIEM | ECS, EQL `sequence`, Threat Intel index, Detection Rule, MITRE ATT&CK |
| 5 | [05_vector_search.md](05_vector_search.md) | 시맨틱 / 벡터 검색 | `dense_vector`, HNSW, kNN, `retriever.rrf` (BM25+kNN), `semantic_text`, ELSER, E5 |

---

## 추천 학습 순서

```
처음 시작        → 01 (상품 검색): 매핑/분석기/쿼리 기본기
운영을 다룬다면  → 02 (로그) → 03 (메트릭): Data Stream + ILM 마스터
보안/SIEM        → 04: ECS + EQL 시퀀스 매칭
LLM/RAG          → 05: 벡터 검색 + Hybrid retriever
```

각 예제는 독립적으로 읽을 수 있도록 작성되었지만, 02 → 03 순서로 보면 ILM/Data Stream 개념이 자연스럽게 확장된다.

---

## 공통 컨벤션

- 모든 DSL은 8.x 기준으로 검증. 9.x 신규 기능(`semantic_text` GA, logsdb 등)은 별도 표시.
- `keyword`/`text`/`numeric` 등 매핑 결정 근거를 표로 정리.
- 안티패턴 섹션을 통해 **하지 말아야 할 것**도 함께 다룬다.
- 운영 체크리스트(API 호출)를 마지막에 모음.

문제 발생 시 [troubleshooting/](../troubleshooting/) 디렉터리의 케이스를 참고할 것.
