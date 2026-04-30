# 예제 4. 보안 / SIEM

윈도우/리눅스 호스트의 process·network·authentication 이벤트를 수집해 위협을 탐지하는 SIEM 시나리오. EQL(Event Query Language), Detection Rule, threat intelligence index가 어떻게 결합되는지 본다.

대상 버전: Elasticsearch 8.x / 9.x (Kibana Security 앱 포함)

---

## 1. 요구사항

- 호스트 약 5,000대 (Windows 60%, Linux 40%)
- Elastic Agent + Endpoint Security Integration으로 수집
  - process 시작/종료
  - 네트워크 연결
  - file 생성/수정
  - authentication 성공/실패
- 탐지 시나리오:
  - PowerShell이 비정상 부모 프로세스에서 실행됨 (`winword.exe → powershell.exe`)
  - SSH brute force (10분간 5회 이상 실패 후 성공)
  - 외부 위협 IP와 통신
  - Living off the land (`certutil.exe -urlcache` 다운로드)
- 모든 알람은 SIEM 워크북에 기록, 심각도별 분류

---

## 2. ECS 기반 데이터 모델

보안 이벤트도 ECS를 따른다. 핵심 필드:

| 필드 | 의미 |
|------|------|
| `@timestamp` | 이벤트 시각 |
| `event.category` | `process`, `network`, `authentication`, `file` |
| `event.type` | `start`, `end`, `connection`, `change` |
| `event.action` | `executed`, `connection_attempted` 등 도메인 액션 |
| `event.outcome` | `success`, `failure` |
| `host.name`, `host.os.type` | 호스트 식별 |
| `user.name`, `user.id` | 인증/권한 주체 |
| `process.name`, `process.executable`, `process.command_line` | 프로세스 |
| `process.parent.name`, `process.parent.command_line` | 부모 프로세스 |
| `process.entity_id` | 프로세스 인스턴스 ID (sequence join에 필수) |
| `source.ip`, `destination.ip`, `destination.port` | 네트워크 |
| `file.path`, `file.hash.sha256` | 파일 |
| `threat.indicator.*` | 위협 IOC |

> `event.category`/`event.type`/`event.action` 3종 세트를 일관되게 채워야 EQL과 Detection Rule이 제대로 동작한다. **이벤트 표준화가 SIEM의 절반이다.**

---

## 3. Threat Intelligence 인덱스

외부 IOC(악성 IP/도메인/파일 해시)를 보관하고 라이브 이벤트와 매칭한다.

```json
PUT /_index_template/threat-intel-template
{
  "index_patterns": ["logs-ti.*"],
  "data_stream": {},
  "template": {
    "mappings": {
      "properties": {
        "@timestamp":              { "type": "date" },
        "threat.indicator.type":   { "type": "keyword" },
        "threat.indicator.ip":     { "type": "ip" },
        "threat.indicator.url.full": { "type": "keyword" },
        "threat.indicator.file.hash.sha256": { "type": "keyword" },
        "threat.indicator.first_seen": { "type": "date" },
        "threat.indicator.last_seen":  { "type": "date" },
        "threat.feed.name":        { "type": "keyword" },
        "threat.indicator.confidence": { "type": "keyword" }
      }
    }
  }
}
```

Filebeat threatintel module 또는 Elastic Agent의 Threat Intel integration이 자동으로 채운다.

---

## 4. EQL 기본 문법

EQL은 Elastic이 만든 시퀀스/조건 중심 쿼리 언어. SQL은 row 단위 조회, EQL은 **시간 순서가 있는 이벤트 시퀀스 매칭**에 강하다.

### 4.1 단일 이벤트 매칭

```json
GET /logs-endpoint.events.*/_eql/search
{
  "query": """
    process where event.type == "start" and process.name == "powershell.exe"
  """
}
```

### 4.2 sequence — 부모/자식 프로세스

`winword.exe`가 `powershell.exe`를 5초 안에 띄운 케이스 (전형적 매크로 멀웨어):

```json
GET /logs-endpoint.events.*/_eql/search
{
  "query": """
    sequence by host.name with maxspan=5s
      [process where event.type == "start"
        and process.name == "winword.exe"]
      [process where event.type == "start"
        and process.name == "powershell.exe"
        and process.parent.name == "winword.exe"]
  """,
  "size": 100
}
```

해석:
- `sequence by host.name`: 같은 호스트 안에서만 매칭
- `maxspan=5s`: 5초 이내 연쇄
- `[ ... ] [ ... ]`: 첫 번째 이벤트 → 두 번째 이벤트 순서

### 4.3 SSH brute force

10분 동안 같은 user에게 5회 실패 후 성공:

```json
GET /logs-system.auth-*/_eql/search
{
  "query": """
    sequence by user.name, host.name with maxspan=10m
      [authentication where event.outcome == "failure"]
      [authentication where event.outcome == "failure"]
      [authentication where event.outcome == "failure"]
      [authentication where event.outcome == "failure"]
      [authentication where event.outcome == "failure"]
      [authentication where event.outcome == "success"]
  """
}
```

### 4.4 LotL (Living off the Land) — certutil 다운로드

```json
GET /logs-endpoint.events.process-*/_eql/search
{
  "query": """
    process where event.type == "start"
      and process.name == "certutil.exe"
      and process.command_line : ("*urlcache*", "*-decode*")
  """
}
```

> EQL의 `:` 연산자는 case-insensitive 와일드카드 매칭. `==`와 차이 인지 필요.

### 4.5 threat intel 매칭 (lookup)

라이브 네트워크 이벤트 ↔ TI 인덱스 IP 비교:

```json
POST /_security/_query
GET /logs-endpoint.events.network-*/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "event.category": "network" } },
        { "range": { "@timestamp": { "gte": "now-15m" } } }
      ],
      "must": [
        {
          "terms": {
            "destination.ip": {
              "index": "logs-ti.malware-default",
              "id":    "_search",
              "path":  "threat.indicator.ip"
            }
          }
        }
      ]
    }
  }
}
```

> 8.x에서는 Detection Engine이 `indicator_match` 룰 타입으로 같은 패턴을 자동화한다.

---

## 5. Detection Rule (Kibana Alerting)

Kibana > Security > Rules에서 정의. 내부적으로는 ES alerting framework 위에서 동작.

### 5.1 룰 종류

| 타입 | 용도 |
|------|------|
| Custom query | 단순 쿼리 매칭 (KQL/Lucene) |
| Threshold | 카운트 임계치 초과 |
| EQL | sequence/조건 |
| Indicator match | 라이브 이벤트와 TI 인덱스 매칭 |
| ML | 이상 탐지 (anomaly job) |
| New terms | 처음 보는 user/host/process 출현 |

### 5.2 EQL 룰 예시 (API)

```json
POST /api/detection_engine/rules
{
  "rule_id": "winword-spawn-powershell",
  "name": "Word Document Spawning PowerShell",
  "description": "Microsoft Word가 PowerShell을 띄우면 매크로 악성코드 가능성",
  "type": "eql",
  "language": "eql",
  "index": ["logs-endpoint.events.process-*"],
  "query": "sequence by host.name with maxspan=5s [process where event.type==\"start\" and process.name==\"winword.exe\"] [process where event.type==\"start\" and process.name==\"powershell.exe\" and process.parent.name==\"winword.exe\"]",
  "severity": "high",
  "risk_score": 73,
  "from": "now-6m",
  "interval": "5m",
  "enabled": true,
  "tags": ["macro", "initial_access"],
  "threat": [
    {
      "framework": "MITRE ATT&CK",
      "tactic": { "id": "TA0002", "name": "Execution" },
      "technique": [{ "id": "T1059.001", "name": "PowerShell" }]
    }
  ],
  "actions": [
    {
      "id": "slack-soc",
      "action_type_id": ".slack",
      "params": { "message": "[HIGH] {{rule.name}} on {{context.alerts.0.host.name}}" },
      "group": "default"
    }
  ]
}
```

### 5.3 Indicator match 룰

```json
{
  "type": "threat_match",
  "name": "Outbound to known malicious IP",
  "index": ["logs-endpoint.events.network-*"],
  "query": "destination.ip : *",
  "threat_index": ["logs-ti.malware-*"],
  "threat_query": "threat.indicator.type : ipv4-addr",
  "threat_mapping": [
    {
      "entries": [
        {
          "field": "destination.ip",
          "type": "mapping",
          "value": "threat.indicator.ip"
        }
      ]
    }
  ],
  "severity": "critical",
  "interval": "5m"
}
```

### 5.4 알람 → 액션

룰이 매치되면 `.alerts-security.alerts-default` 인덱스에 알람 문서가 들어가고, 액션(Slack, PagerDuty, Email, Webhook 등) 실행.

---

## 6. 분석 쿼리 패턴

### 6.1 호스트별 알람 빈도 (마지막 24시간)

```json
GET /.alerts-security.alerts-default/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-24h" } } },
        { "terms": { "kibana.alert.severity": ["high", "critical"] } }
      ]
    }
  },
  "aggs": {
    "by_host": {
      "terms": { "field": "host.name", "size": 10 },
      "aggs": {
        "by_rule": {
          "terms": { "field": "kibana.alert.rule.name", "size": 5 }
        }
      }
    }
  }
}
```

### 6.2 새로 등장한 user-agent (new_terms)

```json
GET /logs-endpoint.events.network-*/_search
{
  "size": 0,
  "aggs": {
    "rare_ua": {
      "rare_terms": {
        "field": "user_agent.original",
        "max_doc_count": 3
      }
    }
  }
}
```

### 6.3 권한 상승 흐름

```json
GET /logs-endpoint.events.*/_eql/search
{
  "query": """
    sequence by host.name, process.entity_id with maxspan=1m
      [process where event.type == "start" and user.name != "SYSTEM"]
      [process where event.type == "start" and user.name == "SYSTEM"
        and process.parent.entity_id == process.entity_id]
  """
}
```

`process.entity_id`로 프로세스 트리를 join 하는 것이 핵심.

---

## 7. 운영 팁

### 7.1 이벤트 표준화의 중요성

같은 윈도우 process 시작 이벤트라도 수집 도구마다 필드가 다르다:

```
Sysmon EventID 1:    Image, ParentImage, CommandLine, ProcessGuid
Windows Security 4688: NewProcessName, ParentProcessName, ProcessCommandLine
Elastic Agent:        process.name, process.parent.name, process.command_line, process.entity_id
```

ECS로 정규화되어야 단일 EQL 룰로 모든 출처를 커버할 수 있다. 직접 수집 시 ingest pipeline에서 반드시 ECS로 매핑.

### 7.2 룰 튜닝 (false positive 줄이기)

```eql
process where event.type == "start"
  and process.name == "powershell.exe"
  and not process.parent.name in ("explorer.exe", "cmd.exe", "code.exe")
  and not user.name == "SYSTEM"
```

조직 환경에 맞게 화이트리스트 추가. 최초 며칠은 dry-run으로 두고 알람 양 확인.

### 7.3 운영 체크리스트

```bash
# 활성 룰 수
GET /.alerts-security.alerts-default/_search
{ "size": 0, "track_total_hits": true,
  "query": { "range": { "@timestamp": { "gte": "now-7d" } } } }

# 룰 실행 상태 (Kibana API)
GET /api/detection_engine/rules/_find?per_page=100

# 룰 별 last_failure_at 확인 → 0이 아니면 트러블슈팅
```

### 7.4 인덱스 보존 정책

```
원본 보안 이벤트:  90~180일 (감사 요건)
알람 (.alerts-security.alerts-*): 1년 이상 (조사 트레일)
TI 인덱스: 30~60일 (오래된 IOC는 제거하여 노이즈 줄이기)
```

---

## 8. 안티패턴

### 8.1 raw 이벤트로 룰 작성

`*powershell*`처럼 wildcard만 잔뜩 → false positive 폭주. ECS의 `process.name == "powershell.exe"`를 쓸 것.

### 8.2 하나의 거대한 룰

10가지 시나리오를 한 룰에 OR로 묶지 말 것. 각 시나리오는 별도 룰 → 실패/튜닝/MITRE 매핑이 독립적.

### 8.3 host.name 없이 sequence

`sequence by` 없이 sequence를 쓰면 호스트 간 이벤트가 섞여 매칭됨. 거의 항상 `by host.name`이 필요하다.

---

## 9. 요약

| 항목 | 선택 |
|------|------|
| 데이터 모델 | ECS (`event.category/type/action` 3종) |
| 수집 | Elastic Agent + Endpoint Security |
| TI 보관 | `logs-ti.*` data stream |
| 시퀀스 매칭 | EQL (`sequence by ... with maxspan=...`) |
| TI 매칭 룰 | `threat_match` rule |
| 프로세스 트리 join | `process.entity_id` / `process.parent.entity_id` |
| 알람 저장소 | `.alerts-security.alerts-*` |
| 표준화 | ingest pipeline에서 ECS로 정규화 (필수) |
