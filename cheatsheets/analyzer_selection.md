# Analyzer 선택

분석기는 색인·검색 시점의 텍스트 처리 파이프라인이다. 잘못 고르면 검색이 안 되거나 디스크가 부풀거나 한국어가 깨진다.

---

## Analyzer 파이프라인

```
입력 텍스트
   ↓
[char_filter]   문자 단위 전처리 (HTML strip, regex 치환)
   ↓
[tokenizer]     문자열을 토큰으로 분할 (단 1개)
   ↓
[token_filter]  토큰 단위 후처리 (소문자화, stopwords, stemming, synonyms)
   ↓
색인되는 토큰들
```

색인 시 analyzer와 검색 시 analyzer는 다를 수 있다 (`analyzer` vs `search_analyzer`).

---

## 빌트인 Analyzer

| Analyzer | 동작 | 권장 케이스 |
|----------|------|-------------|
| `standard` (기본) | Unicode 토큰화 + 소문자화 | 영어 등 라틴 계열 일반 |
| `simple` | 비문자 분할 + 소문자화 | 매우 단순한 텍스트 |
| `whitespace` | 공백 분할만 | 코드, 로그 식별자 |
| `keyword` | 그대로 단일 토큰 | 사실상 keyword 타입과 동등 |
| `pattern` | regex 분할 | 구분자가 특이한 텍스트 |
| `stop` | simple + stopwords | 영어 stopword 제거 |
| `english`, `french`, ... | 언어별 (stem + stopword) | 해당 언어 본문 |
| `nori` (플러그인) | 한국어 형태소 분석 | 한국어 본문 |

---

## 언어별 선택

### 영어

```json
{ "body": { "type": "text", "analyzer": "english" } }
```

`english` analyzer = `standard tokenizer` + `english_possessive_stemmer` + `lowercase` + `english_stop` + `english_stemmer`.

`running, runs, ran` → `run`으로 stemming. 검색 정확도 향상.

### 한국어 (Nori)

플러그인 설치:
```
bin/elasticsearch-plugin install analysis-nori
```

```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "korean": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": ["nori_part_of_speech", "lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "body": { "type": "text", "analyzer": "korean" }
    }
  }
}
```

#### Nori vs 기본 standard

```
입력: "한국어 형태소 분석기"

standard:  ["한국어", "형태소", "분석기"]   ← 공백만 분할
nori:      ["한국어", "한국", "어", "형태소", "형태", "분석기", "분석", "기"]
           (decompound + POS 태깅 + 사전 활용)
```

`nori_part_of_speech` 필터로 조사·어미 등 검색에 무용한 품사 제거.

#### Nori 옵션

```json
{
  "tokenizer": {
    "nori_user_dict": {
      "type": "nori_tokenizer",
      "decompound_mode": "mixed",
      "user_dictionary": "userdict_ko.txt"
    }
  }
}
```

| `decompound_mode` | 동작 |
|-------------------|------|
| `none` | 복합어 그대로 |
| `discard` (기본) | 복합어 분해, 원본 버림 |
| `mixed` | 분해 + 원본 모두 |

검색 recall 우선 → `mixed`. 정밀도 우선 → `discard`.

#### 사용자 사전

```
# userdict_ko.txt
삼성전자
한국전력공사
c++,c++
```

도메인 특수 용어를 보존. ES 재시작 또는 인덱스 close/open 후 적용.

---

## 검색어 자동완성 (Search-as-you-type)

### 옵션 1: search_as_you_type 타입

```json
{ "title": { "type": "search_as_you_type" } }
```

내부적으로 `title`, `title._2gram`, `title._3gram`, `title._index_prefix` 4개 sub-field 자동 생성.

```json
{ "multi_match": {
    "query": "elas",
    "type":  "bool_prefix",
    "fields": ["title", "title._2gram", "title._3gram"]
  }
}
```

### 옵션 2: edge_ngram custom

```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete": {
          "tokenizer": "autocomplete_tk",
          "filter": ["lowercase"]
        },
        "autocomplete_search": {
          "tokenizer": "lowercase"
        }
      },
      "tokenizer": {
        "autocomplete_tk": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 10,
          "token_chars": ["letter", "digit"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "autocomplete",
        "search_analyzer": "autocomplete_search"
      }
    }
  }
}
```

**중요**: 색인은 edge_ngram, **검색은 일반 분석기**. 안 그러면 검색어 "elasticsearch"가 ngram되어 노이즈가 폭증한다.

### 옵션 3: completion suggester

```json
{ "title": { "type": "completion" } }
```

FST 기반 in-memory 자동완성. 매우 빠르나 fuzzy·prefix 조합 제한적. 단순 prefix 자동완성에 최적.

---

## 태그 필드: 분석 OFF

```json
{ "tags": { "type": "keyword" } }
```

토큰화 X, 소문자화 X. `["AWS", "aws"]`는 다른 값. 정규화가 필요하면 `normalizer`:

```json
{
  "settings": {
    "analysis": {
      "normalizer": {
        "lowercase_normalizer": {
          "type": "custom",
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "tags": { "type": "keyword", "normalizer": "lowercase_normalizer" }
    }
  }
}
```

`normalizer`는 keyword 전용, 단일 토큰 후처리만. 토큰화는 못 한다.

---

## Synonyms (동의어)

### 인라인

```json
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonyms": {
          "type": "synonym_graph",
          "synonyms": [
            "tv, television",
            "smart phone => smartphone, mobile"
          ]
        }
      },
      "analyzer": {
        "with_syn": {
          "tokenizer": "standard",
          "filter": ["lowercase", "my_synonyms"]
        }
      }
    }
  }
}
```

### Synonyms API (8.10+)

```
PUT _synonyms/my-set
{ "synonyms_set": [ { "id": "1", "synonyms": "tv, television" } ] }
```

런타임에 동의어 변경 가능. 인덱스 close/open 불필요.

**`synonym_graph` vs `synonym`**: graph는 multi-word phrase를 정확히 처리. 검색 시 분석기에서 사용. 색인 시 분석기에서는 일반 `synonym` 권장.

---

## Multi-field로 다중 분석

같은 데이터를 여러 분석기로 색인 → 검색 시 most_fields multi_match.

```json
{
  "title": {
    "type": "text",
    "analyzer": "standard",
    "fields": {
      "english": { "type": "text", "analyzer": "english" },
      "korean":  { "type": "text", "analyzer": "korean" },
      "ngram":   { "type": "text", "analyzer": "autocomplete", "search_analyzer": "autocomplete_search" }
    }
  }
}
```

```json
{ "multi_match": {
    "query": "...",
    "fields": ["title^2", "title.english", "title.korean"],
    "type": "most_fields"
  }
}
```

---

## Analyzer 테스트

```
POST /_analyze
{
  "analyzer": "english",
  "text": "Running quickly to the stations"
}
```

```
POST /my-index/_analyze
{
  "field": "title",
  "text": "한국어 형태소 분석기"
}
```

매핑 변경 전·후 반드시 `_analyze`로 토큰을 직접 확인. 검색 안 되는 문제의 90%는 analyzer 미스매치.

---

## Nori vs custom 비교

| 항목 | Nori | 직접 구성 (custom) |
|------|------|---------------------|
| 도입 비용 | 플러그인 설치 1회 | 설계·튜닝 다수 |
| 정확도 | 높음 (사전 + POS) | 토크나이저 따라 변동 |
| 사용자 사전 | 지원 | 토크나이저별 |
| 동의어 | filter 결합 가능 | 가능 |
| 운영 | ES 버전 호환 매트릭스 확인 | 자유 |
| 권장 | 한국어 본문이면 기본값 | Nori가 안 맞는 도메인 (코드, 숫자 ID 위주) |

---

## 운영 체크리스트

- [ ] 색인 analyzer와 검색 analyzer가 일치하거나 의도적으로 분리되어 있다
- [ ] `_analyze` API로 실제 토큰을 확인했다
- [ ] 한국어 텍스트는 Nori (또는 동급) 사용 중
- [ ] 자동완성은 별도 multi-field로 분리 (본문 분석기와 섞지 않음)
- [ ] 태그는 `keyword` (+ 필요시 normalizer)
- [ ] synonyms는 검색 시 분석기에서만 적용
- [ ] 사용자 사전은 버전 관리되고 있다
- [ ] 인덱스 단위 analyzer 변경은 reindex가 필요함을 인지한다

---

*관련 cheatsheets: mapping_field_types, query_dsl_quickref*
