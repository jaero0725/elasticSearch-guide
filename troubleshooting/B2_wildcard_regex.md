# B2. 와일드카드 / Regex 쿼리로 CPU 폭증

## 증상

- 특정 검색이 들어오면 클러스터 CPU가 100%로 치솟음
- 다른 정상 쿼리들의 응답 시간도 함께 느려짐
- 코디네이팅 노드의 search thread pool queue가 가득 참
- circuit breaker exception:

```
{
  "type": "circuit_breaking_exception",
  "reason": "[parent] Data too large, data for [<reused_arrays>] would be ..."
}
```

또는:

```
{
  "type": "too_complex_to_determinize_exception",
  "reason": "Determinizing automaton with 10001 states would result in more
             than 10000 states."
}
```

---

## 원인

### Leading Wildcard

```json
// CPU 폭주의 주범
{ "wildcard": { "username": "*kim" } }
{ "query_string": { "query": "*error*" } }
```

```
Lucene의 inverted index는 "토큰의 prefix tree"
  → 'apple', 'app' 으로 시작하는 토큰 검색은 매우 빠름
  → '*ple' 같이 끝부분만 알면 모든 토큰을 다 봐야 함

수억 개 토큰이 있는 인덱스에서 leading wildcard는
  → 사실상 풀스캔 + automaton 매칭
  → CPU 시간 O(N × pattern_complexity)
```

### 복잡한 Regex

```json
// 문제적
{ "regexp": { "log": ".*error.*timeout.*5\\d{3}.*" } }
{ "regexp": { "url": "(http|https)://([a-z0-9-]+\\.)+[a-z]{2,6}/.*" } }
```

복잡한 정규식은 자동기계(automaton) 상태가 폭증하여 위 `too_complex_to_determinize_exception` 발생.

### `query_string` / `simple_query_string`의 위험

```json
// 사용자 입력을 그대로 query_string 에 넣으면 위험
{ "query_string": { "query": "*" } }                  // 매치 올
{ "query_string": { "query": "name:*" } }             // 모든 문서
{ "query_string": { "query": "*foo* AND *bar*" } }    // 폭주
```

사용자가 의도적으로 또는 실수로 ES를 폭주시킬 수 있다.

---

## 진단

### 1) 현재 도는 쿼리 확인

```bash
GET /_tasks?actions=*search*&detailed=true

# 응답에서 task별 description 확인
# "description": "indices[my-index], search_type[...], source[..wildcard..]"
```

### 2) 무거운 쿼리 종료

```bash
# task_id로 캔슬
POST /_tasks/<task_id>/_cancel
```

### 3) Slow log 확인

```bash
PUT /my-index/_settings
{
  "index.search.slowlog.threshold.query.warn": "1s"
}

# log/elasticsearch_index_search_slowlog.log 에서
# wildcard / regexp 가 들어간 쿼리 패턴 추출
```

### 4) Hot threads 분석

```bash
GET /_nodes/hot_threads?threads=10&interval=500ms&snapshots=10
# search thread에서 RegExp / Automaton 호출이 자주 보이면 의심
```

### 5) 인덱스의 `allow_expensive_queries` 설정

```bash
GET /_cluster/settings?include_defaults=true&filter_path=**.search.allow_expensive_queries
```

---

## 해결

### 단계 1: 즉각 차단 — expensive queries 비활성화

```bash
PUT /_cluster/settings
{
  "transient": {
    "search.allow_expensive_queries": false
  }
}
```

이렇게 하면 leading wildcard, complex regex, script 쿼리 등이 거부된다:

```
{ "type": "elasticsearch_exception",
  "reason": "[wildcard] queries cannot be executed when
             'search.allow_expensive_queries' is set to false." }
```

> 응급용. 정상 쿼리도 영향 받을 수 있으니 영구 설정은 신중히.

### 단계 2: `wildcard` 필드 타입 사용 (8.x)

`wildcard` 타입은 grams 기반으로 `*pattern*`을 빠르게 처리하도록 설계됨.

```bash
PUT /logs-v2
{
  "mappings": {
    "properties": {
      "message":           { "type": "match_only_text" },
      "error_stack_trace": { "type": "wildcard" }
    }
  }
}

# leading wildcard 도 빠름
GET /logs-v2/_search
{
  "query": {
    "wildcard": {
      "error_stack_trace": "*NullPointerException*"
    }
  }
}
```

`wildcard` 필드의 특징:

```
- ngram 비슷하게 색인 시점에 부분 패턴을 인덱싱
- 디스크 사용량은 keyword보다 약간 큼 (1.2~1.5배)
- 패턴 길이 ≥ 3자 이상이면 매우 빠름
- stack trace, URL, 식별자 컬럼 등 부분 일치 검색이 잦은 필드에 적합
```

### 단계 3: 매핑 설계로 우회

```bash
# 도메인 분리 (URL 검색이 잦다면)
PUT /docs
{
  "mappings": {
    "properties": {
      "url": {
        "type": "keyword",
        "fields": {
          "host":   { "type": "keyword" },     // 호스트 부분만 별도 색인
          "path":   { "type": "keyword" },
          "ngram":  {
            "type": "text",
            "analyzer": "url_ngram"
          }
        }
      }
    }
  }
}
```

ingest pipeline에서 url을 host/path로 분리:

```json
PUT /_ingest/pipeline/url-split
{
  "processors": [
    {
      "uri_parts": {
        "field": "url",
        "target_field": "url_parts"
      }
    },
    { "set": { "field": "url.host", "value": "{{url_parts.domain}}" } },
    { "set": { "field": "url.path", "value": "{{url_parts.path}}" } }
  ]
}
```

### 단계 4: prefix만 가능하면 prefix 쿼리

```bash
# 좋음: 빠름 (prefix tree 활용)
GET /logs/_search
{ "query": { "prefix": { "username": "kim" } } }

# 나쁨: 느림
GET /logs/_search
{ "query": { "wildcard": { "username": "kim*" } } }   // prefix 와 동일하지만 prefix 쿼리가 더 명확
GET /logs/_search
{ "query": { "wildcard": { "username": "*kim" } } }   // leading wildcard, 느림
```

### 단계 5: edge_ngram 분석기로 자동완성/일부 일치

```json
PUT /logs
{
  "settings": {
    "analysis": {
      "analyzer": {
        "edge_ngram_analyzer": {
          "tokenizer": "edge_ngram_tk"
        }
      },
      "tokenizer": {
        "edge_ngram_tk": {
          "type": "edge_ngram",
          "min_gram": 2, "max_gram": 20
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "username": {
        "type": "keyword",
        "fields": {
          "edge": { "type": "text", "analyzer": "edge_ngram_analyzer" }
        }
      }
    }
  }
}

GET /logs/_search
{ "query": { "match": { "username.edge": "kim" } } }
```

### 단계 6: 사용자 입력을 query_string에 그대로 넣지 말 것

```javascript
// 위험
{ "query_string": { "query": userInput } }

// 안전: 검증·이스케이프
const escaped = userInput.replace(/([+\-=&|><!(){}\[\]^"~*?:\\\/])/g, '\\$1');
{ "match": { "field": userInput } }   // match 가 일반적으로 안전

// 또는 simple_query_string + flags 제한
{
  "simple_query_string": {
    "query": userInput,
    "fields": ["title", "body"],
    "flags": "AND|OR|NOT|PHRASE",     // wildcard/regexp/fuzzy 제외
    "default_operator": "AND"
  }
}
```

---

## 예방

### 1) Cluster 레벨 설정

`search.allow_expensive_queries`는 **cluster-level** 설정이다 (index-level 설정 아님). 8.x에서는 cluster 설정만 존재한다.

```bash
PUT /_cluster/settings
{
  "persistent": {
    "search.allow_expensive_queries": false
  }
}
```

### 2) 쿼리 게이트웨이/검증 레이어

```
애플리케이션 ↔ ES 사이에 게이트웨이 두기:
  - 화이트리스트된 쿼리 패턴만 통과
  - leading wildcard 차단
  - 정규식 길이/복잡도 제한
  - 결과 size 강제 cap
```

### 3) Search slow log 상시 모니터링

```bash
PUT /_template/all-slow-log
{
  "index_patterns": ["*"],
  "settings": {
    "index.search.slowlog.threshold.query.warn":  "2s",
    "index.search.slowlog.threshold.query.info":  "1s",
    "index.search.slowlog.threshold.fetch.warn":  "1s"
  }
}
```

### 4) 매핑 설계 가이드

```
필드 사용 패턴별 권장:

  prefix 매칭 자주     → keyword + prefix 쿼리 또는 edge_ngram
  *substring* 자주     → wildcard 타입
  정규식 복잡          → ingest pipeline 에서 미리 추출 → 별도 keyword 필드
  full-text 검색       → text + match 쿼리 (가장 빠르고 안전)
  structured 검색      → keyword + term/terms 쿼리
```

### 5) 안티패턴

- `query_string` 에 사용자 원본 입력 그대로 전달
- leading wildcard (`*foo`) 의 일상 사용
- `regexp` 로 모든 패턴 표현 (대안 없는지 확인)
- 텍스트 필드에 `wildcard` 쿼리 (text는 토큰 단위라 의도와 다르게 매치)

**핵심: 와일드카드/Regex는 "정 안 되면" 쓰는 도구. 매핑·analyzer 설계로 미리 우회하라.**
