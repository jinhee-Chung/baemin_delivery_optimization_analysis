# 배달의민족 분석 프로젝트 실행 로드맵
**v4.0 기준 | 실제 구현 참고용**

---

## 로드맵 개요

```
Phase 0  환경 세팅              (1~2일)
Phase 1  외부 데이터 수집       (2~3일)
Phase 2  스키마 설계 & DDL      (2~3일)
Phase 3  시뮬레이션 데이터 생성 (3~5일)
Phase 4  선행검증 EDA           (2~3일)
Phase 5  마트 ETL & Airflow     (3~5일)
Phase 6  KPI 쿼리 & 가설 검증  (3~5일)
Phase 7  시각화 & 대시보드      (3~5일)
Phase 8  포트폴리오 정리        (3~5일)
                        총 22~36일
```

---

## 계획서 참고 방법 (핵심 원칙)

> 계획서(v4.0)는 **"무엇을 만들어야 하는가"의 지도**다.
> 실제 작업 중 막히면 계획서의 해당 섹션을 다시 읽고,
> 작업하면서 계획서가 현실과 다르면 계획서를 수정한다.
> 계획서는 고정된 정답이 아니라 살아있는 문서다.

| 작업 단계 | 계획서에서 볼 섹션 |
|---------|----------------|
| 스키마 설계할 때 | 5장 OLTP 스키마 + 테이블 관계 |
| 시뮬레이션 파라미터 결정할 때 | 4장 가설 + 날씨 파라미터 표 |
| 마트 SQL 짤 때 | 8장 마트 설계 (Grain, 핵심 공식) |
| KPI 쿼리 짤 때 | 9장 KPI 지표 |
| 막힐 때 | 13장 논리 검토 (이미 발견된 결함 참고) |

---

## Phase 0. 환경 세팅 (1~2일)

### 목표
로컬에서 PostgreSQL + Python 컨테이너를 Docker로 띄우고
깃 저장소 구조를 세팅한다.

### 순서

**① 깃 저장소 생성**
```bash
mkdir baemin_delivery_optimization_analysis
cd baemin_delivery_optimization_analysis
git init
git checkout -b feature/01-domain-analysis
```

브랜치 구조 (계획서 7장 참고):
```
master
├── feature/01-domain-analysis   ← 지금 여기서 시작
├── feature/02-data-collection
├── feature/03-eda
├── feature/04-modeling
└── feature/05-reporting
```

**② docker-compose.yml 작성**
```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: baemin_oltp
      POSTGRES_USER: analyst
      POSTGRES_PASSWORD: yourpassword
    ports:
      - "5432:5432"
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
      - ./sql:/docker-entrypoint-initdb.d

  python:
    build: .
    volumes:
      - .:/workspace
    depends_on:
      - postgres
```

**③ 폴더 구조**
```
baemin_delivery_optimization_analysis/
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── sql/
│   └── ddl.sql              ← Phase 2에서 작성
├── scripts/
│   └── simulate_data.py     ← Phase 3에서 작성
├── notebooks/
│   ├── eda.ipynb            ← Phase 4
│   └── hypothesis_test.ipynb ← Phase 6
├── mart/
│   ├── mart_kitchen_delay.sql
│   ├── mart_rider_rejection.sql
│   ├── mart_weather_supply_gap.sql
│   └── mart_eta_decomposition.sql
├── airflow/
│   ├── dag_external_collect.py
│   └── dag_mart_etl.py
└── docs/
    └── baemin_plan_v4.md    ← 계획서 사본 보관
```

**④ requirements.txt**
```
psycopg2-binary
pandas
numpy
faker
sqlalchemy
matplotlib
seaborn
folium
apache-airflow
```

**⑤ feature/01 커밋**
```bash
# 계획서 v4.0을 docs/에 복사 후 커밋
git add .
git commit -m "feat: 프로젝트 초기 환경 세팅 + 계획서 v4.0"
```

### 이 단계에서 계획서 활용 포인트
- 7장 브랜치 구조 → 폴더·브랜치 세팅 기준
- 10장 기술 스택 → requirements.txt 항목 결정

---

## Phase 1. 외부 데이터 수집 (2~3일)
`feature/02-data-collection`

### 목표
시뮬레이션 파라미터에 현실 근거를 부여할 외부 데이터를 수집한다.
**이 단계를 먼저 하는 이유:** 파라미터를 근거 없이 설정하면
시뮬레이션 전체가 순환논리가 된다. (계획서 13장 결함 #1 참조)

### 수집 대상 (계획서 2-1 참고)

| 데이터 | 출처 | 활용 목적 |
|-------|------|---------|
| 배달앱 시장 규모 | 공정거래위원회 보고서 | 시장 배경 서술 |
| 플랫폼 점유율 | 모바일인덱스, 와이즈앱 공개 자료 | 배민 시장 지위 |
| 배달 종사자 수 | 통계청 e-나라지표 | 라이더 공급 규모 파라미터 |
| 강수량·풍속 분포 | 기상청 공공데이터포털 | 날씨 등급 기준·수급 파라미터 |

### 기상청 데이터 수집 예시
```python
import requests
import pandas as pd

# 기상청 API (공공데이터포털에서 인증키 발급)
API_KEY = "your_api_key"
url = "http://apis.data.go.kr/1360000/AsosDalyInfoService/getWthrDataList"

params = {
    "serviceKey": API_KEY,
    "pageNo": 1,
    "numOfRows": 365,
    "dataType": "JSON",
    "dataCd": "ASOS",
    "dateCd": "DAY",
    "startDt": "20240101",
    "endDt": "20241231",
    "stnIds": "108"  # 서울
}

response = requests.get(url, params=params)
data = response.json()
df = pd.DataFrame(data['response']['body']['items']['item'])
df.to_csv("data/weather_2024.csv", index=False)
```

### 파라미터 매핑 작업
수집한 데이터로 시뮬레이션 파라미터를 확정한다.

```python
# weather_2024.csv 분석
import pandas as pd

df = pd.read_csv("data/weather_2024.csv")

# 강수량 기준으로 날씨 등급 기준 수치화
print(df['rn'].describe())  # 강수량 분포
# → 0mm: 맑음, 0-5mm: 흐림, 5-30mm: 비, 30mm+: 폭우 기준 확정

# 날씨 등급별 발생 빈도
df['grade'] = pd.cut(df['rn'].fillna(0),
                     bins=[-1, 0, 5, 30, 999],
                     labels=['맑음', '흐림', '비', '폭우'])
print(df['grade'].value_counts())
```

이렇게 확정된 수치가 계획서 4-3절 파라미터 표의 근거가 된다.

```markdown
# 파라미터 확정 기록 (params_log.md)
| 파라미터 | 값 | 근거 출처 |
|---------|---|---------|
| 폭우 기준 강수량 | 30mm/일 이상 | 기상청 2024년 서울 강수 분포 |
| 라이더 공급 배수 (비) | 0.75 | 통계청 배달 종사자 콜 수 계절 변동 |
```

### Airflow DAG-1 작성 (계획서 6장 참고)
```python
# airflow/dag_external_collect.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def fetch_weather():
    # 기상청 API 호출 → CSV 저장 → PostgreSQL 적재
    pass

def fetch_statistics():
    # 통계청 파일 다운로드 → 파라미터 비교 테이블 생성
    pass

with DAG('dag_external_collect',
         start_date=datetime(2026, 1, 1),
         schedule_interval='@once') as dag:

    t1 = PythonOperator(task_id='fetch_weather', python_callable=fetch_weather)
    t2 = PythonOperator(task_id='fetch_statistics', python_callable=fetch_statistics)

    t1 >> t2
```

### 이 단계에서 계획서 활용 포인트
- 2-1절 외부 데이터 매핑 표 → 수집 대상 확인
- 4-3절 날씨 파라미터 표 → 수집 후 수치 채워넣기
- 13장 결함 #5 → 파라미터마다 근거 출처 기록하는 습관

---

## Phase 2. 스키마 설계 & DDL (2~3일)
`feature/02-data-collection`

### 목표
계획서 5장 OLTP 스키마를 실제 DDL SQL로 구현하고
Docker PostgreSQL에 적재한다.

### DDL 작성 순서
계획서 5장 테이블 관계를 보고 의존 순서대로 작성한다.

```
1. weather          (의존 없음)
2. users            (의존 없음)
3. restaurants      (의존 없음)
4. riders           (의존 없음)
5. user_addresses   (← users)
6. menus            (← restaurants)
7. orders           (← users, restaurants, user_addresses)
8. deliveries       (← orders, riders, promotions, weather)
9. order_items      (← orders, menus)
10. payments        (← orders)
11. cs_tickets      (← orders, users)
12. promotions      (의존 없음, deliveries보다 먼저)
```

### DDL 예시 (계획서 5-1 스키마를 SQL로 변환)
```sql
-- sql/ddl.sql

CREATE TABLE weather (
    weather_id        SERIAL PRIMARY KEY,
    recorded_at       TIMESTAMP NOT NULL,
    precipitation     FLOAT,
    wind_speed        FLOAT,
    weather_grade     VARCHAR(10) CHECK (weather_grade IN ('맑음','흐림','비','폭우','눈')),
    is_adverse        BOOLEAN DEFAULT FALSE,
    demand_multiplier FLOAT DEFAULT 1.0,
    supply_multiplier FLOAT DEFAULT 1.0
);

CREATE TABLE users (
    user_id             SERIAL PRIMARY KEY,
    social_account_type VARCHAR(20),
    signup_at           TIMESTAMP NOT NULL,
    nickname            VARCHAR(50),
    phone               VARCHAR(20),
    email               VARCHAR(100),
    gender              VARCHAR(10),
    age_group           VARCHAR(10),
    last_order_at       TIMESTAMP,
    total_order_count   INT DEFAULT 0,
    is_active           BOOLEAN DEFAULT TRUE
);

CREATE TABLE restaurants (
    restaurant_id      SERIAL PRIMARY KEY,
    business_number    VARCHAR(20) UNIQUE,
    registered_at      TIMESTAMP,
    category           VARCHAR(20) CHECK (category IN ('한식','치킨','피자','분식','기타')),
    region             VARCHAR(50),
    lat                FLOAT,
    lng                FLOAT,
    avg_cooking_time   INT,
    max_capacity       INT,
    operation_status   VARCHAR(20) DEFAULT '영업중',
    delivery_radius_km FLOAT,
    min_order_amount   INT,
    is_converted_offline BOOLEAN DEFAULT FALSE,
    converted_at       TIMESTAMP
);

-- ... 나머지 테이블 동일 패턴으로 작성
```

### 체크리스트
```
□ 모든 FK 제약조건 설정됐는가
□ ENUM 값이 계획서 스키마 주석과 일치하는가
□ 날짜 파티셔닝 필요한 테이블 파티션 설정했는가
  (deliveries, orders → 날짜 파티셔닝 계획서 2-4절 참고)
□ Docker 컨테이너에서 DDL 실행 성공했는가
□ 테이블 관계 ERD 그려서 계획서 5장과 비교 검증했는가
```

### 이 단계에서 계획서 활용 포인트
- 5-1절 스키마 전체 → DDL 작성 기준
- 5-1절 테이블 관계 → FK 설정 순서
- 2-4절 DW 분류 → 파티셔닝 전략 결정

---

## Phase 3. 시뮬레이션 데이터 생성 (3~5일)
`feature/02-data-collection`

### 목표
계획서 4장 가설과 Phase 1에서 확정한 파라미터를 기반으로
각 테이블에 현실적인 시뮬레이션 데이터를 생성하고 적재한다.

### 생성 순서 (DDL 의존 순서와 동일)

```python
# scripts/simulate_data.py

import pandas as pd
import numpy as np
from faker import Faker
from sqlalchemy import create_engine

fake = Faker('ko_KR')
engine = create_engine("postgresql://analyst:yourpassword@localhost:5432/baemin_oltp")

# ── 파라미터 (Phase 1에서 확정한 값) ──────────────────
PARAMS = {
    "n_users": 10_000,
    "n_restaurants": 500,
    "n_riders": 1_000,
    "n_orders": 100_000,
    "date_range": ("2024-01-01", "2024-12-31"),

    # 4-1절 Kitchen Delay 파라미터
    "avg_cooking_time": {
        "한식": (25, 35),   # (min, max) 분
        "피자": (20, 30),
        "치킨": (15, 20),
        "분식": (10, 15),
    },

    # 4-3절 날씨 파라미터 (Phase 1 기상청 데이터 기반)
    "weather_grade_dist": {"맑음": 0.55, "흐림": 0.25, "비": 0.15, "폭우": 0.05},
    "demand_multiplier":  {"맑음": 1.0, "흐림": 1.1, "비": 1.3, "폭우": 1.5},
    "supply_multiplier":  {"맑음": 1.0, "흐림": 0.95, "비": 0.75, "폭우": 0.5},

    # 4-2절 콜사 파라미터
    "base_rejection_rate": 0.12,
    "dead_zone_rejection_boost": 0.15,
    "fatigue_threshold": 7,        # N건 이상 연속배달 시 콜사율 급증
    "fatigue_rejection_boost": 0.20,
}
```

### 가설 A: Kitchen Delay 오버수락 반영
```python
def generate_orders(restaurants_df, n=100_000):
    orders = []
    for _ in range(n):
        restaurant = restaurants_df.sample(1).iloc[0]

        # 피크타임 여부 결정 (계획서 4-1절)
        hour = np.random.choice(range(24),
                                p=build_hour_prob())  # 11-12시, 18-19시 가중치 높게

        # 오버수락 시뮬레이션 (가설 A)
        concurrent_orders = np.random.randint(1, restaurant['max_capacity'] * 1.5)
        is_overloaded = concurrent_orders > restaurant['max_capacity']

        base_cook = np.random.uniform(*PARAMS['avg_cooking_time'][restaurant['category']])
        # 오버수락 시 조리시간 1.3~1.8배 증가
        actual_cook = base_cook * (np.random.uniform(1.3, 1.8) if is_overloaded else 1.0)

        order_accepted_at = generate_datetime(hour)
        pickup_ready_at = order_accepted_at + pd.Timedelta(minutes=actual_cook)

        orders.append({
            "restaurant_id": restaurant['restaurant_id'],
            "order_accepted_at": order_accepted_at,
            "pickup_ready_at": pickup_ready_at,
            "concurrent_orders": concurrent_orders,  # 오버수락률 계산용
            # ...
        })
    return pd.DataFrame(orders)
```

### 가설 B: Dead Zone 반영
```python
def generate_deliveries(orders_df, riders_df):
    deliveries = []
    for _, order in orders_df.iterrows():

        # 라이더 선택 (Dead Zone 여부 반영)
        rider = riders_df.sample(1).iloc[0]

        # Dead Zone 판정: 완료 좌표가 수요 밀집지와 멀면 콜사율 상승
        dist_to_demand = haversine(rider['current_lat'], rider['current_lng'],
                                   order['restaurant_lat'], order['restaurant_lng'])
        is_dead_zone = dist_to_demand > 2.0  # 2km 이상이면 Dead Zone

        # 피로도 반영
        is_fatigued = rider['consecutive_deliveries'] >= PARAMS['fatigue_threshold']

        # 콜사율 계산 (가설 A~D 복합)
        rejection_rate = PARAMS['base_rejection_rate']
        if is_dead_zone:
            rejection_rate += PARAMS['dead_zone_rejection_boost']
        if is_fatigued:
            rejection_rate += PARAMS['fatigue_rejection_boost']

        is_rejected = np.random.random() < rejection_rate
        # ...
    return pd.DataFrame(deliveries)
```

### 데이터 규모 기준
```
users:          10,000건
restaurants:       500건
riders:          1,000건
orders:        100,000건  ← 분석 핵심 단위
deliveries:    100,000건
order_items:   250,000건  (주문당 평균 2.5개 메뉴)
payments:      100,000건
cs_tickets:     15,000건  (주문의 15% → CS 접수)
weather:         8,760건  (2024년 시간별)
promotions:        100건
```

### 적재 후 검증
```sql
-- 가설 파라미터가 실제 데이터에 반영됐는지 확인
-- 1. Kitchen Wait Time 분포
SELECT category,
       AVG(EXTRACT(EPOCH FROM (pickup_ready_at - order_accepted_at))/60) AS avg_wait_min
FROM orders o
JOIN restaurants r USING (restaurant_id)
GROUP BY category;
-- → 한식 25-35분, 치킨 15-20분 범위 확인

-- 2. 날씨 등급별 주문량 배수
SELECT w.weather_grade,
       COUNT(*) / AVG(COUNT(*)) OVER () AS demand_multiplier_actual
FROM deliveries d
JOIN weather w ON DATE_TRUNC('hour', d.assigned_at) = w.recorded_at
GROUP BY w.weather_grade;
-- → 파라미터 표 수치와 ±10% 범위 내인지 확인
```

### 이 단계에서 계획서 활용 포인트
- 4장 가설 전체 → 파라미터 설정 기준 (가설마다 파라미터 1개)
- 4-3절 날씨 파라미터 표 → demand_multiplier, supply_multiplier 수치
- 13장 결함 #1 → 순환논리 방지: 라이더 유형별 콜사율 동일하게 설정

---

## Phase 4. 선행검증 EDA (2~3일)
`feature/03-eda`

### 목표
**계획서 8장 마트 4(mart_eta_decomposition) 분석 순서 정당화** 참고:
Kitchen Delay가 ETA의 주된 원인인지 먼저 검증하고
가설 방향이 맞는지 확인한다.

### Step 1: ETA 구성요소 분해 (최우선)

```sql
-- eta_decomposition.sql
SELECT
    -- Kitchen Wait Time
    AVG(EXTRACT(EPOCH FROM (pickup_ready_at - order_accepted_at))/60) AS kitchen_wait_avg,

    -- Pickup Wait Time (픽업 준비 완료 ~ 라이더 실제 픽업)
    AVG(EXTRACT(EPOCH FROM (d.pickup_at - o.pickup_ready_at))/60)     AS pickup_wait_avg,

    -- Last-mile Time
    AVG(EXTRACT(EPOCH FROM (d.completed_at - d.pickup_at))/60)        AS lastmile_avg,

    -- Total ETA
    AVG(d.actual_eta_min)                                              AS total_eta_avg

FROM orders o
JOIN deliveries d USING (order_id)
WHERE d.is_rejected = FALSE;
```

**판단 기준:**
```
Kitchen Wait > 50% of Total ETA → 계획서 방향 유지 (마트 1 우선)
Kitchen Wait < 30% of Total ETA → 분석 메인 타겟 재검토 필요
```

### Step 2: 핵심 분포 탐색 (EDA 노트북)

```python
# notebooks/eda.ipynb

# ── 1. Kitchen Delay 분포 ──
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# 카테고리별 Kitchen Wait Time 박스플롯
df.boxplot('kitchen_wait_min', by='category', ax=axes[0])
axes[0].set_title('카테고리별 Kitchen Wait Time')

# 시간대별 Slot Congestion Index 히트맵
pivot = df.pivot_table('slot_congestion', index='restaurant_id', columns='hour', aggfunc='mean')
sns.heatmap(pivot.head(50), ax=axes[1], cmap='YlOrRd')
axes[1].set_title('식당별 시간대별 혼잡도')

# ── 2. 콜사율 분포 ──
# Dead Zone 여부별 콜사율
df.groupby('is_dead_zone')['is_rejected'].mean().plot(kind='bar')

# 연속배달 수 vs 콜사율
df.groupby('consecutive_deliveries_band')['is_rejected'].mean().plot()

# ── 3. 날씨 등급별 수급 Gap ──
df.groupby('weather_grade')[['demand_multiplier', 'supply_multiplier']].mean()
```

### Step 3: EDA 결과로 계획서 업데이트

EDA 후 아래 내용을 계획서에 반영:
```markdown
# EDA 결과 요약 (계획서 3장 논리보완에 추가)
- Kitchen Wait Time: Total ETA의 XX% 차지 (예상 범위 충족/미충족)
- 콜사율 가장 높은 요인: Dead Zone XX% / 피로도 XX% / 수수료 XX%
- → 4-2절 가설 우선순위: A > D > C > B 순으로 심층 분석
```

### 이 단계에서 계획서 활용 포인트
- 8장 마트 4 분석 순서 정당화 → EDA 판단 기준
- 4-2절 콜사 가설 A~D → 요인별 분석 방향
- 13장 결함 #2 (검증 순서 역전) → EDA에서 순서 맞게 진행

---

## Phase 5. 마트 ETL & Airflow (3~5일)
`feature/04-modeling`

### 목표
계획서 8장 마트 설계 기반으로 4개 마트 SQL을 작성하고
Airflow DAG-2로 의존 순서를 보장해 적재한다.

### 마트 생성 순서 (DAG-2 의존 순서)
```
mart_kitchen_delay      → 먼저
mart_rider_rejection    → 먼저
mart_weather_supply_gap → 먼저
mart_eta_decomposition  → 마지막 (위 3개 완료 후)
```

### 마트 SQL (계획서 8장 핵심 공식 그대로 구현)

```sql
-- mart/mart_kitchen_delay.sql
CREATE TABLE mart_kitchen_delay AS
SELECT
    r.restaurant_id,
    r.category,
    r.region,
    DATE_TRUNC('hour', o.order_accepted_at)                        AS time_slot,
    EXTRACT(HOUR FROM o.order_accepted_at)                         AS hour_of_day,

    -- Kitchen Wait Time (계획서 8장 핵심 공식)
    AVG(EXTRACT(EPOCH FROM (o.pickup_ready_at - o.order_accepted_at))/60) AS kitchen_wait_min,

    -- Slot Congestion Index (가설 B)
    COUNT(*)                                                        AS slot_order_count,

    -- 오버수락률 (가설 A) = 동시 진행 주문 수 / max_capacity
    AVG(o.concurrent_orders::FLOAT / r.max_capacity)               AS over_accept_rate,

    COUNT(*) FILTER (WHERE o.concurrent_orders > r.max_capacity)
        / COUNT(*)::FLOAT                                           AS over_accept_flag_rate

FROM orders o
JOIN restaurants r USING (restaurant_id)
WHERE o.status NOT IN ('cancelled')
GROUP BY r.restaurant_id, r.category, r.region,
         DATE_TRUNC('hour', o.order_accepted_at),
         EXTRACT(HOUR FROM o.order_accepted_at);
```

```sql
-- mart/mart_rider_rejection.sql
CREATE TABLE mart_rider_rejection AS
SELECT
    d.rider_id,
    ri.rider_type,
    ri.vehicle_type,
    d.is_rejected,

    -- Dead Zone 판정 (계획서 8장 마트 2 Dead Zone 분석 데이터 소스)
    CASE WHEN ST_Distance(
        ST_MakePoint(d.dropoff_lng, d.dropoff_lat)::geography,
        ST_MakePoint(nearest_demand.lng, nearest_demand.lat)::geography
    ) > 2000 THEN TRUE ELSE FALSE END                              AS is_dead_zone,

    ri.consecutive_deliveries,
    EXTRACT(EPOCH FROM (ri.call_informed_at - d.completed_at))/60  AS gap_to_next_call_min,
    d.commission_amount,
    d.commission_rate,
    d.distance_km,
    w.weather_grade,
    d.assigned_at::DATE                                             AS delivery_date

FROM deliveries d
JOIN riders ri USING (rider_id)
LEFT JOIN weather w ON DATE_TRUNC('hour', d.assigned_at) = w.recorded_at
CROSS JOIN LATERAL (
    SELECT ua.lat, ua.lng
    FROM user_addresses ua
    ORDER BY ST_Distance(
        ST_MakePoint(d.dropoff_lng, d.dropoff_lat),
        ST_MakePoint(ua.lng, ua.lat)
    )
    LIMIT 1
) nearest_demand;
```

```sql
-- mart/mart_weather_supply_gap.sql
CREATE TABLE mart_weather_supply_gap AS
WITH base AS (
    SELECT
        DATE_TRUNC('hour', d.assigned_at)                          AS time_slot,
        w.weather_grade,
        r.region,
        COUNT(DISTINCT d.order_id)                                 AS actual_orders,
        COUNT(DISTINCT d.rider_id) FILTER (WHERE NOT d.is_rejected) AS active_riders,
        AVG(d.is_rejected::INT)                                    AS rejection_rate,
        p.incentive_amount
    FROM deliveries d
    JOIN weather w ON DATE_TRUNC('hour', d.assigned_at) = w.recorded_at
    JOIN orders o USING (order_id)
    JOIN restaurants r USING (restaurant_id)
    LEFT JOIN promotions p ON d.promotion_id = p.promotion_id
    GROUP BY DATE_TRUNC('hour', d.assigned_at), w.weather_grade, r.region, p.incentive_amount
),
normal AS (
    SELECT region,
           AVG(actual_orders) AS avg_normal_orders,
           AVG(active_riders) AS avg_normal_riders
    FROM base WHERE weather_grade = '맑음'
    GROUP BY region
)
SELECT
    b.*,
    -- 수급 Gap 공식 (계획서 8장 마트 3 핵심 공식)
    (b.actual_orders::FLOAT / n.avg_normal_orders)
    - (b.active_riders::FLOAT / n.avg_normal_riders) AS supply_gap
FROM base b
JOIN normal n USING (region);
```

```sql
-- mart/mart_eta_decomposition.sql
CREATE TABLE mart_eta_decomposition AS
SELECT
    o.order_id,
    o.restaurant_id,
    r.category,
    d.rider_id,
    ri.rider_type,
    w.weather_grade,

    -- ETA 분해 (계획서 8장 마트 4 공식)
    EXTRACT(EPOCH FROM (o.pickup_ready_at - o.order_accepted_at))/60 AS kitchen_wait_min,
    EXTRACT(EPOCH FROM (d.pickup_at - o.pickup_ready_at))/60         AS pickup_wait_min,
    EXTRACT(EPOCH FROM (d.completed_at - d.pickup_at))/60            AS lastmile_min,
    d.actual_eta_min,
    d.expected_eta_min,
    d.eta_gap_min,   -- KPI #9: ETA 지연율의 원천

    mk.kitchen_wait_min   AS mart_kitchen_wait,
    mk.over_accept_rate,
    mw.supply_gap,
    mr.is_dead_zone,
    mr.rejection_rate

FROM orders o
JOIN deliveries d USING (order_id)
JOIN restaurants r USING (restaurant_id)
JOIN riders ri ON d.rider_id = ri.rider_id
LEFT JOIN weather w ON DATE_TRUNC('hour', d.assigned_at) = w.recorded_at
LEFT JOIN mart_kitchen_delay mk
    ON mk.restaurant_id = o.restaurant_id
    AND mk.time_slot = DATE_TRUNC('hour', o.order_accepted_at)
LEFT JOIN mart_weather_supply_gap mw
    ON mw.time_slot = DATE_TRUNC('hour', d.assigned_at)
    AND mw.region = r.region
LEFT JOIN (
    SELECT rider_id, AVG(is_rejected::INT) AS rejection_rate, BOOL_OR(is_dead_zone) AS is_dead_zone
    FROM mart_rider_rejection
    GROUP BY rider_id
) mr ON mr.rider_id = d.rider_id;
```

### Airflow DAG-2 작성
```python
# airflow/dag_mart_etl.py
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime

with DAG('dag_mart_etl', start_date=datetime(2026, 1, 1), schedule_interval='@once') as dag:

    t1 = PostgresOperator(task_id='mart_kitchen_delay',
                          sql='mart/mart_kitchen_delay.sql')
    t2 = PostgresOperator(task_id='mart_rider_rejection',
                          sql='mart/mart_rider_rejection.sql')
    t3 = PostgresOperator(task_id='mart_weather_supply_gap',
                          sql='mart/mart_weather_supply_gap.sql')
    t4 = PostgresOperator(task_id='mart_eta_decomposition',
                          sql='mart/mart_eta_decomposition.sql')

    # 계획서 6장 DAG-2 의존 순서
    [t1, t2, t3] >> t4
```

### 이 단계에서 계획서 활용 포인트
- 8장 각 마트 Grain·핵심 공식 → SQL GROUP BY, 계산식 기준
- 8장 마트 4 ETA 분해 공식 → SELECT 절 컬럼 구성
- 6장 DAG-2 태스크 구성 → Airflow 의존 순서

---

## Phase 6. KPI 쿼리 & 가설 검증 (3~5일)
`feature/04-modeling`

### 목표
계획서 9장 KPI 9개를 쿼리로 구현하고
4장 가설 A~D를 데이터로 검증한다.

### KPI 쿼리 (계획서 9장 순서대로)

```sql
-- KPI 1: Kitchen Wait Time
SELECT category, AVG(kitchen_wait_min) AS kpi_kitchen_wait
FROM mart_kitchen_delay
GROUP BY category ORDER BY 2 DESC;

-- KPI 2: Slot Congestion Index
SELECT hour_of_day, category, AVG(slot_order_count) AS kpi_slot_congestion
FROM mart_kitchen_delay
GROUP BY hour_of_day, category;

-- KPI 3: 오버수락률
SELECT category, AVG(over_accept_rate) AS kpi_over_accept
FROM mart_kitchen_delay
GROUP BY category;

-- KPI 5: 콜사율
SELECT rider_type, weather_grade,
       AVG(is_rejected::INT) AS kpi_rejection_rate
FROM mart_rider_rejection
GROUP BY rider_type, weather_grade;

-- KPI 6: Dead Zone 발생률
SELECT region,
       AVG(is_dead_zone::INT) AS kpi_dead_zone_rate
FROM mart_rider_rejection mr
JOIN restaurants r USING (restaurant_id)
GROUP BY region;

-- KPI 7: 수급 Gap
SELECT weather_grade, region, AVG(supply_gap) AS kpi_supply_gap
FROM mart_weather_supply_gap
GROUP BY weather_grade, region;

-- KPI 9: ETA 지연율 (가장 중요한 결과 지표)
SELECT
    COUNT(*) FILTER (WHERE eta_gap_min > 0)::FLOAT / COUNT(*) AS kpi_eta_delay_rate,
    AVG(eta_gap_min) FILTER (WHERE eta_gap_min > 0)           AS avg_delay_min
FROM mart_eta_decomposition;
```

### 가설 검증 노트북 구조

```python
# notebooks/hypothesis_test.ipynb

# ── 가설 A: 오버수락이 Kitchen Delay를 유발하는가 ──
df = pd.read_sql("SELECT * FROM mart_kitchen_delay", engine)

# 오버수락 여부별 Kitchen Wait Time 비교
over = df[df['over_accept_flag_rate'] > 0.3]['kitchen_wait_min']
normal = df[df['over_accept_flag_rate'] <= 0.3]['kitchen_wait_min']

from scipy import stats
t_stat, p_val = stats.ttest_ind(over, normal)
print(f"가설 A 검증: t={t_stat:.2f}, p={p_val:.4f}")
print(f"오버수락 시 평균 Kitchen Wait: {over.mean():.1f}분")
print(f"정상 수락 시 평균 Kitchen Wait: {normal.mean():.1f}분")
# → p < 0.05이면 "만약 실데이터에서 이 패턴이 발견된다면 통계적으로 유의미"

# ── 가설 B: Dead Zone이 콜사율을 높이는가 ──
df2 = pd.read_sql("SELECT * FROM mart_rider_rejection", engine)
dead_zone_rate = df2.groupby('is_dead_zone')['is_rejected'].mean()
print(f"\n가설 A 검증 - Dead Zone별 콜사율:")
print(dead_zone_rate)

# ── 가설 C: 수수료가 낮을수록 콜사율이 높은가 ──
df2['commission_band'] = pd.cut(df2['commission_amount'],
                                 bins=[0, 3000, 4000, 5000, 99999],
                                 labels=['~3천', '3-4천', '4-5천', '5천+'])
print(f"\n가설 C 검증 - 수수료 구간별 콜사율:")
print(df2.groupby('commission_band')['is_rejected'].mean())

# ── 가설 D: 연속배달 피로도가 콜사율을 높이는가 ──
df2['fatigue_band'] = pd.cut(df2['consecutive_deliveries'],
                              bins=[0, 3, 6, 9, 99],
                              labels=['1-3건', '4-6건', '7-9건', '10건+'])
print(f"\n가설 D 검증 - 연속배달 수별 콜사율:")
print(df2.groupby('fatigue_band')['is_rejected'].mean())
```

### 인사이트 작성 원칙
```markdown
# 인사이트 작성 템플릿 (계획서 전제 명시 준수)

## 가설 A: 오버수락과 Kitchen Delay

**분석 결과:**
오버수락 식당(동시 주문 / max_capacity > 1.0)의 평균 Kitchen Wait Time은 XX분으로,
정상 수락 식당 XX분 대비 XX% 높게 나타났다.
한식 카테고리에서 오버수락 발생률이 XX%로 가장 높았다.

**인사이트 (만약 이런 패턴이 실데이터에서 발견된다면):**
한식·피자 카테고리의 오버수락 식당에 실시간 경고 시스템을 도입하면
Kitchen Wait Time을 약 XX분 단축할 수 있을 것으로 추정된다.

**한계:**
이 결과는 시뮬레이션 파라미터 설정의 영향을 받으며,
실데이터에서의 검증이 필요하다.
```

### 이 단계에서 계획서 활용 포인트
- 9장 KPI 표 → 쿼리 작성 기준 (원천 마트, 선정 이유)
- 4장 가설 측정 지표 → 검증 SQL/Python 설계
- 1장 전제 명시 → 인사이트 표현 방식

---

## Phase 7. 시각화 & 대시보드 (3~5일)
`feature/05-reporting`

### seaborn 시각화

```python
import seaborn as sns
import matplotlib.pyplot as plt
import folium

# ── 1. Kitchen Wait Time 히트맵 (시간대 × 카테고리) ──
pivot = df_kitchen.pivot_table(
    values='kitchen_wait_min',
    index='category',
    columns='hour_of_day',
    aggfunc='mean'
)
plt.figure(figsize=(16, 5))
sns.heatmap(pivot, cmap='YlOrRd', annot=True, fmt='.0f')
plt.title('카테고리별 시간대별 Kitchen Wait Time (분)')
plt.savefig('charts/kitchen_heatmap.png', dpi=150)

# ── 2. 수급 Gap 라인차트 (날씨 등급별) ──
df_weather.groupby(['weather_grade', 'time_slot'])['supply_gap'].mean()\
    .unstack(0).plot(figsize=(14, 5))
plt.title('시간대별 수급 Gap (날씨 등급별)')
plt.savefig('charts/supply_gap.png', dpi=150)
```

### folium Dead Zone 히트맵 (계획서 마트 2 참고)

```python
import folium
from folium.plugins import HeatMap

# Dead Zone 위치 데이터
dead_zones = df_rider[df_rider['is_dead_zone']]\
    [['dropoff_lat', 'dropoff_lng']].values.tolist()

m = folium.Map(location=[37.5665, 126.9780], zoom_start=11)
HeatMap(dead_zones,
        radius=15,
        blur=10,
        gradient={'0.4': 'blue', '0.65': 'lime', '1': 'red'}).add_to(m)

# 수요 밀집지 마커도 추가
for _, row in demand_centers.iterrows():
    folium.CircleMarker(
        location=[row['lat'], row['lng']],
        radius=5, color='blue', fill=True
    ).add_to(m)

m.save('charts/dead_zone_heatmap.html')
```

### Tableau 대시보드 구성 (계획서 KPI 순서 기반)
```
대시보드 1: Kitchen Delay 현황
  - Kitchen Wait Time by Category (막대)
  - 시간대별 Slot Congestion Index (히트맵)
  - 오버수락률 Top 20 식당 (테이블)

대시보드 2: 배차 공백 현황
  - Dead Zone 히트맵 (folium 임베드)
  - 라이더 유형별 콜사율 (파이)
  - 연속배달 vs 콜사율 (산점도)

대시보드 3: 악천후 수급 현황
  - 날씨 등급별 수급 Gap (라인)
  - 할증 금액 vs 수락률 (산점도)

대시보드 4: Executive Summary
  - KPI 9개 현황판
  - ETA 구성요소 분해 (스택 바)
  - ETA 지연율 추이 (라인)
```

---

## Phase 8. 포트폴리오 정리 (3~5일)
`feature/05-reporting`

### README.md 구조
```markdown
# 배달의민족 배달 최적화 분석

## 프로젝트 개요
- 기간: 2026-05-28 ~
- 포지션: 데이터 분석가 포트폴리오
- 핵심 질문: "ETA 지연의 진짜 원인은 무엇인가?"

## 분석 계층 위치
[계획서 1장 분석 계층 다이어그램 삽입]

## 버전 히스토리 (포트폴리오 핵심 스토리)
| 버전 | 주요 변경 | 배운 점 |
|-----|---------|-------|
| v1  | 초기 문제 정의 | CS 경험 → 정량 가설 연결 |
| v2  | 파이프라인 설계 | Airflow 오케스트레이션 |
| v3  | 논리 결함 발견 | 검증 순서 중요성 |
| v4  | 스키마 완결 | 가설-데이터 정합성 |

## 기술 스택
[계획서 10장 표 삽입]

## 주요 인사이트
1. Kitchen Delay는 Total ETA의 XX%를 차지
2. Dead Zone 라이더의 콜사율은 일반 대비 XX%p 높음
3. 악천후 시 수급 Gap이 -X.X까지 확대되나 현행 할증의 수락률 개선 효과는 미미

## 한계 및 후속 과제
[계획서 13장 요약 + v5 방향 1단락]
```

### 포트폴리오 9챕터 작성 순서
```
Chapter 9 (한계) 먼저 → Chapter 1 (Executive Summary) 마지막에
이유: 한계를 먼저 쓰면 인사이트가 과장되지 않음
```

---

## 전체 체크리스트

### Phase별 완료 기준

```
Phase 0  □ Docker 컨테이너 정상 실행
         □ 브랜치 구조 세팅 완료

Phase 1  □ 기상청 데이터 수집 완료
         □ 파라미터 확정 기록 (params_log.md) 작성
         □ Airflow DAG-1 실행 성공

Phase 2  □ 전체 DDL 실행 오류 없음
         □ FK 제약조건 전부 설정
         □ ERD 계획서 5장과 대조 완료

Phase 3  □ 전체 테이블 적재 완료
         □ 파라미터 검증 쿼리 수치 범위 내
         □ 순환논리 방지 확인 (라이더 유형별 파라미터 동일)

Phase 4  □ ETA 분해 결과 기록 (Kitchen Wait % 확인)
         □ 가설 방향 유지/수정 결정
         □ EDA 결과로 계획서 업데이트

Phase 5  □ 마트 4개 생성 완료
         □ Airflow DAG-2 의존 순서 실행 성공
         □ 마트 데이터 건수 검증

Phase 6  □ KPI 9개 쿼리 전부 실행
         □ 가설 4개 통계 검증 완료
         □ 인사이트 "만약…" 표현으로 작성

Phase 7  □ seaborn 차트 4개 이상 저장
         □ folium Dead Zone 히트맵 생성
         □ Tableau 대시보드 4개 완성

Phase 8  □ README.md 버전 히스토리 포함
         □ 포트폴리오 9챕터 작성 완료
         □ master 브랜치 최종 머지
```

---

## 자주 막히는 지점 & 해결 방법

| 막히는 지점 | 먼저 볼 것 | 해결 방향 |
|---------|----------|---------|
| 시뮬레이션 파라미터 모르겠음 | 계획서 4장 가설 표 + Phase 1 수집 데이터 | 수집 데이터의 평균값을 그대로 사용 |
| 마트 SQL Grain이 뭔지 모르겠음 | 계획서 8장 각 마트 Grain 항목 | Grain = GROUP BY 기준 |
| 가설 검증 결과가 예상과 다름 | 계획서 13장 결함 목록 | 순환논리 여부 먼저 확인 |
| Airflow DAG 실행 안 됨 | 계획서 6장 DAG 태스크 구성 | 의존 순서 재확인 |
| 인사이트 어떻게 써야 할지 모름 | 계획서 1장 전제 명시 | "만약 이런 패턴이 실데이터에서 발견된다면" |

---

> 💡 **계획서는 지도다. 실제 구현 중에 지도와 현실이 다르면 지도를 업데이트한다.**
> 계획서 v4.0의 "Living Document" 원칙대로, 각 Phase 완료 후 계획서를 검토하고
> 발견한 논리 결함이나 스키마 수정이 있으면 계획서에 반영 후 버전을 올린다.
