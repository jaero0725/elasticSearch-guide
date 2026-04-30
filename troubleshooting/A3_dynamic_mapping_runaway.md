# A3. Dynamic Mapping 무한 증식

## 증상

```
Limit of total fields [1000] has been exceeded
```

또는 시간이 갈수록 다음과 같은 현상:

- 새 매핑이 매시간 추가됨 (`PUT mapping` 로그가 끊임없이 발생)
- 클러스터 상태 메모리 사용량이 매주 1~5%씩 증가
- 마스터 노드의 GC 시간이 길어짐 (cluster state publish 지연)
- 인덱스가 read-only로 락(`index.blocks.write: true`)
- Kibana Discover가 점점 느려지다 멈춤 (필드 카탈로그 로딩)

---

## 원인

이벤트 페이로드의 키가 다양할 때 발생.

```json
// 사용자 행동 이벤트
{ "event": "click", "props": { "button_id_signup":  true } }
{ "event": "click", "props": { "button_id_login":   true } }
{ "event": "click", "props": { "button_id_buy_42": true } }
// ... 매일 새 button_id_* 키가 추가됨

// 동적 매핑 결과:
// props.button_id_signup → boolean
// props.button_id_login → boolean
// props.button_id_buy_42 → boolean
// 매핑 = 무한 증식
```

다른 흔한 패턴:

```json
// API gateway 로그에 동적 헤더
{ "headers": { "x-request-id": "...", "x-user-token": "...", "x-feature-flag-promo-2025": "..." } }

// 머신러닝 feature 벡터를 JSON 객체로
{ "features": { "feat_001": 0.5, "feat_002": 0.3, "feat_003": 0.7, ... } }
// → 수만 feature 키가 매핑에 등록됨

// IoT 센서 이름이 device 모델별로 다름
{ "telemetry": { "temp_zone_1": 23.5, "humidity_zone_1": 60 } }
{ "telemetry": { "vibration_motor_a": 0.02 } }
```

핵심: **JSON 객체의 키가 데이터 자체일 때**, 동적 매핑은 그 키를 하나하나 새 필드로 등록한다.

---

## 진단

### 1) 인덱스의 매핑 필드 수

```bash
GET /events/_mapping

# 응답을 jq로 카운트하거나
# 또는 field_caps 응답 키 수
GET /events/_field_caps?fields=*
```

### 2) 어떤 prefix가 폭증 중인지

```bash
# 매핑을 받아 prefix 별로 묶어 카운트
GET /events/_mapping/field/*

# 또는 _cluster/state 직접 조회 (관리자)
GET /_cluster/state/metadata/events?filter_path=metadata.indices.events.mappings.properties
```

### 3) 매핑 변경 추세 (cluster log)

```bash
# 마스터 로그에서 mapping update 빈도 확인
# elasticsearch.log 에서:
#   "[events] update_mapping [_doc]"
# 이 라인이 분당 수십~수백 회 나오면 동적 매핑 폭증 중
```

### 4) 클러스터 상태 크기

```bash
GET /_cluster/state?human&filter_path=cluster_uuid
# 응답 시간이 1초 이상이면 cluster state 비대 의심

GET /_nodes/stats/jvm?filter_path=nodes.*.jvm.mem.heap_used_percent
# 마스터 노드 heap이 70% 이상이면 위험
```

---

## 해결

### 단계 1: 인덱싱 즉시 차단 또는 dynamic 끄기

```bash
# 옵션 A: dynamic 을 false 로 (저장은 되되 매핑 안 함)
PUT /events/_mapping
{
  "dynamic": "false"
}

# 옵션 B: strict 로 (새 필드 시도 시 색인 거부 → 호출자가 알게 됨)
PUT /events/_mapping
{
  "dynamic": "strict"
}

# 옵션 C: runtime 으로 (매핑 안 하고 검색 시점에 처리)
PUT /events/_mapping
{
  "dynamic": "runtime"
}
```

### 단계 2: 가변 키를 `flattened`로 격리

```bash
# 새 인덱스 매핑
PUT /events-v2
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "@timestamp":  { "type": "date" },
      "event":       { "type": "keyword" },
      "user_id":     { "type": "keyword" },
      "props":       { "type": "flattened" },     // 가변 키들을 한 필드로
      "headers":     { "type": "flattened" },
      "features":    { "type": "flattened" }
    }
  }
}
```

`flattened` 동작:

```
입력: { "props": { "button_id_signup": true, "page": "/home" } }

저장 (내부적으로):
  props = ["button_id_signup", "true", "page", "/home"]
  → keyword 토큰 셋. 매핑 폭증 없음.

쿼리:
  { "term": { "props.button_id_signup": "true" } }   ← 동작
  { "exists": { "field": "props.page" } }            ← 동작

제약:
  - 모든 값이 keyword (숫자도 문자열로)
  - text 분석 안 됨
  - range 쿼리 제한적
```

### 단계 3: key/value 쌍으로 정규화 (대안)

가변 키를 nested array로 풀기:

```json
// 변환 전
{ "props": { "button_id_signup": true, "page": "/home" } }

// 변환 후 (ingest pipeline 또는 애플리케이션에서)
{
  "props": [
    { "key": "button_id_signup", "value": "true" },
    { "key": "page",             "value": "/home" }
  ]
}
```

매핑:

```json
"props": {
  "type": "nested",
  "properties": {
    "key":   { "type": "keyword" },
    "value": { "type": "keyword" }
  }
}
```

쿼리 (nested 쿼리 필요):

```json
{
  "nested": {
    "path": "props",
    "query": {
      "bool": {
        "must": [
          { "term": { "props.key":   "button_id_signup" } },
          { "term": { "props.value": "true" } }
        ]
      }
    }
  }
}
```

장단점:

```
flattened:    매핑 1개. 쿼리 단순. 그러나 모든 값 keyword.
nested:       매핑 2개 (key/value). 쿼리 복잡 (nested). 값 타입 다양 가능.
                                      → 메트릭/숫자 값 섞이면 nested 권장
```

### 단계 4: ingest pipeline으로 변환

```json
PUT /_ingest/pipeline/flatten-props
{
  "processors": [
    {
      "script": {
        "source": """
          if (ctx.props instanceof Map) {
            def list = new ArrayList();
            for (entry in ctx.props.entrySet()) {
              def m = new HashMap();
              m.put('key',   entry.getKey());
              m.put('value', String.valueOf(entry.getValue()));
              list.add(m);
            }
            ctx.props = list;
          }
        """
      }
    }
  ]
}
```

인덱스에 `index.default_pipeline: flatten-props` 설정.

### 단계 5: Reindex로 마이그레이션

```bash
POST /_reindex
{
  "source": { "index": "events" },
  "dest":   { "index": "events-v2", "pipeline": "flatten-props" }
}
```

---

## 예방

### 1) 인덱스 템플릿에서 가변 키 영역을 미리 격리

```json
PUT /_index_template/events-template
{
  "index_patterns": ["events-*"],
  "template": {
    "settings": {
      "index.mapping.total_fields.limit": 1500
    },
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "@timestamp": { "type": "date" },
        "event":      { "type": "keyword" },
        "user_id":    { "type": "keyword" },
        "props":      { "type": "flattened" },
        "context":    { "type": "flattened" }
      }
    }
  }
}
```

### 2) Dynamic Templates로 자동 격리

```json
"dynamic_templates": [
  {
    "props_to_flattened": {
      "path_match": "props.*",
      "mapping": { "type": "flattened" }
    }
  }
]
```

### 3) 모니터링

```
경보 기준:
  - 인덱스당 필드 수가 일주일에 10개 이상 증가하면 알람
  - cluster state size > 100MB 알람
  - mapping update rate > 분당 5회 알람
```

```bash
# 모니터링 쿼리
GET /_field_caps?fields=*&include_unmapped=false
# 시간별 응답 크기 추세를 외부 모니터링에 저장
```

### 4) 데이터 모델링 원칙

```
"키 자체가 데이터인가?"
  → 그렇다면 절대 plain object로 색인하지 말 것
  → flattened 또는 key/value nested 로

"키가 정해진 스키마인가?"
  → 명시적 properties로 매핑
  → dynamic: strict
```

**핵심: 동적 매핑은 편리하지만 운영의 적이다. 신규 인덱스는 항상 `dynamic: "strict"`로 시작하라.**
