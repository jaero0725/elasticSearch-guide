# 6장. Text Analysis — Analyzer 파이프라인과 한국어 처리

text 필드에 들어간 문자열은 그대로 저장되지 않는다. **analyzer**라는 파이프라인을 거쳐 토큰들로 분해되고, 각 토큰이 역색인의 키가 된다. 검색 시 질의어도 같은 analyzer를 거쳐야 하므로, analyzer 설계는 곧 **무엇이 매칭되고 무엇이 매칭되지 않는지**를 결정한다.

공식 문서: [Text analysis](https://www.elastic.co/docs/manage-data/data-store/text-analysis)

---

## 6.1 Analyzer 파이프라인: Char filter → Tokenizer → Token filter

모든 analyzer는 정확히 다음 3단계로 구성된다:

```
원본 문자열
    │
    ▼
┌─────────────────┐
│ Character Filter│  0개 이상. 문자 단위 전처리
│                 │  (HTML 제거, 문자 치환 등)
└─────────────────┘
    │
    ▼
┌─────────────────┐
│  Tokenizer      │  정확히 1개. 문자열을 토큰으로 분해
│                 │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ Token Filter    │  0개 이상. 토큰 단위 변환
│                 │  (lowercase, stop, synonym, stemming)
└─────────────────┘
    │
    ▼
역색인에 저장될 토큰들
```

### 단계별 예시: standard analyzer

입력: `"<p>The 2 Quick Brown-Foxes!</p>"`

```
1) char_filter (없음)
   "<p>The 2 Quick Brown-Foxes!</p>"

2) standard tokenizer (Unicode word boundary)
   ["<p>The", "2", "Quick", "Brown", "Foxes", "p"]
   → 실제로는 standard tokenizer가 <,>,/ 등도 분리: ["p", "The", "2", "Quick", "Brown", "Foxes", "p"]

3) lowercase token filter
   ["p", "the", "2", "quick", "brown", "foxes", "p"]
```

결과 토큰들이 역색인의 키가 된다. 검색 시 질의어도 동일하게 처리되므로 `"Quick"`이나 `"quick"`으로 검색해도 매칭된다.

> 참고: standard analyzer에는 stop filter가 기본 비활성. `"the"`도 색인된다. 필요하면 `stop` filter를 추가.

---

## 6.2 내장 Analyzer

| Analyzer | 동작 | 용도 |
|----------|------|------|
| `standard` (기본) | Unicode 텍스트 분할 + lowercase | 다국어 일반 검색 |
| `simple` | 영문자 외 모두 분할 + lowercase | 단순 영문 |
| `whitespace` | 공백으로만 분할. lowercase 안 함 | 토큰을 그대로 보존 (코드 등) |
| `stop` | simple + 불용어 제거 | 영문 단순 검색 |
| `keyword` | 분석하지 않음 — 입력 전체가 1개 토큰 | text 필드인데 분석 안 하고 싶을 때 |
| `pattern` | 정규식 분할 | 커스텀 구분자 |
| `language` | 언어별 stemmer 등 (`english`, `korean`, ...) | 언어 특화 |

```json
POST /_analyze
{
  "analyzer": "standard",
  "text": "The 2 Quick Brown-Foxes!"
}
// 결과 토큰: the, 2, quick, brown, foxes
```

---

## 6.3 Custom Analyzer 만들기

내장 analyzer로 부족하면 char filter / tokenizer / token filter를 조합한다.

```json
PUT /blog
{
  "settings": {
    "analysis": {
      "char_filter": {
        "html_strip_keep_emoji": {
          "type": "html_strip"
        }
      },
      "filter": {
        "english_stop": {
          "type": "stop",
          "stopwords": "_english_"
        },
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        }
      },
      "analyzer": {
        "blog_text": {
          "type": "custom",
          "char_filter": ["html_strip_keep_emoji"],
          "tokenizer":   "standard",
          "filter":      ["lowercase", "english_stop", "english_stemmer"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "body": { "type": "text", "analyzer": "blog_text" }
    }
  }
}
```

**핵심**: `analyzer`는 색인 시점과 검색 시점 모두에 적용된다. 다르게 적용하려면 `search_analyzer`를 별도로 지정.

```json
"body": {
  "type": "text",
  "analyzer": "blog_text",          // 색인 시
  "search_analyzer": "blog_search"  // 검색 시
}
```

대표적 비대칭 사용 사례: **자동완성**(색인 시 edge_ngram, 검색 시 standard).

---

## 6.4 한국어 분석: Nori

영어와 달리 한국어는 공백이 의미 단위와 일치하지 않는다. `"검색엔진을"`은 `"검색엔진" + "을"`로 분해되어야 한다. 이를 위해 **형태소 분석기(morphological analyzer)**가 필수다.

ES 공식 한국어 플러그인이 **Nori**다 (mecab-ko-dic 기반).

### 설치

```bash
bin/elasticsearch-plugin install analysis-nori
# 모든 노드에 설치 후 재시작 필요
```

### 기본 사용

```json
POST /_analyze
{
  "analyzer": "nori",
  "text": "검색엔진을 공부합니다"
}
// tokens: 검색, 엔진, 공부, 하
```

`nori` analyzer는 다음 3개의 조합:
- `nori_tokenizer` — 형태소 분해
- `nori_part_of_speech` token filter — 조사/어미 등 제거
- `nori_readingform` token filter — 한자를 한글로 변환

### 중요 파라미터: decompound_mode

복합명사를 어떻게 처리할지 결정한다.

| 모드 | 입력 `가곡역` | 의미 |
|------|------|------|
| `none` | `[가곡역]` | 분해 안 함 |
| `discard` (기본) | `[가곡, 역]` | 분해하고 원본 버림 |
| `mixed` | `[가곡역, 가곡, 역]` | 분해하고 원본도 유지 |

**검색 정확도 vs 재현율 트레이드오프**:
- `discard`: `"가곡역"` 검색 시 `"가곡"`만 있는 문서도 매칭됨 (재현율↑)
- `mixed`: 정확 매칭과 부분 매칭 둘 다 가능 (재현율↑, 점수 분포 풍부)
- `none`: 복합명사 그대로 → 부분 매칭 못 함 (정밀도↑)

### 운영 매핑 예시

```json
PUT /korean_news
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "korean_nori_tokenizer": {
          "type": "nori_tokenizer",
          "decompound_mode": "mixed",
          "user_dictionary": "userdict_ko.txt"
        }
      },
      "filter": {
        "korean_pos_filter": {
          "type": "nori_part_of_speech",
          "stoptags": [
            "E",   "IC", "J", "MAG", "MAJ", "MM",
            "SP",  "SSC", "SSO", "SC", "SE",
            "XPN", "XSA", "XSN", "XSV",
            "UNA", "NA", "VSV"
          ]
        }
      },
      "analyzer": {
        "korean_analyzer": {
          "type": "custom",
          "tokenizer": "korean_nori_tokenizer",
          "filter": [
            "korean_pos_filter",
            "nori_readingform",
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "korean_analyzer" },
      "body":  { "type": "text", "analyzer": "korean_analyzer" }
    }
  }
}
```

**stoptags의 의미**: 조사(J), 어미(E), 보조용언(VSV) 등 의미 없는 품사를 제거한다. 위 리스트가 일반적 권장값.

**user_dictionary**: 신조어, 고유명사 등을 사전에 추가. 노드 파일 시스템(`config/userdict_ko.txt`)에 위치.

```
# userdict_ko.txt
삼성전자
판교역
파이썬
```

공식 문서: [nori_tokenizer](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori-tokenizer.html), [nori analyzer](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori-analyzer.html)

---

## 6.5 동의어 (Synonyms)

### synonym vs synonym_graph

ES는 두 종류의 synonym filter를 제공한다.

| 필터 | 차이 |
|------|------|
| `synonym` | 단순 1:1 / 1:N 토큰 치환. multi-word 동의어에서 phrase 쿼리가 깨질 수 있음 |
| `synonym_graph` | 토큰 그래프 기반. multi-word를 정확히 처리 (`ny, new york`). **권장** |

```json
"filter": {
  "my_synonyms": {
    "type": "synonym_graph",
    "synonyms": [
      "노트북, 랩탑, laptop",
      "휴대폰, 핸드폰, 스마트폰",
      "ai => 인공지능"   // explicit mapping (단방향)
    ]
  }
}
```

### 색인 시점 vs 검색 시점

```
색인 시점 적용 (analyzer):
  + 검색이 빠름 (이미 확장됨)
  - 동의어 변경 시 재색인 필요
  - 동의어 사전이 색인에 박혀 있음

검색 시점 적용 (search_analyzer):
  + 동의어 변경 즉시 반영
  - 검색 시 매번 확장 → 약간 느림
  - 운영 친화적
```

대부분의 운영에서 **검색 시점 + reload 가능 형태**를 권장한다.

```json
"analyzer": {
  "korean_index": {                    // 색인용 — 동의어 없음
    "type": "custom",
    "tokenizer": "korean_nori_tokenizer",
    "filter": ["korean_pos_filter", "lowercase"]
  },
  "korean_search": {                   // 검색용 — 동의어 적용
    "type": "custom",
    "tokenizer": "korean_nori_tokenizer",
    "filter": ["korean_pos_filter", "lowercase", "my_synonyms"]
  }
}
```

mapping에서:

```json
"title": {
  "type": "text",
  "analyzer":        "korean_index",
  "search_analyzer": "korean_search"
}
```

### 8.x: Synonyms API + reload

8.x부터는 synonym set을 클러스터에 등록하고 파일 없이 관리할 수 있다.

```json
PUT /_synonyms/my-synonyms-set
{
  "synonyms_set": [
    { "id": "rule-1", "synonyms": "노트북, 랩탑" },
    { "id": "rule-2", "synonyms": "휴대폰, 핸드폰, 스마트폰" }
  ]
}
```

```json
"filter": {
  "my_synonyms": {
    "type": "synonym_graph",
    "synonyms_set": "my-synonyms-set",
    "updateable": true
  }
}
```

`updateable: true`이면 검색 시점에만 사용 가능하지만, **사전을 변경한 후 인덱스 reload만으로 즉시 반영**된다 — 재색인 불필요.

```bash
POST /korean_news/_reload_search_analyzers
```

공식 문서: [Synonyms API](https://www.elastic.co/docs/api/doc/elasticsearch/group/endpoint-synonyms), [synonym_graph token filter](https://www.elastic.co/docs/reference/elasticsearch/text-analysis/analysis-synonym-graph-tokenfilter)

---

## 6.6 N-gram, Edge N-gram — 자동완성

부분 문자열 검색이나 자동완성을 위한 토큰 필터.

### N-gram

```
"검색"  →  ["검", "색", "검색"]   (min=1, max=2)
```

모든 위치의 N-gram을 만든다. 부분 매칭은 강력하지만 색인 크기가 폭증.

### Edge N-gram

```
"검색엔진"  →  ["검", "검색", "검색엔", "검색엔진"]   (min=1, max=10)
```

**시작 부분**의 prefix만 생성. 자동완성에 최적.

### 자동완성 매핑

```json
PUT /autocomplete
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "edge_ngram_tokenizer": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 20,
          "token_chars": ["letter", "digit"]
        }
      },
      "analyzer": {
        "edge_ngram_index": {
          "tokenizer": "edge_ngram_tokenizer",
          "filter":    ["lowercase"]
        },
        "edge_ngram_search": {
          "tokenizer": "standard",
          "filter":    ["lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer":        "edge_ngram_index",
        "search_analyzer": "edge_ngram_search"
      }
    }
  }
}
```

**중요**: 색인 시점에만 edge_ngram을 적용하고 **검색 시점에는 standard를 사용**해야 한다. 검색 시점에도 edge_ngram을 쓰면 `"검색"`을 검색했을 때 `"검"`만 있는 모든 문서가 매칭되어버린다.

```
색인: "검색엔진"  → [검, 검색, 검색엔, 검색엔진]
검색: "검색" (standard) → [검색]
                     ↓
              매칭됨: "검색엔진" 문서
              매칭 안 됨: "검" 만 있는 문서 (의도대로)
```

### 대안: completion suggester

자동완성 전용 자료구조 `completion` 필드 타입은 검색 자동완성용으로 7장에서 다룬다 (간략 소개, FST 기반).

---

## 6.7 `_analyze` API로 디버깅

analyzer가 의도대로 동작하는지 항상 `_analyze`로 검증해야 한다.

### 임시 analyzer 테스트

```json
POST /_analyze
{
  "tokenizer": "nori_tokenizer",
  "filter":    ["nori_part_of_speech", "lowercase"],
  "text":      "삼성전자의 새로운 갤럭시 노트북"
}
```

### 인덱스에 정의된 analyzer 사용

```json
POST /korean_news/_analyze
{
  "analyzer": "korean_search",
  "text":     "검색엔진 동의어 처리"
}
```

### 특정 필드의 analyzer 사용

```json
POST /korean_news/_analyze
{
  "field": "title",
  "text":  "검색엔진 동의어 처리"
}
```

### 토큰의 상세 정보 (explain)

```json
POST /korean_news/_analyze
{
  "field": "title",
  "text":  "검색엔진을",
  "explain": true
}
```

→ 각 단계(tokenizer → 각 filter)별 토큰 변화, position, offset, 품사 태그까지 확인 가능. 동의어/품사 필터 디버깅의 핵심 도구.

```
ASCII: 디버깅의 황금 패턴
─────────────────────────────────────────────────────
  검색이 의도와 다르게 동작 →
    1) 검색어를 _analyze로 토큰화
    2) 색인된 문서의 해당 필드를 _analyze로 토큰화
    3) 두 토큰 집합의 교집합 확인
    4) 교집합 비어 있으면 매칭 안 되는 게 당연
─────────────────────────────────────────────────────
```

---

## 6.8 운영 팁

### 동의어 사전의 무중단 갱신 (8.x)

1. Synonyms API로 클러스터에 사전 등록 (`PUT /_synonyms/my-set`)
2. mapping에 `updateable: true`로 search_analyzer만 사용
3. 사전 수정 후 `POST /index/_reload_search_analyzers`
4. 재색인 불필요, 즉시 반영

### Nori user_dictionary의 hot reload

`user_dictionary_rules`로 mapping 안에 직접 단어 리스트를 넣으면, 사전을 변경하려면 인덱스를 다시 만들어야 한다.

```json
"korean_nori_tokenizer": {
  "type": "nori_tokenizer",
  "decompound_mode": "mixed",
  "user_dictionary_rules": ["삼성전자", "판교역"]
}
```

vs 파일 기반(`user_dictionary: "userdict_ko.txt"`)은 노드 재시작이 필요. 운영에선 ILM rollover 시 신규 인덱스에 새 사전이 적용되는 패턴이 흔하다.

### Analyzer 비용 측정

복잡한 analyzer는 색인 처리량을 떨어뜨린다. 벤치마크로 영향 측정:
- bulk indexing throughput 비교 (analyzer 변경 전후)
- `_nodes/stats?metric=indices`의 indexing time 관찰
- text 필드 1개당 analyzer 1회 → 필드 수 × 문서 수만큼 실행됨

### 한국어 + 영어 혼용

뉴스/블로그 본문은 한·영 혼용이 일반적이다. Nori는 영어 토큰을 잘 처리하지만, stemming까지 원하면 multi-fields로 분리:

```json
"body": {
  "type": "text",
  "analyzer": "korean_analyzer",
  "fields": {
    "english": { "type": "text", "analyzer": "english" }
  }
}
```

검색 시 `multi_match`로 `body`와 `body.english` 동시 검색.

---

## 6.9 Key Takeaways

| 항목 | 핵심 |
|------|------|
| **Analyzer 파이프라인** | char_filter → tokenizer (1개) → token_filter |
| **색인 = 검색 동일 분석** | 기본 원칙. 의도적 비대칭은 `search_analyzer` |
| **Nori** | 한국어 형태소 분석. `decompound_mode: mixed`가 운영 베이스라인 |
| **stoptags** | 조사·어미를 제거하는 nori_part_of_speech 필터의 핵심 설정 |
| **synonym_graph** | multi-word 동의어 정확 처리. `synonym`보다 우선 사용 |
| **검색 시점 동의어 + Synonyms API** | 8.x 운영 표준. 재색인 없이 사전 갱신 |
| **edge_ngram** | 자동완성. 색인에만 적용, 검색은 standard 분리 필수 |
| **`_analyze` API** | 매칭이 안 될 때 첫 번째 디버깅 도구 |

---

*다음 장: 7장. Query DSL — bool 쿼리, 점수 계산, 페이지네이션*
