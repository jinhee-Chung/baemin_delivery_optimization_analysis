# 배달의민족 배달 최적화 분석 프로젝트
**종합 계획서 v4.0 | 2026-05-28**
포지션: 데이터 분석가 포트폴리오
핵심 컨셉: "배달 ETA 문제는 표면이고, 진짜 문제는 플랫폼의 공급 통제력 약화다"

---

## 1. Executive Summary

| 항목 | 내용 |
|------|------|
| KPI | 배달 ETA 단축 & 배차 안정성 향상 |
| 메인 문제 | Kitchen Delay (식당 조리 지연) |
| 보조 문제 | 라이더 콜사 (배차 공백), 악천후 수급 붕괴 |
| 근본 문제 | 배달 계약업체 의존도 → 플랫폼 협상력·품질 통제 약화 |
| 데이터 전략 | 시뮬레이션(메인) + 외부 공개데이터(파라미터 현실 근거) 병행 |
| 분석 차별점 | CS 실무 경험에서 문제 발굴 → 정량 가설 수립 → 시뮬레이션으로 검증 |

> ⚠ **전제 명시:** 본 분석은 시뮬레이션 데이터로 가설 검증 구조를 설계하고, 분석가로서 '어떤 질문을 해야 하는가'와 '어떤 지표를 어떻게 설계하는가'를 보여주는 것이 목적이다. 인사이트는 '만약 이런 패턴이 실데이터에서 발견된다면'으로 표현한다.

---

## 2. 시장 현황 및 도메인 이해

### 2-1. 활용 외부 공개데이터 및 파라미터 매핑

| 데이터 출처 | 활용 목적 | 뒷받침하는 파라미터 |
|------------|----------|-------------------|
| 공정거래위원회 | 배달앱 시장 규모·성장 추이 | 시장 규모 배경 서술 |
| 모바일인덱스, 와이즈앱 | 플랫폼별 점유율 비교 | 배민의 시장 지위 근거 |
| 통계청, 고용노동부 | 배달 종사자 수, 콜 수 현황 | 라이더 공급 파라미터 규모 검증 |
| DoorDash / Uber Eats 논문 | ETA가 핵심 경쟁지표임을 외부 사례로 입증 | ETA 분석 방향의 근거 |
| 기상청 공공데이터 | 강수량·풍속 분포 | 악천후 등급 기준, 날씨 파라미터 |
| 배민 기술 블로그 | Kitchen 운영, 배차 알고리즘 언급 | 조리시간 파라미터 방향 참고 |

### 2-2. 도메인 구조 (3자 플랫폼)

```
고객(User) ──주문──▶ 플랫폼(Baemin) ──배차──▶ 라이더(Rider)
               │
           ◀──수락── 식당(Restaurant)
```

**핵심 흐름:** 주문 생성 → 식당 수락 → 조리 → 픽업 대기 → 라이더 배차 → 픽업 → 배달 완료

### 2-3. ERD 기반 도메인 테이블 구조

- **출처 명시:** ERD는 erdcloud.com 학습용 클론 ERD(12개 테이블)를 도메인 이해 목적으로 활용. 분석용 OLTP 스키마는 이 ERD를 참고하되 별도 설계한다. (5장 참조)
- **ERD 한계:** 배달원(Rider) 테이블 없음 → 라이더 데이터는 분석용 스키마에서 독자 설계.

### 2-4. DW 관점 테이블 분류

| 테이블 유형 | 대상 테이블 예시 | DW 적재 전략 |
|------------|----------------|-------------|
| 원장성 (Master) | user_master, restaurant_master, rider_master | 시계열 적재 + 선분이력 구축 |
| 거래성 (Transactional) | order_transaction, delivery_log, payment_transaction | 1:1 적재, 날짜 파티셔닝 |
| 양면성 (Duplicity) | restaurant_application, rider_contract | 점이력 구축 |
| 이력 (History) | user_address_history, restaurant_status_history | 선분/점이력 방식 확정 후 최적화 |

---

## 3. 문제 정의 (3층 구조)

```
[표면 문제] 배달 ETA 지연
      ↓
[운영 문제] Kitchen Delay (메인) + 라이더 콜사 + 악천후 수급 붕괴
      ↓
[구조 문제] 배달 계약업체 의존도
           → 플랫폼 협상력 약화 / 품질 통제 불가 / 수급 안정성 저하
      ↓
[솔루션 방향] 자체 라이더 수락률 최적화 → 계약업체 의존도 탈피
```

**Kitchen Delay를 메인으로 선정한 이유**

1. CS 문의 토픽으로 외부 가시성이 가장 높음 (고객 체감 직결)
2. '정성 CS 데이터 → 정량 가설 → 데이터 검증' 스토리가 자연스러움
3. 식당 단위 개입 vs 플랫폼 단위 개입으로 솔루션이 명확히 분기됨

> ⚠ **논리보완:** 'Kitchen Delay가 ETA의 가장 큰 요인'은 CS 경험 기반 가정이다. 분석 전에 mart_eta_decomposition에서 ETA 구성요소를 먼저 분해 검증해야 논리적 순서가 맞다. → 파이프라인 Step 5에 선행 배치.

---

## 4. 가설 수립 및 측정 지표

### 4-1. 문제 1: Kitchen Delay

| 가설 | 내용 | 주요 식당 유형 | 측정 지표 |
|-----|------|--------------|---------|
| A. 오버수락 | 식당이 처리 가능량보다 무리하게 수락 | 한식, 피자 (조리시간 긴 유형) | 오버수락률 = 동시 처리 주문 수 / max_capacity |
| B. 피크 집중 | 같은 시간대 주문 폭주로 구조적 지연 | 치킨, 분식 (주문이 몰리는 구조) | Slot Congestion Index = 슬롯 내 주문 수 |

| 식당 유형 | 평균 조리시간 | 피크 집중도 | 연결 가설 |
|----------|------------|-----------|---------|
| 한식 | 25-35분 | 중간 | 가설 A |
| 피자 | 20-30분 | 중간 | 가설 A |
| 치킨 | 15-20분 | 매우 높음 | 가설 B |
| 분식/기타 | 10-15분 | 높음 | 가설 B |

**피크타임:** 점심 11:00–12:30 / 저녁 18:00–19:30

### 4-2. 문제 2: 라이더 콜사

**용어 정의:** '콜사(콜 사절)' = 라이더가 배차 요청(콜)을 거절하는 행위. 콜사 발생 시 재배차 지연 → ETA 직접 증가.

| 가설 | 내용 | 측정 지표 |
|-----|------|---------|
| A. Dead Zone | 배달 완료 좌표가 다음 콜 수요 밀집지와 멀어 고립 | Post-delivery Cold Zone Rate (좌표 기반) |
| B. 타이트한 콜 | 완료~다음 콜 간격이 너무 짧아 체감 여유 없음 | 완료 후 콜 요청까지 소요 시간 |
| C. 수수료 구조 | 한집배달 인센티브가 콜사 비용보다 낮음 | 콜사율 × 배달유형 × 수수료 구간 교차분석 |
| D. 피로도 | 연속 배달 N건 이후 콜사율 급증 | consecutive_deliveries vs 콜사율 |

### 4-3. 문제 3: 악천후 수급 불균형

| 가설 | 내용 | 측정 지표 |
|-----|------|---------|
| A. 콜사율 급등 | 악천후 강도↑ → 콜사율↑ | 날씨 등급별 콜사율 |
| B. Kitchen 동반 악화 | 주문 폭증 → Kitchen Delay도 악화 | 날씨 등급 × Kitchen Wait Time |
| C. 프로모션 효과 | 현재 할증이 수락률에 실제 효과가 있는가 | 수락률 예측 모델 → 피처 중요도 |

| 날씨 등급 | 주문량 배수 | 라이더 공급 배수 | 수급 Gap |
|---------|-----------|--------------|---------|
| 맑음 | 1.0 | 1.0 | 0 |
| 흐림 | 1.1 | 0.95 | -0.15 |
| 비 | 1.3 | 0.75 | -0.55 |
| 폭우/눈 | 1.5 | 0.5 | -1.0 |

> ⚠ CS 문의량 체감 3배 / 주문량 1.5배는 보수적 추정 → 실데이터 확보 시 재검증 명시

**프로모션 수락률 예측 모델 피처 (가설 C 측정 방법)**

```
[입력 피처]
날씨 강도, 배달 거리, 완료 후 콜 밀도, 할증 금액,
연속 배달 수, 시간대, 배달 유형 (한집/알뜰)

[출력]
수락률 (0~1) → 피처 중요도 도출 → 프로모션 설계 근거
```

> 스키마 연결: promotions 테이블의 incentive_amount / incentive_rate가 "할증 금액" 피처의 실제 데이터 소스. deliveries.promotion_id를 통해 배달 건별 적용 프로모션을 조인한다.

---

## 5. 데이터 전략

### 5-1. 분석용 OLTP 스키마

> **출처 구분:** ERD(erdcloud)와 무관하게 분석 목적으로 독자 설계. 실제 배민 DB 구조와 다를 수 있으며, 포트폴리오 시뮬레이션용임을 명시.

```sql
orders [수정]
  order_id               INT
  restaurant_id          INT          -- FK → restaurants
  user_id                INT          -- [추가] FK → users
  delivery_address_id    INT          -- [추가] FK → user_addresses (주문 당시 배달지)
  order_accepted_at      TIMESTAMP    -- 식당 주문 수락 시각
  cooking_started_at     TIMESTAMP    -- 조리 시작 시각
  pickup_ready_at        TIMESTAMP    -- 픽업 준비 완료 시각
  order_type             VARCHAR      -- 한집 / 알뜰
  status                 VARCHAR      -- pending/accepted/cooking/ready/delivering/completed/cancelled
  created_at             TIMESTAMP
  updated_at             TIMESTAMP    -- [추가] 상태 변경·취소 추적 이력
  cancelled_at           TIMESTAMP    -- [추가] 취소 시각 (NULL = 정상완료)
  cancel_reason          VARCHAR      -- [추가] 고객취소/식당거절/라이더미배차
  -- is_converted_offline 제거 → restaurants / riders 테이블로 이동


restaurants [수정] ← restaurant_master
  -- [고정 속성 - 마스터 불변값]
  restaurant_id          INT          -- PK
  business_number        VARCHAR      -- [추가] 사업자번호 (마스터 식별 고정값)
  registered_at          TIMESTAMP    -- [추가] 최초 등록일 (가입 이력 기준)
  -- [가변 속성 - 운영 중 변경 가능]
  category               VARCHAR      -- 한식 / 치킨 / 피자 / 분식
  region                 VARCHAR
  lat                    FLOAT        -- 위도 (Dead Zone 분석·거리 계산)
  lng                    FLOAT        -- 경도
  avg_cooking_time       INT
  max_capacity           INT          -- 동시 처리 가능 주문 수 (오버수락 측정 핵심)
  operation_status       VARCHAR      -- [추가] 영업중/휴무/폐업 (가변)
  delivery_radius_km     FLOAT        -- [추가] 배달 가능 반경 (가변)
  min_order_amount       INT          -- [추가] 최소 주문금액 (가변)
  is_converted_offline   BOOLEAN      -- [이동] orders에서 이쪽으로
  converted_at           TIMESTAMP    -- 오프라인→온라인 전환 시점


deliveries [수정]
  delivery_id            INT
  order_id               INT
  rider_id               INT
  promotion_id           INT          -- [추가] FK → promotions (가설 3-C 수락률 검증)
  assigned_at            TIMESTAMP
  pickup_at              TIMESTAMP
  completed_at           TIMESTAMP
  pickup_lat             FLOAT
  pickup_lng             FLOAT
  dropoff_lat            FLOAT
  dropoff_lng            FLOAT
  distance_km            FLOAT
  total_weight_kg        FLOAT        -- [추가] order_items 무게 합산 → 배차 차량 선정 근거
  is_rejected            BOOLEAN
  rejection_reason       VARCHAR      -- 거리 / 시간 / 날씨
  commission_amount      INT          -- [추가] 해당 배달 수수료 금액 (가설 2-C)
  commission_rate        FLOAT        -- [추가] 수수료율 (배달유형별 비교)
  expected_eta_min       INT          -- [추가] 주문 수락 시 고객에게 안내한 예상 소요시간(분)
  actual_eta_min         INT          -- [추가] 실제 총 소요시간 = Total ETA (completed_at - order_accepted_at)
  eta_gap_min            INT          -- [추가] actual - expected (양수=지연 / 음수=조기도착 / 분석 핵심 타겟)


riders [수정] ← rider_master
  -- [고정 속성 - 마스터 불변값]
  rider_id               INT          -- PK
  registered_at          TIMESTAMP    -- [추가] 등록일 (고정)
  -- [가변 속성 - 운영 중 변경 가능]
  rider_type             VARCHAR      -- 자체 / 계약업체
  vehicle_type           VARCHAR      -- 자전거/오토바이/보도/자동차 (배차 가중치 계산)
  activity_region        VARCHAR      -- [추가] 활동 지역 (가변 - 이동 가능)
  delivery_status        VARCHAR      -- [추가] 대기중/배달중/휴식/오프라인 (가변)
  rating                 FLOAT        -- [추가] 라이더 평점 (가변 - 누적 갱신)
  consecutive_deliveries INT          -- 연속 배달 수 (피로도 proxy)
  current_lat            FLOAT
  current_lng            FLOAT
  call_informed_at       TIMESTAMP    -- 배차 요청 알림 시각 (가설 2-B: 완료→다음콜 간격 핵심)
  is_converted_offline   BOOLEAN      -- [이동] orders에서 이쪽으로


weather
  weather_id             INT
  recorded_at            TIMESTAMP
  precipitation          FLOAT        -- 강수량 (mm)
  wind_speed             FLOAT        -- 풍속 (m/s)
  weather_grade          VARCHAR      -- 맑음 / 흐림 / 비 / 폭우 / 눈
  is_adverse             BOOLEAN
  demand_multiplier      FLOAT
  supply_multiplier      FLOAT


menus [신규] ← menu_master
  -- [고정 속성 - 마스터 불변값]
  menu_id                INT          -- PK
  restaurant_id          INT          -- FK → restaurants (고정)
  -- [가변 속성 - 운영 중 변경 가능]
  menu_name              VARCHAR
  category               VARCHAR      -- 메인/사이드/음료
  price                  INT          -- [마스터 가변] 가격 변동 가능
  description            TEXT         -- [추가] 메뉴 설명 (가변)
  image_url              VARCHAR      -- [추가] 메뉴 이미지 URL (가변)
  is_available           BOOLEAN      -- [추가] 품절 여부 (가변 - 실시간 변경)
  avg_weight_per_unit    FLOAT        -- 단위당 무게(g) → 총중량 추정 → 차량 배차 기준
  avg_prep_time_min      INT          -- 해당 메뉴의 조리 기여 시간
  created_at             TIMESTAMP
  updated_at             TIMESTAMP


order_items [신규] — 주문 상세 트랜잭션
  order_item_id          INT          -- PK
  order_id               INT          -- FK → orders
  menu_id                INT          -- FK → menus
  quantity               INT          -- 수량
  unit_price             INT          -- 주문 시점 단가 (메뉴가격 변동 이력 보존)
  subtotal               INT          -- quantity × unit_price
  estimated_weight_g     FLOAT        -- quantity × avg_weight_per_unit
  created_at             TIMESTAMP


payments [신규] — 결제 트랜잭션
  payment_id             INT          -- PK
  order_id               INT          -- FK → orders
  amount                 INT          -- 결제 금액
  payment_method         VARCHAR      -- 카드/배민페이/현금
  status                 VARCHAR      -- paid/refunded/partial_refund
  paid_at                TIMESTAMP
  refund_amount          INT          -- 부분취소 대응
  refunded_at            TIMESTAMP    -- Kitchen Delay → 취소 → 환불 흐름 추적


cs_tickets [신규] — CS 처리 이력
  ticket_id              INT          -- PK
  order_id               INT          -- FK → orders
  user_id                INT          -- [추가] FK → users (CS 다발 고객 분석)
  issue_type             VARCHAR      -- ETA지연/음식불량/배달오류/취소요청
  issue_detail           TEXT
  is_eta_related         BOOLEAN      -- ETA 관련 여부 (분석 핵심 필터)
  created_at             TIMESTAMP    -- CS 접수 시각
  resolved_at            TIMESTAMP
  resolution             VARCHAR      -- 환불/재배달/안내
  response_time_min      INT          -- CS 처리 소요 시간


promotions [신규] — 프로모션/인센티브
  promotion_id           INT          -- PK
  promotion_type         VARCHAR      -- 악천후할증/심야수당/거리인센티브
  weather_grade          VARCHAR      -- 적용 날씨 조건 (NULL = 상시)
  vehicle_type           VARCHAR      -- 적용 차량 조건 (NULL = 전체)
  incentive_amount       INT          -- 할증 금액
  incentive_rate         FLOAT        -- 할증률
  min_distance_km        FLOAT        -- 적용 최소 거리
  start_at               TIMESTAMP
  end_at                 TIMESTAMP
  is_active              BOOLEAN


users [신규] ← user_master
  -- [고정 속성 - 마스터 불변값]
  user_id                INT          -- PK
  social_account_type    VARCHAR      -- [추가] 카카오/네이버/애플 등 소셜 계정 유형 (고정)
  signup_at              TIMESTAMP    -- 가입일 (고정)
  -- [가변 속성 - 운영 중 변경 가능]
  nickname               VARCHAR      -- [추가] 닉네임 (가변 - 변경 가능)
  phone                  VARCHAR      -- [추가] 연락처 (가변)
  email                  VARCHAR
  gender                 VARCHAR      -- 세그먼트 분석 (선택적)
  age_group              VARCHAR      -- 10대/20대/30대... (구간으로 관리)
  last_order_at          TIMESTAMP    -- 마지막 주문일 (이탈 감지, 갱신됨)
  total_order_count      INT          -- 누적 주문 수 (헤비유저 분류, 갱신됨)
  is_active              BOOLEAN      -- 활성 여부


user_addresses [신규] — 배달 주소 이력 (1:N)
  address_id             INT          -- PK
  user_id                INT          -- FK → users
  address_label          VARCHAR      -- 집/회사/기타
  address_full           VARCHAR      -- 전체 주소
  lat                    FLOAT        -- 배달 목적지 위도 (수요 밀집지·Dead Zone 분석)
  lng                    FLOAT        -- 배달 목적지 경도
  is_default             BOOLEAN      -- 기본 배달지 여부
  created_at             TIMESTAMP
  deleted_at             TIMESTAMP    -- soft delete (이력 보존)
```

**테이블 관계 [v4 최종]**

```
-- 고객
users            1:N  orders
users            1:N  user_addresses
users            1:N  cs_tickets
user_addresses   1:N  orders (delivery_address_id)

-- 식당 · 메뉴
restaurants      1:N  orders
restaurants      1:N  menus

-- 주문
orders           1:N  order_items
orders           1:1  payments
orders           1:1  deliveries
orders           1:N  cs_tickets
menus            1:N  order_items

-- 배달 · 라이더
riders           1:N  deliveries
promotions       1:N  deliveries
weather          1:N  deliveries (recorded_at 시간 조인)
```

**분석에서 어디에 쓰이나?**

| 분석 목적 | 활용 테이블 |
|---------|-----------|
| 신규 vs 기존 고객의 ETA 불만 차이 | users.signup_at + cs_tickets |
| 배달 목적지 밀집 지역 분석 | user_addresses.lat/lng |
| 헤비유저의 콜사율 상관관계 | users.total_order_count + deliveries.is_rejected |
| 고객 이탈 - Kitchen Delay 연관성 | users.last_order_at + mart_kitchen_delay |
| Dead Zone: 수요지 vs 라이더 완료 좌표 | user_addresses.lat/lng + deliveries.completed_at 좌표 비교 |

---

## 6. 올바른 분석 프로세스 흐름

> ⚠ **설계 원칙:** DB/파이프라인(그릇)이 먼저 구축되어야 한다. 외부 리서치 데이터는 시뮬레이션 생성 이후 파라미터 검증·보정 용도로 활용한다. 둘 다 feature/02-data-collection 브랜치에서 처리한다.

| Step | 작업 내용 | Git 브랜치 | Airflow 역할 |
|------|----------|-----------|------------|
| Step 1 | 도메인 분석 + 문제 정의 + 가설 문서화 | feature/01-domain-analysis | - |
| Step 2 | [DB/인프라] Docker Compose 작성 + PostgreSQL 컨테이너 + OLTP DDL 작성 및 적재 | feature/02-data-collection | - (1회성 셋업, 자동화 불필요) |
| Step 3 | [시뮬레이션] Python 스크립트로 orders / restaurants / deliveries / riders / weather 생성 및 OLTP 적재 (1차) | feature/02-data-collection | - (1회성 실행) |
| Step 4 | [외부 데이터] 기상청·통계청 공개데이터 수집 → 시뮬레이션 파라미터와 비교 검증 → 필요시 파라미터 조정 후 재적재 | feature/02-data-collection | O [DAG-1] 외부 API 스케줄 수집 DAG (오케스트레이션: 수집→검증→저장 순서를 자동 스케줄링) |
| Step 5 | [선행검증 EDA] ETA 구성요소 분해 쿼리 → Kitchen Delay 비중 확인 + 탐색적 분석 | feature/03-eda | - |
| Step 6 | [마트 ETL] OLTP → 4개 데이터 마트 변환·적재 | feature/04-modeling | O [DAG-2] 마트 ETL 오케스트레이션 DAG (오케스트레이션: 마트 간 의존 순서 보장 - kitchen/rider/weather → eta_decomposition) |
| Step 7 | [분석] KPI 지표 8개 쿼리 + 가설 A/B/C/D 검증 | feature/04-modeling | - |
| Step 8 | [보고] seaborn / folium 시각화 + Tableau 대시보드 + 포트폴리오 9챕터 정리 | feature/05-reporting | - |

**Airflow 역할 상세**

| DAG 위치 (브랜치) | 트리거 방식 | 태스크 구성 |
|-----------------|-----------|-----------|
| DAG-1: 외부 데이터 수집 — feature/02-data-collection | 스케줄 (@daily 또는 1회 실행) | ① 기상청 API 호출 → ② 통계청 파일 다운로드 → ③ 파라미터 비교 테이블 생성 → ④ PostgreSQL 적재 |
| DAG-2: 마트 ETL — feature/04-modeling | 수동 트리거 or @once | ① mart_kitchen_delay → ② mart_rider_rejection → ③ mart_weather_supply_gap → ④ mart_eta_decomposition (순서 의존성 보장) |

> 💡 **오케스트레이션(Orchestration)이란:** 여러 태스크(작업)의 실행 순서, 의존성, 재시도, 스케줄을 중앙에서 자동으로 관리하는 것. 예: 마트 A가 완료된 후에만 마트 B를 실행 → Airflow DAG가 이 순서를 보장해준다.

---

## 7. Git 브랜치 전략

**저장소:** baemin_delivery_optimization_analysis | **현재 작업:** feature/01-domain-analysis

```
master
├── feature/01-domain-analysis   ← [현재] 도메인 이해, 문제 정의
├── feature/02-data-collection   ← 스키마 설계, Docker + DDL,
│                                   시뮬레이션 생성 & 적재,
│                                   외부 리서치 수집 (Airflow DAG-1),
│                                   파라미터 검증 & 조정
├── feature/03-eda               ← ETA 분해 선행검증, 탐색 분석
├── feature/04-modeling          ← 마트 ETL (Airflow DAG-2), KPI 쿼리, 가설 검증
└── feature/05-reporting         ← 시각화, 대시보드, 포트폴리오 정리
```

**브랜치 핵심 산출물**

| 브랜치 | 핵심 산출물 |
|--------|-----------|
| feature/01-domain-analysis | 계획서 v3.0, ERD 분석 노트, 문제 정의 문서 |
| feature/02-data-collection | docker-compose.yml, ddl.sql, simulate_data.py, airflow/dag_external_collect.py |
| feature/03-eda | eda_notebook.ipynb, eta_decomposition.sql |
| feature/04-modeling | mart_*.sql, airflow/dag_mart_etl.py, hypothesis_test.ipynb |
| feature/05-reporting | dashboard.twb, charts/, README.md (포트폴리오 최종) |

> 💡 **브랜치 병합 규칙:** 각 브랜치 작업 완료 후 master로 PR → 머지. master는 항상 완성 상태만 유지.

---

## 8. 데이터 마트 설계 (4개)

```
mart_kitchen_delay      ─┐
mart_rider_rejection    ─┼──▶ mart_eta_decomposition ──▶ Tableau 대시보드
mart_weather_supply_gap ─┘
```

### 마트 1: mart_kitchen_delay

| 항목 | 내용 |
|-----|------|
| 목적 | Kitchen Delay 원인 분석 → 식당 단위 개입 vs 플랫폼 단위 개입 분기 근거 도출 (가설 A/B 검증) |
| Grain | 식당 × 시간대 × 카테고리 |
| 핵심 지표 | Kitchen Wait Time, Slot Congestion Index, 오버수락률 |
| 핵심 공식 | Kitchen Wait Time = pickup_ready_at - order_accepted_at \| 오버수락률 = 동시 진행 주문 수 / max_capacity (> 1.0 = 오버수락) |

### 마트 2: mart_rider_rejection

| 항목 | 내용 |
|-----|------|
| 목적 | 콜사 패턴 분석 + Dead Zone 탐지 → 배차 알고리즘 개선 및 라이더 계약 구조 검토 근거 도출 (가설 A~D 검증) |
| Grain | 라이더 × 완료 좌표(격자) × 배달 유형 |
| 핵심 지표 | 콜사율, Dead Zone 발생률, 라이더 유형별 콜사율, 피로도-콜사율 |
| 시각화 | folium Dead Zone 히트맵 |

**계약업체 의존도 수치화 분석 방향**

```
계약업체 라이더 콜사율  vs  자체 라이더 콜사율
→ 격차가 크다면: 계약업체 의존도 리스크의 정량적 근거
→ riders.rider_type 기준으로 분리 집계
```

> ⚠ 시뮬레이션에서 두 유형의 콜사율 파라미터를 동일하게 설정하거나 다중 시나리오로 분석한다. 결론은 "만약 실데이터에서 이 격차가 관찰된다면"으로 프레임을 전환한다. (순환논리 방지 — 13장 결함 #1 참조)

**Dead Zone 분석 데이터 소스**

```
라이더 완료 좌표: deliveries.dropoff_lat / dropoff_lng
수요지 좌표:     user_addresses.lat / lng
식당 좌표:       restaurants.lat / lng

→ 라이더 완료 지점과 인근 수요지(배달 목적지, 식당) 밀집도를 비교해
   수요가 적은 지역에 고립된 라이더를 Dead Zone으로 정의한다.
```

### 마트 3: mart_weather_supply_gap

| 항목 | 내용 |
|-----|------|
| 목적 | 악천후 수급 불균형 분석 + 프로모션 효과 검증 → 악천후 대응 인센티브 구조 최적화 근거 도출 (가설 A~C 검증) |
| Grain | 시간대 × 날씨 등급 × 지역 |
| 핵심 지표 | 수급 Gap, 날씨 등급별 콜사율, 프로모션 피처별 수락률 변화 |
| 핵심 공식 | 수급 Gap = (실제 주문 수 / 평시 평균) - (실제 라이더 공급 수 / 평시 평균) |

### 마트 4: mart_eta_decomposition (통합 마트)

| 항목 | 내용 |
|-----|------|
| 목적 | ETA 전체 구성요소 분해 → 3개 마트 인사이트의 ETA 영향 최종 검증 + Executive Summary 소스 (통합 KPI 모니터링) |
| Grain | 주문 단위 |
| 핵심 공식 | Kitchen Wait Time + Pickup Wait Time + Last-mile Time = Total ETA |
| 역할 | 3개 마트 인사이트의 ETA 영향 최종 검증 + Executive Summary 소스 |

**ETA 분해 공식 상세**

```
Total ETA = Kitchen Wait Time       (order_accepted_at → pickup_ready_at)
          + Pickup Wait Time        (pickup_ready_at → 라이더 픽업 완료)
          + Last-mile Time          (픽업 완료 → 배달 완료)

ETA 지연 측정 = eta_gap_min (deliveries.actual_eta_min - deliveries.expected_eta_min)
              양수 = 지연 / 음수 = 조기 도착
```

> ⚠️ **분석 순서 정당화**: 이 마트는 파이프라인 Step 5(선행검증 EDA)에서 가장 먼저 운영한다. Kitchen Delay(마트 1)를 메인으로 설정한 것은 CS 경험 기반 가정이므로, mart_eta_decomposition에서 Kitchen Wait Time이 Total ETA에서 차지하는 비중이 가장 크다는 것이 먼저 확인되어야 마트 1의 심층 분석이 논리적으로 정당화된다. 반대로 Pickup Wait Time이나 Last-mile Time이 더 크게 나온다면 분석 메인 타겟을 재설정해야 한다.

---

## 9. 핵심 KPI 지표 8개

| # | 지표명 | 원천 마트 | 선정 이유 |
|---|-------|---------|---------|
| 1 | Kitchen Wait Time | mart_kitchen_delay | ETA 핵심 구성요소, 분석 메인 타겟 |
| 2 | Slot Congestion Index | mart_kitchen_delay | Kitchen Wait Time의 선행 지표 |
| 3 | 오버수락률 | mart_kitchen_delay | 가설 A 검증 핵심 |
| 4 | Pickup Wait Time | mart_eta_decomposition | Kitchen Wait Time의 결과 지표 |
| 5 | 콜사율 | mart_rider_rejection | 배차 공백 → ETA 직접 증가 |
| 6 | Dead Zone 발생률 | mart_rider_rejection | 콜사율의 선행 지표 |
| 7 | 수급 Gap | mart_weather_supply_gap | 악천후 ETA 폭등 직접 원인 |
| 8 | 피로도-콜사율 | mart_rider_rejection | 프로모션 피처, 공급 안정화 근거 |
| 9 | ETA 지연율 (eta_gap_min > 0 비율) | mart_eta_decomposition | 예상 대비 실제 지연을 직접 측정하는 최종 성과 지표 |

> **KPI #9 추가 이유:** 기존 8개 지표는 모두 지연의 원인(Kitchen Wait Time, 콜사율 등)을 측정하는 선행 지표다. eta_gap_min은 "고객에게 안내한 ETA 대비 실제로 얼마나 늦었는가"를 측정하는 결과 지표로, deliveries 테이블에 직접 저장되어 있으며 분석의 최종 성과를 평가하는 기준이 된다.

**cs_tickets 분석 연결**

cs_tickets는 독립 마트를 갖지 않고 아래 방식으로 기존 마트와 교차 분석한다.

| 분석 목적 | 연결 마트 | 조인 방법 |
|---------|---------|---------|
| Kitchen Delay → CS 접수율 연관성 | mart_kitchen_delay | order_id 기준 조인 |
| ETA 지연 → 고객 이탈 연관성 | mart_eta_decomposition | order_id → user_id → users.last_order_at |
| 악천후 시 CS 폭증 패턴 | mart_weather_supply_gap | recorded_at 시간 조인 |

---

## 10. 기술 스택

| 도구 | 역할 | 적용 브랜치 |
|-----|------|-----------|
| Docker | 환경 세팅 / PostgreSQL + Python 컨테이너 | feature/02 |
| PostgreSQL | OLTP 저장 + 데이터 마트 저장 | feature/02, 04 |
| Python | 시뮬레이션 데이터 생성 / ETL 스크립트 / 분석 시각화 | feature/02, 03, 04 |
| Apache Airflow | DAG-1 외부 데이터 수집 스케줄링 \| DAG-2 마트 ETL 오케스트레이션 | feature/02, 04 |
| SQL | 데이터 조회 / KPI 지표 쿼리 / 마트 생성 | feature/03, 04 |
| seaborn / folium | 분석 시각화 + Dead Zone 히트맵 | feature/05 |
| Tableau | 최종 대시보드 (경영진 보고용) | feature/05 |

---

## 11. 솔루션 방향

| 시기 | 솔루션 | 근거 마트 |
|-----|-------|---------|
| 단기 | 오버수락 식당 실시간 경고 시스템 | mart_kitchen_delay |
| 단기 | 악천후 프로모션 피처 최적화 | mart_weather_supply_gap |
| 중기 | Dead Zone 기반 배차 알고리즘 개선 | mart_rider_rejection |
| 중기 | 자체 라이더 비율 확대 계획 수립 | mart_rider_rejection |
| 장기 | 계약업체 의존도 목표치 설정 + KPI화 | 구조적 문제 정량 근거 |
| 장기 | 라이더 데이터 내재화 → 배차 알고리즘 고도화 | 데이터 단절 문제 해결 |

---

## 12. 포트폴리오 목차 (9챕터)

1. Executive Summary
2. 시장 현황 분석 (외부 공개데이터 활용)
   - 2-1. 시장 규모 및 성장 추이 (공정위)
   - 2-2. 플랫폼별 점유율 비교 (모바일인덱스, 와이즈앱)
   - 2-3. 콜 수 및 배차 현황 (통계청, 고용노동부)
   - 2-4. ETA가 핵심 경쟁지표인 근거 (DoorDash/Uber Eats 논문)
3. 문제 정의 (3층 구조)
4. 가설 수립 및 측정 설계
5. 데이터 설계 및 수집
   - 5-1. 분석용 OLTP 스키마
   - 5-2. 외부 공개데이터 파라미터 근거
   - 5-3. 시뮬레이션 데이터 생성 방법론
6. 분석
   - 6-1. ETA 구성요소 분해 (선행 검증)
   - 6-2. Kitchen Delay 분석 (가설 A/B)
   - 6-3. 라이더 콜사 분석 (가설 A~D)
   - 6-4. 악천후 수급 분석 (가설 A~C)
7. 인사이트 (So what? 관점)
8. 솔루션 제안 (단기/중기/장기)
9. 한계 및 후속 과제

---

## 13. 논리 검토 및 한계 명시

### 13-1. 발견된 논리적 결함 및 해결 방향

| # | 결함 유형 | 내용 | 해결 방향 |
|---|---------|------|---------|
| 1 | 순환논리 | 시뮬레이션에서 계약업체 콜사율을 높게 설정 → 그 데이터로 '계약업체가 문제'를 검증하는 건 순환논리 | 라이더 유형 파라미터를 동일하게 설정하거나 다중 시나리오. 결론은 '만약 실데이터에서…' 프레임으로 전환 |
| 2 | 검증 순서 역전 | Kitchen Delay가 ETA의 주된 원인이라고 전제하고 분석 | 파이프라인 Step 5에 ETA 분해 선행 배치 (v3.0 반영 완료) |
| 3 | 임계값 근거 부재 | Slot Congestion Index의 시간 슬롯 기준, N건 임계값 근거 없음 | CS 경험 기반임을 명시 + 민감도 테스트(sensitivity test) 수행 |
| 4 | ERD-스키마 출처 혼용 | 학습용 ERD를 실제 배민 구조인 것처럼 오해할 소지 | ERD는 도메인 이해 목적, 분석 스키마는 별도 설계 명시 (2장/5장 처리) |
| 5 | 외부 데이터 매핑 불완전 | 외부 데이터가 어떤 파라미터를 뒷받침하는지 연결 고리 미흡 | 시뮬레이션 파라미터 설정 시 각 수치의 근거 출처를 테이블로 명시 |
| 6 | 콜사 가설 우선순위 미정 | 가설 A~D가 동시에 적용될 수 있어 판단 기준 없음 | EDA에서 요인별 상관관계 탐색 → 피처 중요도 기반 우선순위 결정 |

### 13-2. 구조적 한계

**한계 1: 시뮬레이션 데이터의 근본 한계**
파라미터 설정에 따라 결론이 달라질 수 있음. '분석 구조 설계 능력'을 보여주는 것이 목적임을 명확히 할 것.

**한계 2: Dead Zone 공간 분석**
단순 랜덤 좌표 사용 시 신뢰도 저하. 서울 행정동 경계 공공데이터로 현실적인 좌표 분포 구성 권장.

**한계 3: 계약업체 의존도 단일 지표 부재**
정성적 개념 → mart_rider_rejection에서 복수 지표(콜사율, 피로도, 배달시간)로 보완 필요.

---

## 14. 다음 액션 플랜

| 순서 | 내용 | 브랜치 |
|-----|------|-------|
| Step 1 | 도메인 분석 완료 + 문제 정의 문서화 + 계획서 v3.0 커밋 | feature/01-domain-analysis |
| Step 2 | Docker Compose 작성 (PostgreSQL + Python 컨테이너) + OLTP DDL 작성 | feature/02-data-collection |
| Step 3 | Python 시뮬레이션 스크립트 작성 (orders, restaurants, deliveries, riders, weather) | feature/02-data-collection |
| Step 4 | PostgreSQL 스키마 DDL + 시뮬레이션 데이터 1차 적재 | feature/02-data-collection |
| Step 5 | 외부 공개데이터 수집 (기상청, 통계청) + Airflow DAG-1 작성 + 파라미터 검증 및 조정 | feature/02-data-collection |
| Step 6 | ETA 분해 쿼리 → Kitchen Delay 비중 선행 확인 + EDA | feature/03-eda |
| Step 7 | ETL + 데이터 마트 4개 구축 + Airflow DAG-2 작성 | feature/04-modeling |
| Step 8 | KPI 쿼리 + 가설 A/B/C/D 검증 분석 | feature/04-modeling |
| Step 9 | seaborn / folium 시각화 + Tableau 대시보드 + 포트폴리오 9챕터 최종 정리 | feature/05-reporting |

---

> 💡 **본 계획서는 Living Document입니다.** 분석 진행에 따라 섹션을 업데이트하고 버전을 관리합니다. (현재: v4.0)

---

## v3 → v4 변경 분석

> **비교 기준**: v3 = `baemin_plan_v3.md` (종합 계획서 v3.0) / v4 = `baemin_plan_v4.md` (종합 계획서 v2.0)
> v3와 v4의 차이는 **5장(데이터 전략 / OLTP 스키마) 단 하나**다. 나머지 모든 섹션(1~4장, 6~14장)은 동일하다.

---

### 🔴 v3에 있었으나 v4에서 삭제된 항목

---

**① orders 테이블의 is_converted_offline 필드**

v3에서 orders 테이블에 있던 `is_converted_offline` 필드가 v4에서 제거됐다.

**삭제 이유:** 오프라인→온라인 전환 여부는 주문 단위 속성이 아니라 식당/라이더 단위 속성이다. 주문마다 이 값이 달라지지 않으므로 orders에 두는 것은 비정규화다. v4에서는 restaurants와 riders 테이블로 이동하고 `converted_at`(전환 시점)까지 함께 관리하도록 수정했다. 데이터 정합성 측면에서 올바른 이동이다.

---

### 🟢 v4에서 새로 추가된 항목

---

**① users + user_addresses 테이블 (신규)**

고객 마스터와 배달 주소 이력 테이블이 추가됐다. users는 고정속성(user_id, social_account_type, signup_at)과 가변속성(nickname, phone, last_order_at 등)으로 분리됐고, user_addresses는 배달 목적지 위경도를 포함한다.

**추가 이유:** v3의 스키마에는 주문자 정보가 전혀 없었다. 이 상태에서는 두 가지 분석이 구조적으로 불가능하다. 첫째, 고객 세그먼트(신규 vs 기존, 헤비유저) 기반 ETA 불만 분석. 둘째, Dead Zone 분석 — 라이더 완료 좌표만으로는 "수요지에서 멀어졌는가"를 판단할 수 없고, 배달 목적지(user_addresses.lat/lng)와의 비교가 있어야 Dead Zone이 정의된다. v3에서 Dead Zone 히트맵을 산출물로 명시했지만, 수요지 좌표 없이는 히트맵 자체가 성립하지 않는 논리적 공백이 있었다. v4에서 이를 메웠다.

---

**② cs_tickets 테이블 (신규)**

CS 처리 이력 테이블이 추가됐다. issue_type, is_eta_related, resolved_at, response_time_min 등의 필드를 포함한다.

**추가 이유:** 이 프로젝트의 분석 출발점이 "CS 실무 경험"이고, 1장 분석 차별점에도 "CS 실무 경험에서 문제 발굴"이라고 명시되어 있다. 그런데 v3의 스키마에는 CS 데이터를 담을 테이블이 없었다. CS 문의량과 Kitchen Delay의 연관성, 고객 이탈과 ETA 지연의 연관성을 정량화하려면 CS 이력이 반드시 필요하다. 분석의 시작점을 데이터로 뒷받침하는 구조가 v4에서 완성됐다.

---

**③ promotions 테이블 (신규) + deliveries.promotion_id 추가**

프로모션/인센티브 테이블이 추가됐고, deliveries에 promotion_id(FK → promotions)가 연결됐다.

**추가 이유:** 가설 4-3-C는 "현재 할증이 수락률에 실제 효과가 있는가"이다. v3에서는 이 가설의 측정 지표로 "수락률 예측 모델 → 피처 중요도"를 명시했지만, 어떤 프로모션이 어떤 배달 건에 적용됐는지 데이터가 없었다. promotions 테이블과 deliveries.promotion_id 연결 없이는 가설 C가 구조적으로 검증 불가능하다. v4에서 가설과 스키마가 비로소 정합성을 갖췄다.

---

**④ menus + order_items 테이블 (신규)**

메뉴 마스터와 주문 상세 트랜잭션 테이블이 추가됐다. menus에는 avg_weight_per_unit, avg_prep_time_min이 포함되고, order_items에는 unit_price(주문 시점 단가), estimated_weight_g가 포함된다.

**추가 이유:** v4의 deliveries에 추가된 total_weight_kg를 계산하려면 메뉴별 무게 정보(menus.avg_weight_per_unit)와 주문 수량(order_items.quantity)이 필요하다. 또한 unit_price를 주문 시점에 스냅샷으로 저장함으로써 메뉴 가격 변동 이력이 보존된다. menus.avg_prep_time_min은 식당 카테고리별 조리시간 파라미터를 실제 메뉴 단위로 더 정밀하게 추정할 수 있게 한다.

---

**⑤ payments 테이블 (신규)**

결제 트랜잭션 테이블이 추가됐다. refund_amount, refunded_at 필드가 포함된다.

**추가 이유:** "Kitchen Delay → 고객 취소 → 환불" 흐름을 추적하려면 결제 및 환불 데이터가 필요하다. v3에서는 orders.cancelled_at조차 없었고(v4에서 추가), payments 테이블도 없어서 취소/환불 흐름 분석이 불가능했다. CS 티켓과 환불 데이터를 연결하면 Kitchen Delay가 실제 이탈·손실로 이어지는 흐름을 정량화할 수 있다.

---

**⑥ deliveries에 eta_gap_min / expected_eta_min / actual_eta_min 추가**

예상 ETA, 실제 ETA, 그 차이(eta_gap_min = actual - expected)가 추가됐다.

**추가 이유:** v3의 분석 핵심 타겟은 "ETA 단축"인데, 정작 "얼마나 지연됐는가"를 측정할 필드가 스키마에 없었다. Total ETA를 각 구간으로 분해하는 것은 마트에서 처리할 수 있지만, 고객에게 안내한 예상 ETA 대비 실제 지연 여부는 주문 단위 원장 데이터로 보존해야 한다. eta_gap_min이 양수면 지연, 음수면 조기 도착으로 분석의 핵심 지표가 된다.

---

**⑦ deliveries에 commission_amount / commission_rate 추가**

배달 건별 수수료 금액과 수수료율이 추가됐다.

**추가 이유:** 가설 4-2-C는 "한집배달 인센티브가 콜사 비용보다 낮아서 콜사가 발생한다"는 수수료 구조 가설이다. v3에서는 이 가설의 측정 지표로 "콜사율 × 배달유형 × 수수료 구간 교차분석"을 명시했지만, 수수료 데이터가 스키마 어디에도 없었다. commission_amount / commission_rate 없이는 가설 C 검증이 구조적으로 불가능하다.

---

**⑧ riders에 call_informed_at / vehicle_type / delivery_status / rating 추가**

배차 요청 알림 시각, 차량 유형, 배달 상태, 평점이 추가됐다.

**추가 이유:** call_informed_at은 가설 4-2-B("완료~다음 콜 간격이 너무 짧아 콜사가 발생한다") 검증에 필수다. 배달 완료 시각(deliveries.completed_at)과 다음 콜 알림 시각(riders.call_informed_at)의 간격이 이 가설의 핵심 측정값인데 v3에는 없었다. vehicle_type은 차량 유형별 배차 가중치 계산 및 total_weight_kg 기반 차량 선정 근거로 활용된다. delivery_status는 실시간 배차 가용 여부 판단에 필요하다.

---

**⑨ restaurants에 lat/lng / operation_status / business_number 등 추가 + 고정/가변 속성 분리**

위경도(Dead Zone 거리 계산용), 영업 상태, 사업자번호, 배달 반경 등이 추가됐고 고정속성과 가변속성이 명시적으로 구분됐다.

**추가 이유:** restaurants.lat/lng는 Dead Zone 분석에서 "라이더 완료 좌표가 다음 주문 수요지(식당)와 얼마나 떨어져 있는가"를 계산하는 데 필요하다. v3에는 식당 위경도가 없어 Dead Zone 거리 계산이 불가능했다. 고정/가변 속성 분리는 2-4절에서 설계한 DW 적재 전략(원장성 테이블의 선분이력 구축)을 스키마 레벨에서 구현하기 위한 것으로, v3에서 문서 상단의 DW 분류와 실제 스키마 사이에 있던 불일치를 해소했다.

---

### 🟡 종합 평가

v3 → v4는 **파이프라인·마트·가설 설계는 그대로 유지하면서 스키마만 전면 개편**한 버전이다. 핵심은 v3에서 가설로는 정의했으나 데이터 구조로 뒷받침되지 않았던 항목들을 스키마 레벨에서 완결한 것이다. 가설 2-B(call_informed_at), 가설 2-C(commission_amount), 가설 3-C(promotions), Dead Zone 분석(user_addresses.lat/lng, restaurants.lat/lng), CS 기반 분석(cs_tickets)이 모두 이에 해당한다. v4의 스키마는 4장에서 정의한 모든 가설을 실제로 검증할 수 있는 구조를 갖췄다.

---

### 🔵 v1→v4 경로에서 소실되었다가 v4.0에서 복원된 항목

v1부터 v4까지 버전을 거치면서 삭제됐으나 분석 논리상 필요하다고 판단해 v4.0에서 복원했다.

---

**① 프로모션 수락률 예측 모델 피처 (4-3절 복원)**

v1, v2에 있었고 v3에서 삭제됐다. v4에서 promotions 테이블과 deliveries.promotion_id가 추가되면서 가설 C의 측정 방법을 구체화하는 이 내용이 오히려 더 필요해졌다. 4-3절 하단에 입력 피처와 출력 구조, promotions 테이블 스키마 연결 주석을 함께 복원했다.

---

**② 마트 2 계약업체 수치화 분석 방향 (8장 복원)**

v1에 있었고 v2부터 소실됐다. "계약업체 vs 자체 라이더 콜사율 비교 → 계약업체 의존도 리스크 정량화"는 이 프로젝트의 구조적 문제(근본 문제: 계약업체 의존도)를 데이터로 연결하는 핵심 분석이다. mart_rider_rejection 설명에 분석 방향과 순환논리 방지 주의사항을 함께 복원했다.

---

**③ 마트 4 ETA 분해 공식 상세 + 분석 순서 정당화 (8장 복원)**

v3 출력본에 추가했으나 v4에 반영되지 않았다. Kitchen Delay를 메인 분석 타겟으로 설정한 것이 CS 경험 기반 가정임을 명시하고, mart_eta_decomposition 선행 실행으로 이를 검증해야 마트 1 심층 분석이 정당화된다는 논리 흐름이 없으면 전체 파이프라인 설계의 근거가 약해진다. ETA 분해 공식 상세와 함께 복원했다.

---

### 🟠 v4 내부 불일치 — 스키마 확장 후 다른 섹션에 미반영된 항목 (v4.0에서 해소)

v4에서 스키마가 대폭 확장됐으나, 마트 설계·KPI·가설 측정 섹션에 반영되지 않아 발생한 불일치들이다.

---

**① eta_gap_min → KPI 9번 추가 (9장)**

deliveries에 eta_gap_min(actual - expected)이 추가됐으나 KPI 8개에 포함되지 않았다. 기존 8개 KPI는 전부 지연의 원인을 측정하는 선행 지표인데, 정작 "얼마나 지연됐는가"라는 최종 결과 지표가 없었다. eta_gap_min 기반 ETA 지연율을 KPI 9번으로 추가해 원인 지표와 결과 지표의 균형을 맞췄다.

---

**② cs_tickets → 마트 교차 분석 연결 명시 (9장)**

cs_tickets 테이블이 생겼고 "분석에서 어디에 쓰이나" 표에도 언급됐으나, 어떤 마트와 어떻게 연결되는지 분석 흐름이 없었다. 9장 KPI 섹션 하단에 cs_tickets와 기존 마트 3개의 교차 분석 방법(조인 방법 포함)을 추가해 독립 테이블이 분석 구조 안에서 활용되는 방식을 명시했다.

---

**③ Dead Zone 분석 데이터 소스 → 마트 2 명시 (8장)**

마트 2에서 Dead Zone 히트맵을 산출물로 명시하고 있으나, v4에서 추가된 user_addresses.lat/lng와 restaurants.lat/lng가 실제로 어떻게 활용되는지 마트 설명에 없었다. 라이더 완료 좌표, 수요지 좌표, 식당 좌표 세 가지를 비교해 Dead Zone을 정의하는 방법을 마트 2 설명에 추가했다.
