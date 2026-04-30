# 8장. Aggregations — Metric, Bucket, Pipeline과 정확도

Aggregation은 ES의 분석 엔진이다. 검색이 "어느 문서가 매칭되는가"를 묻는다면, aggregation은 "이 문서들의 분포·합계·추세는 어떠한가"를 답한다. Kibana 대시보드의 거의 모든 시각화가 aggregation 위에 만들어진다.

이 장은 aggregation의 종류, 트리 구조, 그리고 가장 자주 잘못 이해되는 **정확도 문제**를 다룬다.

공식 문서: [Aggregations](https://www.elastic.co/docs/reference/aggregations)

---

## 8.1 집계의 세 종류

| 종류 | 산출물 | 예 |
|------|--------|------|
| **Metric** | 숫자(또는 통계 객체) | avg, sum, cardinality, percentiles |
| **Bucket** | 문서들의 그룹(버킷) | terms, range, date_histogram |
| **Pipeline** | 다른 aggregation의 출력을 입력으로 받음 | derivative, moving_avg, bucket_script |

```
ASCII: Aggregation 트리
─────────────────────────────────────────────────────
  Search request
       │
       ▼
   ┌─────────────┐
   │ aggs        │
   └─────────────┘
       │
       ├── bucket: by_category (terms)
       │       │
       │       ├── metric:  avg_price
       │       ├── metric:  total_revenue
       │       │
       │       └── bucket: by_month (date_histogram)
       │               │
       │               ├── metric:   monthly_sum
       │               └── pipeline: 7d_moving_avg
       │
       └── metric: total_unique_users (cardinality)
─────────────────────────────────────────────────────
```

bucket aggregation 안에 metric/bucket/pipeline을 자식으로 둘 수 있고, 이 트리가 곧 분석 결과의 구조가 된다.

---

## 8.2 Metric Aggregations

### 단일 값 metric

```json
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "avg_price":     { "avg":   { "field": "price" } },
    "total_revenue": { "sum":   { "field": "price" } },
    "min_price":     { "min":   { "field": "price" } },
    "max_price":     { "max":   { "field": "price" } },
    "order_count":   { "value_count": { "field": "order_id" } }
  }
}
```

`size: 0`은 "검색 hits는 필요 없다, aggregation만"이라는 뜻. 응답 페이로드 크기를 크게 줄인다.

### stats / extended_stats — 한 번에 여러 통계

```json
"price_stats": { "stats": { "field": "price" } }
// → count, min, max, avg, sum 한 번에

"price_extended": { "extended_stats": { "field": "price" } }
// → 위 + sum_of_squares, variance, std_deviation, std_deviation_bounds
```

### cardinality — 고유 값 개수 (HLL)

```json
"unique_users": { "cardinality": { "field": "user_id" } }
```

**주의**: 정확한 개수가 아니라 **HyperLogLog++** 기반 근사값.

- `precision_threshold` (기본 3000, 최대 40000): 이 값 이하에서는 정확. 그 이상은 약 1~2% 오차.
- 정확한 distinct count가 꼭 필요하면 ES는 적합한 도구가 아니다 (ClickHouse `uniqExact` 같은 거).

```json
"unique_users": {
  "cardinality": {
    "field": "user_id",
    "precision_threshold": 40000
  }
}
```

precision_threshold를 키우면 정확도↑, 메모리↑ (대략 `precision_threshold * 8` 바이트).

### percentiles — TDigest / HDR

```json
"latency_percentiles": {
  "percentiles": {
    "field": "response_ms",
    "percents": [50, 90, 95, 99, 99.9]
  }
}
```

기본 알고리즘은 **TDigest** (메모리 효율, 극단값 정확). HDR Histogram도 선택 가능 (`"hdr": { "number_of_significant_value_digits": 3 }`).

`percentile_ranks`는 반대 — "값이 X 이하인 비율은?"

### top_hits — 버킷 내 대표 문서

```json
"by_category": {
  "terms": { "field": "category" },
  "aggs": {
    "top_in_category": {
      "top_hits": {
        "size": 3,
        "_source": ["name", "price"],
        "sort": [{ "popularity": "desc" }]
      }
    }
  }
}
```

각 카테고리 버킷 안에 인기 상위 3개 상품 미리보기. SQL의 `ROW_NUMBER() OVER (PARTITION BY ...)`에 가장 가깝다.

---

## 8.3 Bucket Aggregations

### terms — 그룹별 집계

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_brand": {
      "terms": {
        "field": "brand",
        "size":  10,
        "order": { "_count": "desc" }
      }
    }
  }
}
```

`size`: 반환할 버킷 수. 기본 10.

**주의**: `terms`는 **정확하지 않을 수 있다**. 8.6에서 상세히 다룬다.

```json
// 정렬은 자식 metric으로도 가능
"by_brand": {
  "terms": {
    "field": "brand",
    "size":  10,
    "order": { "total_revenue": "desc" }
  },
  "aggs": {
    "total_revenue": { "sum": { "field": "price" } }
  }
}
```

### range — 사용자 정의 구간

```json
"by_price_range": {
  "range": {
    "field": "price",
    "ranges": [
      { "to":   100000 },
      { "from": 100000, "to": 500000 },
      { "from": 500000, "to": 2000000 },
      { "from": 2000000 }
    ]
  }
}
```

### date_histogram — 시계열의 핵심

```json
"sales_over_time": {
  "date_histogram": {
    "field": "ordered_at",
    "calendar_interval": "1d",
    "time_zone": "Asia/Seoul",
    "min_doc_count": 0,
    "extended_bounds": {
      "min": "2025-01-01",
      "max": "2025-04-30"
    }
  }
}
```

**`calendar_interval` vs `fixed_interval`**:

| | calendar_interval | fixed_interval |
|---|---|---|
| 단위 | `1m`, `1h`, `1d`, `1w`, `1M`, `1q`, `1y` | `1ms`, `1s`, `1m`, `1h`, `1d` (요일/월 단위 불가) |
| DST/윤달 | 캘린더에 맞춤 (월말까지 등) | 절대 시간 길이 |
| 권장 | 사용자에게 보여줄 일/주/월 차트 | 수치적으로 균등한 시간 윈도우 |

`time_zone`을 지정해야 "한국 기준 일별"이 의도대로 나온다. 기본 UTC.

`min_doc_count: 0`과 `extended_bounds`는 **데이터 없는 날도 0으로 표시**해 그래프 끊김 방지.

### histogram — 숫자 균등 구간

```json
"price_histogram": {
  "histogram": { "field": "price", "interval": 100000 }
}
```

### filters — 명명된 필터들

```json
"by_status": {
  "filters": {
    "filters": {
      "shipped":    { "term": { "status": "shipped" } },
      "delivered":  { "term": { "status": "delivered" } },
      "returned":   { "term": { "status": "returned" } }
    }
  }
}
```

`terms`와 다른 점: 버킷 키와 조건을 **명시적으로** 정의. 이름·정의가 미리 알려진 그룹에 적합.

### nested — nested 필드 집계

```json
"items_agg": {
  "nested": { "path": "items" },
  "aggs": {
    "by_sku": { "terms": { "field": "items.sku" } }
  }
}
```

5장의 nested 필드를 집계하려면 반드시 `nested` aggregation으로 진입 후 자식에서 일반 aggregation 사용.

### composite — 버킷의 페이지네이션

```json
"category_brand": {
  "composite": {
    "size": 1000,
    "sources": [
      { "category": { "terms": { "field": "category" } } },
      { "brand":    { "terms": { "field": "brand" } } }
    ]
  }
}
```

`terms`는 모든 버킷을 한 번에 반환하지만, `composite`는 **after_key로 페이지네이션** 가능. 수십만 버킷을 처리할 때 사용.

```json
// 다음 페이지
"composite": {
  "size": 1000,
  "sources": [...],
  "after": { "category": "laptop", "brand": "samsung" }
}
```

---

## 8.4 Pipeline Aggregations

다른 aggregation의 결과를 입력으로 받는 메타 aggregation. 시계열 분석에서 특히 유용.

### derivative — 차분

```json
"sales_over_time": {
  "date_histogram": { "field": "ordered_at", "calendar_interval": "1d" },
  "aggs": {
    "daily_revenue": { "sum": { "field": "price" } },
    "revenue_diff":  { "derivative": { "buckets_path": "daily_revenue" } }
  }
}
```

전일 대비 증감.

### cumulative_sum — 누적 합

```json
"cumulative_revenue": { "cumulative_sum": { "buckets_path": "daily_revenue" } }
```

### moving_fn — 이동 평균/합/최대 등

`moving_avg`는 deprecated. 대체로 **`moving_fn`** 사용:

```json
"7d_moving_avg": {
  "moving_fn": {
    "buckets_path": "daily_revenue",
    "window": 7,
    "script": "MovingFunctions.unweightedAvg(values)"
  }
}
```

### bucket_script — 산술 조합

```json
"profit_margin": {
  "bucket_script": {
    "buckets_path": {
      "rev":  "revenue",
      "cost": "cost"
    },
    "script": "params.rev > 0 ? (params.rev - params.cost) / params.rev : 0"
  }
}
```

다른 버킷 metric들의 조합으로 새 metric 생성.

---

## 8.5 Composite Aggregation = 버킷의 페이지네이션

`terms`로 큰 카디널리티 필드를 집계하면 메모리 폭발. 이때 `composite`이 답.

```
terms (size=10000):
  모든 shard에서 상위 N개 모음 → coordinating에서 합산 → 상위 10000개 반환
  메모리 부담 큼. 결과 정확도도 의심.

composite (size=1000, after_key 사용):
  정렬된 버킷을 페이지 단위로 반환
  → 모든 버킷을 결정적으로 순회 가능
  → ETL/덤프 용도 적합
```

```json
// 페이지 1
{
  "size": 0,
  "aggs": {
    "all_categories": {
      "composite": {
        "size": 1000,
        "sources": [{ "category": { "terms": { "field": "category" } } }]
      },
      "aggs": {
        "rev": { "sum": { "field": "price" } }
      }
    }
  }
}

// 응답에 after_key가 있으면 다음 페이지 호출
{
  "all_categories": {
    "composite": {
      "size": 1000,
      "sources": [...],
      "after": { "category": "wear" }
    }
  }
}
```

**제약**:
- `composite` 안에서는 `order`를 자식 metric으로 못 잡음 (sources 순서 고정)
- 한 요청에 하나의 composite만

---

## 8.6 terms Aggregation의 정확도 문제

> "the doc_count values for a terms aggregation may be approximate"  
> — [Terms aggregation, Elastic Docs](https://www.elastic.co/docs/reference/aggregations/search-aggregations-bucket-terms-aggregation)

ES의 가장 자주 잘못 이해되는 부분이다.

### 왜 부정확한가

데이터는 여러 shard에 분산돼 있다. `terms` 집계는:

1. 각 shard에서 상위 `shard_size`개 버킷을 추려 coordinating node로 보냄
2. coordinating에서 이를 합산해 최종 상위 `size`개를 결정

**문제**: 어떤 term이 shard A에서는 11위(잘림), shard B에서는 9위(포함)이라면, shard A의 카운트가 누락되어 최종 카운트가 실제보다 낮다.

```
ASCII: terms 부정확의 시나리오
─────────────────────────────────────────────────────────
  size=5, shard_size=10, shards=2

  Shard A 상위 10개 (term X는 11위, 잘림):
    [a:100, b:90, c:80, d:70, e:60, f:50, g:40, h:30, i:20, j:15]
    → X: 14 (전송 안 됨!)

  Shard B 상위 10개 (term X는 5위, 포함):
    [k:200, l:150, m:120, n:100, X:90, ... ]

  Coordinating: 두 shard 결과 합산
    → X 최종 카운트: 90 (실제 90+14=104였어야 함)
─────────────────────────────────────────────────────────
```

### 대응

```json
"by_brand": {
  "terms": {
    "field":     "brand",
    "size":      10,
    "shard_size": 100,                  // size * 1.5 + 10이 기본. 키우면 정확도↑
    "show_term_doc_count_error": true   // 오차 상한 응답에 포함
  }
}
```

응답:

```json
"by_brand": {
  "doc_count_error_upper_bound": 14,   // 가장 큰 잠재 오차
  "sum_other_doc_count": 5320,          // 상위 N개 외 나머지 카운트 합
  "buckets": [
    {
      "key": "samsung",
      "doc_count": 1200,
      "doc_count_error_upper_bound": 0   // 이 버킷의 잠재 오차
    },
    ...
  ]
}
```

`doc_count_error_upper_bound`가 작으면 신뢰 가능. 크면 `shard_size`를 더 키우거나, 정확도가 진짜 필요한 경우 **composite**으로 모든 버킷을 순회.

### 운영 가이드

- 시각화/대시보드용: 기본값으로 충분 (Top 10 보여주기)
- 보고서/회계: composite 또는 단일 shard로 reindex
- 카디널리티가 높을수록(수만 이상의 고유값) 부정확 위험 증가

공식 문서: [Document counts are approximate](https://www.elastic.co/docs/reference/aggregations/search-aggregations-bucket-terms-aggregation#search-aggregations-bucket-terms-aggregation-approximate-counts)

---

## 8.7 Sub-aggregation (트리)

bucket aggregation 안에 자식을 넣어 트리를 만든다.

```json
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 10 },
      "aggs": {
        "revenue":      { "sum": { "field": "price" } },
        "unique_users": { "cardinality": { "field": "user_id" } },
        "by_month": {
          "date_histogram": {
            "field": "ordered_at",
            "calendar_interval": "1M"
          },
          "aggs": {
            "monthly_revenue": { "sum": { "field": "price" } },
            "moving_3m_avg": {
              "moving_fn": {
                "buckets_path": "monthly_revenue",
                "window": 3,
                "script": "MovingFunctions.unweightedAvg(values)"
              }
            }
          }
        }
      }
    }
  }
}
```

해석: 카테고리별로 [총매출, 고유사용자, 월별 매출 + 3개월 이동평균]을 본다.

---

## 8.8 실전: 일별 매출 + 7일 이동평균

```json
GET /sales/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "status": "completed" } },
        { "range": { "ordered_at": { "gte": "now-90d/d" } } }
      ]
    }
  },
  "aggs": {
    "daily_sales": {
      "date_histogram": {
        "field": "ordered_at",
        "calendar_interval": "1d",
        "time_zone": "Asia/Seoul",
        "min_doc_count": 0,
        "extended_bounds": {
          "min": "now-90d/d",
          "max": "now/d"
        }
      },
      "aggs": {
        "revenue":       { "sum": { "field": "price" } },
        "order_count":   { "value_count": { "field": "order_id" } },
        "avg_basket":    { "avg": { "field": "price" } },
        "7d_moving_avg": {
          "moving_fn": {
            "buckets_path": "revenue",
            "window": 7,
            "script": "MovingFunctions.unweightedAvg(values)"
          }
        },
        "wow_growth": {
          "moving_fn": {
            "buckets_path": "revenue",
            "window": 14,
            "script": """
              if (values.length < 14) return null;
              double thisWeek = 0; double lastWeek = 0;
              for (int i=0;i<7;i++) lastWeek += values[i];
              for (int i=7;i<14;i++) thisWeek += values[i];
              return lastWeek > 0 ? (thisWeek - lastWeek) / lastWeek : null;
            """
          }
        }
      }
    }
  }
}
```

**여기서 일어나는 일**:
- 90일 범위로 필터링 (filter context, 캐시)
- 일별 버킷, 한국 시간대 기준
- 데이터 없는 날도 0으로 표시
- 일별 매출, 주문 수, 평균 객단가
- 7일 이동평균과 전주 대비 증감률 계산

---

## 8.9 실전: Top-N 상품 with 매출

```json
GET /sales/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "status": "completed" } },
        { "range": { "ordered_at": { "gte": "now-30d/d" } } }
      ]
    }
  },
  "aggs": {
    "top_products": {
      "terms": {
        "field": "product_id",
        "size":  20,
        "shard_size": 200,
        "order": { "revenue": "desc" }
      },
      "aggs": {
        "revenue":       { "sum": { "field": "price" } },
        "order_count":   { "value_count": { "field": "order_id" } },
        "unique_buyers": { "cardinality": { "field": "user_id" } },
        "sample_orders": {
          "top_hits": {
            "size": 1,
            "_source": ["product_name", "thumbnail"]
          }
        }
      }
    }
  }
}
```

`top_hits`로 product_name 같은 비집계 필드를 한 번에 가져온다. 별도 lookup 호출 없이.

---

## 8.10 운영 시 주의점

### cardinality는 비싸다

HLL 자료구조를 메모리에 유지해야 한다. `precision_threshold`가 클수록 메모리 비용↑. 여러 cardinality를 한 쿼리에 쓰는 건 신중히.

### terms의 카디널리티 폭발

`terms { "field": "user_id" }`처럼 수백만 고유값 필드에 쓰면 메모리 OOM 위험. composite으로 페이지네이션하거나, 미리 카테고리 같은 저카디널리티 필드로 좁혀라.

### date_histogram의 time_zone 누락

기본 UTC다. KST 기준 일별 그래프인데 자정 직전 주문이 어제로 밀려 보이는 흔한 버그. **항상 `time_zone: "Asia/Seoul"`을 명시**하라.

### track_total_hits: false + size: 0

aggregation만 필요한 경우 거의 항상 다음을 적용:

```json
{
  "size": 0,
  "track_total_hits": false,
  "aggs": { ... }
}
```

hits를 가져오지 않고, 정확한 총 개수도 계산하지 않아 응답이 빠르고 작아진다.

### Runtime field와 aggregation

5장에서 본 runtime field는 쿼리 시 평가된다. aggregation에서 사용하면 **모든 매칭 문서마다 스크립트 평가** → 매우 비쌀 수 있다.

```json
"runtime_mappings": {
  "discount_pct": {
    "type": "double",
    "script": "emit((doc['list_price'].value - doc['price'].value) / doc['list_price'].value)"
  }
},
"aggs": {
  "by_discount": {
    "histogram": { "field": "discount_pct", "interval": 0.1 }
  }
}
```

자주 쓰는 runtime field는 reindex로 정식 필드로 승격하라. aggregation 빈도가 높을수록 더더욱.

### keyword 필드 doc_values

terms aggregation은 doc_values를 사용한다. `doc_values: false`로 끈 keyword 필드는 집계 불가. 5장 참조.

### Profile API로 비용 분석

```json
{ "profile": true, "size": 0, "aggs": { ... } }
```

각 aggregation 단계의 시간이 응답에 포함된다. "왜 이렇게 느리지?"의 첫 답.

---

## 8.11 Key Takeaways

| 항목 | 핵심 |
|------|------|
| **세 가지 종류** | metric / bucket / pipeline |
| **트리 구조** | bucket 안에 metric/bucket/pipeline 자식 |
| **cardinality** | HLL 근사. precision_threshold가 정확도-메모리 트레이드오프 |
| **percentiles** | TDigest 기반. 정확한 분위수가 아닌 근사 |
| **terms 부정확** | shard_size로 완화, show_term_doc_count_error로 측정 |
| **composite** | 버킷의 페이지네이션. ETL·덤프에 적합 |
| **date_histogram** | calendar_interval과 time_zone을 잊지 마라 |
| **pipeline** | derivative, cumulative_sum, moving_fn, bucket_script로 시계열 분석 |
| **size: 0 + track_total_hits: false** | aggregation 전용 쿼리의 표준 헤더 |

---

*다음 장: 9장. Indexing 전략 — Bulk API, Refresh, Translog, ILM, Data Stream*
