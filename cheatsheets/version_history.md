# 버전 히스토리: 7.0 → 9.x

LTS 8.17 baseline에서 운영하면서 알아야 할 주요 변경점. 마이너 버전별 상세는 공식 release notes 참조.

---

## 한눈 타임라인

```
7.0 ────── 7.10 ────── 7.17 LTS ──── 8.0 ──── 8.17 LTS ──── 9.0
2019.04   2020.11      2022.02      2022.02   2024.x         2025.x

핵심 변화:
mapping types     ELSER       security        retriever    Lucene 10
제거             도입         default ON      / semantic   semantic_text
                              vector search   text         GA
```

---

## 7.0 (2019.04): mapping types 제거

### 핵심 변경

- **인덱스당 단일 type만 허용** (`_doc`)
- 이전: `/myindex/user/_doc/1`, `/myindex/post/_doc/2` (한 인덱스에 user, post type)
- 이후: `/myindex/_doc/1` 만 허용

### 영향

- 같은 인덱스에 여러 type이 있던 6.x 코드는 reindex 또는 인덱스 분리 필요
- URL의 type은 모두 `_doc`로 표준화

```
# 6.x
PUT /myindex/user/1 { ... }

# 7.x+
PUT /myindex/_doc/1 { ... }
```

### 그 외

- High-level REST client 안정화
- `_all` 필드 deprecated (이미 6.0부터)
- Adaptive replica selection 기본 ON

---

## 7.10 (2020.11): 데이터 tier, EQL

- **Data tiers** (`data_hot`, `data_warm`, `data_cold`, `data_frozen`) 도입
- **Searchable snapshot** GA (frozen tier 기반)
- **EQL (Event Query Language)**: 보안 이벤트 분석용
- Async search GA
- Runtime fields 도입 (7.11)

---

## 7.17 LTS (2022.02): 7.x 최종 안정화

- **8.x로 가는 마지막 brige 버전**
- 7.x 클러스터를 운영하다가 8.x rolling upgrade 시 반드시 거치는 단계
- security 옵션이 무료 (Basic) 라이센스에 포함됨 (실제로는 7.13부터)
- ILM, Searchable snapshot, Frozen tier 모두 안정

---

## 8.0 (2022.02): 보안 기본 ON, vector search GA

### 핵심 변경

**1. Security default ON**

신규 설치 시:
- TLS 자동 활성화
- Built-in `elastic` 사용자 비밀번호 자동 생성
- HTTP/Transport 통신 모두 암호화
- `enrollment-token` 기반 노드/Kibana 등록

```
bin/elasticsearch
# 출력에 elastic 비밀번호와 enrollment token 표시
```

7.x에서 평문 HTTP를 쓰던 코드는 https + 인증 헤더로 전환 필요.

**2. kNN search GA**

```json
GET /my-index/_search
{
  "knn": { "field": "embedding", "query_vector": [...], "k": 10, "num_candidates": 100 }
}
```

`dense_vector` 타입 + HNSW 인덱싱.

**3. mapping types 완전 제거**

- `include_type_name` 파라미터 완전 제거
- URL `PUT /myindex/_doc/1`은 그대로 유효 (여기서 `_doc`은 endpoint segment이지 type 이름이 아님)
- `PUT /myindex/{type}/_doc/1` 같은 type-aware URL은 거부됨

**4. transient cluster settings deprecated**

`persistent`만 사용 권장.

**5. System indices 보호 강화**

`.security`, `.kibana` 등은 사용자 권한 분리.

### Breaking 영향

- **HTTP 클라이언트는 모두 https**
- 7.x → 8.x rolling upgrade는 7.17 경유 필수
- 일부 deprecated API 제거 (e.g. `_xpack` 일부)

---

## 8.x 마이너 진화

### 8.4–8.6: ELSER + 통합 검색

- **ELSER** (Elastic Learned Sparse EncodeR) 모델 도입
- 영어 sparse 임베딩, 별도 학습 없이 semantic search
- `text_expansion` query

### 8.8–8.11: Retriever, semantic_text

- **Retriever 추상**: query, kNN, RRF, reranker를 합성하는 새 검색 API

```json
{
  "retriever": {
    "rrf": {
      "retrievers": [
        { "standard": { "query": { ... } } },
        { "knn":      { ... } }
      ]
    }
  }
}
```

- **Synonyms API**: 동의어 셋을 인덱스 외부에서 관리, 런타임 변경

### 8.13–8.16: 성능 / 인프라

- ESQL (Elasticsearch Query Language) 강화 (SQL 유사 파이프 문법)
- Indexing pressure 정밀도 향상
- 로그 데이터 streamlining (`logsdb` 인덱스 모드 도입 8.17)
- **`semantic_text` 타입 도입 (8.15, tech preview)**: inference endpoint에 매핑된 자동 임베딩 필드. 사용자가 raw 텍스트만 색인하면 ES가 임베딩 생성·저장

### 8.17 LTS

- 8.x의 안정화 LTS 라인
- **logsdb index mode**: 로그 워크로드 최적화 (저장 50% 절감, 동일 성능)
- 9.x 진입 직전 안정화 버전

### 8.18 / 9.0

- **`semantic_text` GA**: 8.15 tech preview를 거쳐 8.18/9.0에서 정식 출시

---

## 9.0 (2025.x): Lucene 10, 모던화

### 핵심 변경

**1. Lucene 10 기반**

- Vector 검색 성능 향상
- Codec 업데이트
- **Lucene 9 인덱스는 read-only로 유지 가능, 신규 색인은 9 codec**
- 이전 메이저(7.x) 인덱스는 호환 안 됨

**2. `transient` settings 완전 제거**

- 8.x에서 deprecated였던 것이 9.0에서 제거

**3. semantic_text 정식 GA**

- inference endpoint 자동 관리
- ELSER + 외부 LLM provider (OpenAI, Cohere 등) 통합 흐름 표준화

**4. 일부 deprecated API 제거**

- `_xpack/*` 잔존 엔드포인트
- 구 mapping options
- Legacy join 일부

### Upgrade 경로

```
7.x → 7.17 (LTS) → 8.x → 8.17 (LTS) → 9.0
                                     ↑
                       반드시 8.17 경유한 rolling upgrade
```

7.x 인덱스를 직접 9.0에 가져갈 수 없다 → 8.x에서 reindex 후 진행.

---

## 사라진 / 바뀐 것 빠른 표

| 항목 | 마지막 버전 | 처방 |
|------|-------------|------|
| `_all` 필드 | 6.x | `copy_to` + multi_match |
| Mapping types | 7.x (제거 완료) | 인덱스 분리, `_doc` |
| `include_type_name` | 7.x | 제거됨 |
| 평문 HTTP 기본 | 7.x | 8.x부터 TLS 자동 |
| `transient` cluster settings | 8.x deprecated | `persistent`만 사용 |
| Scroll API | 권장 deprecated | search_after + PIT |
| `_xpack` 일부 | 7.x→8.x | `/_security`, `/_license` 등 새 경로 |
| TransportClient (Java) | 7.x | High-level REST → 새 Java API client |
| `string` 필드 | 5.0 | `text` / `keyword` |
| 5.x parent-child | 5.x | `join` 필드 |
| `index.mapper.dynamic` | 7.x deprecated | `mappings.dynamic` per index |

---

## 자바 클라이언트 변천

```
TransportClient (~7.x deprecated)
   ↓
High-Level REST Client (7.0–7.17, 8.x deprecated)
   ↓
Elasticsearch Java API Client (8.x+, 권장)
```

8.x부터 새 프로젝트는 **Elasticsearch Java API Client** (`co.elastic.clients`) 사용. fluent builder + typed.

```java
ElasticsearchClient client = new ElasticsearchClient(transport);
SearchResponse<MyDoc> r = client.search(s -> s
    .index("my-index")
    .query(q -> q.match(m -> m.field("title").query("elastic"))),
    MyDoc.class);
```

---

## Upgrade 체크리스트 (메이저 진행 시)

- [ ] 현재 버전이 직전 LTS인가? (7.17 또는 8.17)
- [ ] Snapshot 기반 백업 완료
- [ ] Deprecation API 응답 점검: `GET /_migration/deprecations`
- [ ] 모든 인덱스가 현재 메이저 codec인가? (`_cat/indices?h=index,creation.date.string,version.created`)
- [ ] 클라이언트 라이브러리 호환 버전 확인
- [ ] Plugin 호환 버전 확인 (특히 analysis-nori 등)
- [ ] Security 설정 마이그레이션 (8.0 진입 시 핵심)
- [ ] Rolling upgrade 절차 숙지: shard allocation 비활성 → 노드 한 대씩 stop/start → 검증
- [ ] ILM, Snapshot, Watcher 일시 정지 (선택)

---

## 알려진 호환성 함정

- **7.x로 색인된 mapping을 8.x에서 그대로 쓸 때**: 일부 필드 옵션이 deprecated. `GET /_migration/deprecations`로 사전 점검
- **Java client 라이브러리 버전 ↔ 서버 버전 매트릭스**: REST API는 호환되나 클라이언트 자체는 보통 +/-1 마이너 호환 권장
- **Plugin (Nori 등)**: 클러스터 메이저와 정확히 일치하는 플러그인 버전 필요
- **Cross-cluster search/replication**: 메이저 버전 차이가 크면 미지원. 단계적 업그레이드

---

## 참고

| 리소스 | URL |
|--------|-----|
| Release notes | https://www.elastic.co/guide/en/elasticsearch/reference/current/es-release-notes.html |
| Breaking changes | https://www.elastic.co/guide/en/elasticsearch/reference/current/breaking-changes.html |
| Migration guide | https://www.elastic.co/guide/en/elasticsearch/reference/current/migrating.html |
| Support matrix | https://www.elastic.co/support/matrix |

---

*관련 cheatsheets: troubleshooting_apis, ilm_policy_templates*
