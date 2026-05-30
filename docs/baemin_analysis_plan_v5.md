# 배달의민족 라이더 배달료 프라이싱 효율화 분석 프로젝트
**종합 계획서 v5.0 | 2026-05-28**
포지션: 데이터 분석가 포트폴리오 (지원 직군: 라이더배달비서비스팀 PA)
핵심 컨셉: "수락률과 배달품질은 라이더 배달료 프라이싱의 함수다"

---

## 1. Executive Summary

| 항목 | 내용 |
|------|------|
| KPI | 라이더 수락률 향상 + 배달품질 지수 개선 + 라이더 CS율 감소 + 배달료 ROI |
| 메인 문제 | 라이더 배달료 프라이싱의 정적 구조 → 거리·시간대·날씨 미반영 → 수락률 저하 |
| 보조 문제 | 할증 구조 비효율 (악천후 수락률 미개선), 연속배달 피로도와 배달료 간 단절 |
| 근본 문제 | 배달료 책정 알고리즘이 수급 신호를 실시간 반영하지 못함 → 라이더 공급 통제력 약화 |
| 데이터 전략 | 시뮬레이션(메인) + 외부 공개데이터(파라미터 현실 근거) 병행 |
| 분석 차별점 | CS 실무 경험에서 콜사·ETA 지연 패턴 발굴 → 배달료 프라이싱이라는 원인으로 역추적 → ROI 기반 최적화 방향 도출 |

> ⚠ **전제 명시:** 본 분석은 시뮬레이션 데이터로 가설 검증 구조를 설계하고, 분석가로서 '어떤 질문을 해야 하는가'와 '어떤 지표를 어떻게 설계하는가'를 보여주는 것이 목적이다. 인사이트는 '만약 이런 패턴이 실데이터에서 발견된다면'으로 표현한다.

**분석 계층 구조**

콜사율 상승, Dead Zone 발생, ETA 지연, 악천후 수급 Gap은 모두 동일한 원인의 서로 다른 증상이다.

```
[근본 원인 — 본 분석의 메인 타겟]
라이더 배달료 프라이싱 비효율
(거리·시간대·날씨·연속배달을 반영하지 못하는 정적 구조)
                  ↓
[운영 증상 — 본 분석에서 정량화하는 결과 지표]
  콜사율 ↑           → 배달료가 낮으면 라이더가 수락하지 않음
  Dead Zone 발생      → 거리·지역별 배달료 차등 없이는 해소 불가
  ETA 지연            → 콜사 → 재배차 → 픽업 지연의 연쇄
  악천후 수급 Gap     → 할증 구조가 비효율적이면 공급 붕괴 방치
                  ↓
[고객 경험]
배달 ETA 불만 / CS 접수
                  ↓
[비즈니스 손실]
플랫폼 수익성 악화 / 라이더 이탈 / ROI 저하
```

---

## 2. 시장 현황 및 도메인 이해

### 2-1. 활용 외부 공개데이터 및 파라미터 매핑

| 데이터 출처 | 활용 목적 | 뒷받침하는 파라미터 |
|------------|----------|-------------------|
| 공정거래위원회 | 배달앱 시장 규모·성장 추이 | 시장 규모 배경 서술 |
| 모바일인덱스, 와이즈앱 | 플랫폼별 점유율 비교 | 배민의 시장 지위 근거 |
| 통계청, 고용노동부 | 배달 종사자 수, 콜 수 현황 | 라이더 공급 파라미터 규모 검증 |
| DoorDash / Uber Eats 논문 | 배달료 프라이싱이 수락률에 미치는 영향 외부 사례 | 배달료-수락률 탄력성 파라미터 근거 |
| 기상청 공공데이터 | 강수량·풍속 분포 | 악천후 등급 기준, 날씨 파라미터 |
| 배민 기술 블로그 | 배차 알고리즘, 라이더 인센티브 구조 언급 | 배달료 책정 파라미터 방향 참고 |

### 2-2. 도메인 구조 (3자 플랫폼 + 배달료 흐름)

```
고객(User) ──주문──▶ 플랫폼(Baemin) ──배차──▶ 라이더(Rider)
                          │                        │
                   [배달료 책정]            [수락/거절 결정]
                   rider_fee_rules          commission_amount
                          │                        │
                   거리·날씨·시간대          건당 실수익 계산
                   기반 동적 설계           → 수락률 결정
                          │
                    ◀──수락── 식당(Restaurant)
```

**핵심 흐름:** 주문 생성 → 배달료 책정(rider_fee_rules) → 라이더 배차 요청 → 수락/거절 → 식당 수락 → 조리 → 픽업 대기 → 픽업 → 배달 완료 → 배달료 지급

### 2-3. ERD 기반 도메인 테이블 구조

- **출처 명시:** ERD는 erdcloud.com 학습용 클론 ERD(12개 테이블)를 도메인 이해 목적으로 활용. 분석용 OLTP 스키마는 이 ERD를 참고하되 별도 설계한다. (5장 참조)
- **ERD 한계:** 배달원(Rider) 테이블 없음, 배달료 책정 테이블 없음 → 분석용 스키마에서 독자 설계.

### 2-4. DW 관점 테이블 분류

| 테이블 유형 | 대상 테이블 예시 | DW 적재 전략 |
|------------|----------------|-------------|
| 원장성 (Master) | user_master, restaurant_master, rider_master | 시계열 적재 + 선분이력 구축 |
| 거래성 (Transactional) | order_transaction, delivery_log, payment_transaction | 1:1 적재, 날짜 파티셔닝 |
| 양면성 (Duplicity) | restaurant_application, rider_contract | 점이력 구축 |
| 이력 (History) | user_address_history, restaurant_status_history, rider_fee_change_history | 선분/점이력 방식 확정 후 최적화 |

---

## 3. 문제 정의 (3층 구조)

```
[표면 문제] 수락률 저하 + 배달품질 불안정 + 라이더 CS 증가
      ↓
[운영 문제] 배달료가 거리·시간대·날씨를 반영하지 못하는 정적 구조
            → 수익성 낮은 콜 거절 → 재배차 지연 → ETA 증가
            할증이 수급 신호에 후행 → 악천후 공급 붕괴 방치
            연속배달 피로도 누적 → 배달품질 저하 → 라이더 CS 증가
      ↓
[구조 문제] 배달료 책정 알고리즘의 정적 구조
            → 수급 신호 실시간 반영 불가
            → 라이더 행동 예측·통제 불가
            → 계약업체 의존도 → 플랫폼 협상력·통제력 약화
      ↓
[솔루션 방향] 거리·시간대·날씨·피로도를 반영한 동적 배달료 책정
              → 수락률 향상 → 배달품질 개선 → ROI 최적화
```

**배달료 프라이싱을 메인으로 선정한 이유**

1. CS 문의에서 관찰된 콜사·ETA 지연 패턴이 모두 배달료 구조로 역추적됨 (고객 체감 직결)
2. '정성 CS 데이터 → 정량 가설 → 데이터 검증' 스토리가 자연스러움
3. 수락률(선행 지표) → ETA(결과 지표) → CS/ROI(비즈니스 지표)로 솔루션이 명확히 분기됨

> ⚠ **논리보완:** '배달료가 낮아서 콜사가 발생한다'는 주장은 CS 경험 기반 가정이다. 분석 전에 mart_rider_fee에서 commission_amount 구간별 수락률을 먼저 분해 검증해야 논리적 순서가 맞다. → 파이프라인 Step 5에 선행 배치.

---

## 4. 가설 수립 및 측정 지표

### 4-1. 문제 1: 배달료 수준과 수락률

| 가설 | 내용 | 주요 대상 | 측정 지표 |
|-----|------|---------|---------|
| A. 배달료 임계점 | commission_amount가 X원 미만이면 수락률이 급감하는 임계점이 존재한다 | 전체 배달 건 | 배달료 구간별 수락률 (단계함수 탐색) |
| B. 거리 미반영 | 장거리 배달의 commission_amount가 단거리와 큰 차이가 없어 장거리 콜사율이 높다 | 3km+ 배달 건 | 거리 구간별 commission_amount 분포 × 콜사율 |
| C. 차량 유형 불일치 | vehicle_type별 적정 배달료 차등이 없어 특정 차량 유형의 콜사율이 높다 | 차량 유형별 | vehicle_type × commission_rate × 콜사율 교차분석 |

**솔루션 분기**
- 가설 A 확인 → 최소 보장 배달료 임계점 설정 (라이더 공급 안정화)
- 가설 B 확인 → 거리 구간별 배달료 차등화 (Dead Zone 해소 연결)
- 가설 C 확인 → 차량 유형별 배달료 가중치 재설계

| 거리 구간 | 현행 commission_rate 추정 | 콜사 비용 추정 | 수락 유인 여부 |
|---------|------------------------|-------------|-------------|
| 0-1km | 기준 | 낮음 | 수락 |
| 1-3km | 기준+α | 중간 | 조건부 수락 |
| 3km+ | 기준+β | 높음 | 콜사 발생 구간 |

**피크타임:** 점심 11:00–12:30 / 저녁 18:00–19:30

### 4-2. 문제 2: 할증 구조와 악천후 수급

**용어 정의:** '콜사(콜 사절)' = 라이더가 배차 요청(콜)을 거절하는 행위. 콜사 발생 시 재배차 지연 → ETA 직접 증가.

| 가설 | 내용 | 측정 지표 |
|-----|------|---------|
| A. 할증 후행성 | 악천후 할증이 수급 Gap 발생 이후에 적용되어 수락률 회복이 늦다 | 날씨 등급 변화 시점 vs 할증 적용 시점 간격 × 수락률 회복 속도 |
| B. 할증 수준 부족 | 현재 할증 금액이 라이더의 악천후 기회비용보다 낮아 수락률 개선 효과가 없다 | 날씨 등급별 incentive_amount × 수락률 변화율 (탄력성) |
| C. 할증 ROI | 할증 금액 증가 대비 수락률 상승폭이 감소하는 수확체감 구간이 존재한다 | 할증 금액 구간별 수락률 한계 상승분 |

**프로모션 수락률 예측 모델 피처 (가설 B/C 측정 방법)**

```
[입력 피처]
날씨 강도, 배달 거리, 완료 후 콜 밀도, 할증 금액,
연속 배달 수, 시간대, 배달 유형 (한집/알뜰), 차량 유형

[출력]
수락률 (0~1) → 피처 중요도 도출 → 동적 배달료 설계 근거
```

> 스키마 연결: promotions.incentive_amount / incentive_rate가 할증 금액 피처의 데이터 소스. rider_fee_rules.dynamic_rate와 비교해 현행 할증 구조의 갭을 측정한다.

| 날씨 등급 | 주문량 배수 | 라이더 공급 배수 | 수급 Gap | 현행 할증 여부 |
|---------|-----------|--------------|---------|-------------|
| 맑음 | 1.0 | 1.0 | 0 | 없음 |
| 흐림 | 1.1 | 0.95 | -0.15 | 없음 |
| 비 | 1.3 | 0.75 | -0.55 | 일부 적용 |
| 폭우/눈 | 1.5 | 0.5 | -1.0 | 적용 (효과 미검증) |

> ⚠ CS 문의량 체감 3배 / 주문량 1.5배는 보수적 추정 → 실데이터 확보 시 재검증 명시

### 4-3. 문제 3: 피로도와 배달품질

| 가설 | 내용 | 측정 지표 |
|-----|------|---------|
| A. 피로도-품질 저하 | consecutive_deliveries N건 이상이면 배달품질(ETA 지연, CS 발생)이 급격히 악화된다 | consecutive_deliveries 구간별 eta_gap_min 분포 × CS 발생률 |
| B. 피로도-콜사 | 연속배달 N건 이후 콜사율이 급증하고, 이때 배달료 인상이 콜사를 억제하는가 | consecutive_deliveries × 콜사율 × commission_amount 3방향 교차분석 |
| C. 라이더 CS 유형 | 라이더 CS의 주요 유형(배달지연/배달사고/미배달)과 피로도·배달료 간의 상관관계 | cs_tickets.is_rider_fault × riders.consecutive_deliveries × commission_amount |


---

## 5. 데이터 전략

### 5-1. 분석용 OLTP 스키마

> **출처 구분:** ERD(erdcloud)와 무관하게 분석 목적으로 독자 설계. 실제 배민 DB 구조와 다를 수 있으며, 포트폴리오 시뮬레이션용임을 명시.

```sql

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
  lat                    FLOAT        -- [추가] 위도 (Dead Zone 분석·거리 계산)
  lng                    FLOAT        -- [추가] 경도
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
  promotion_id           INT          -- [추가] FK → promotions (할증 적용 건 식별)
  fee_rule_id            INT          -- [추가] FK → rider_fee_rules (적용된 배달료 규칙)
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
  rejection_reason       VARCHAR      -- 거리 / 시간 / 날씨 / 배달료
  base_fee               INT          -- [추가] 기본 배달료 (할증 전)
  dynamic_surcharge      INT          -- [추가] 동적 할증 금액 (날씨·시간대·거리 반영)
  commission_amount      INT          -- [추가] 해당 배달 수수료 금액 (가설 1-A/B/C 핵심)
  commission_rate        FLOAT        -- [추가] 수수료율 (배달유형별 비교)
  total_fee              INT          -- [추가] base_fee + dynamic_surcharge = 라이더 실수령
  expected_eta_min       INT          -- [추가] 주문 수락 시 고객에게 안내한 예상 소요시간(분)
  actual_eta_min         INT          -- [추가] 실제 총 소요시간 = Total ETA (completed_at - order_accepted_at)
  eta_gap_min            INT          -- [추가] actual - expected (양수=지연 / 음수=조기도착 / 분석 핵심 타겟)
  delivery_quality_score FLOAT        -- [추가] 배달품질 종합 점수 (ETA 준수율, CS 미발생 등)


riders [수정] ← rider_master
  -- [고정 속성 - 마스터 불변값]
  rider_id               INT          -- PK
  registered_at          TIMESTAMP    -- [추가] 등록일 (고정)
  -- [가변 속성 - 운영 중 변경 가능]
  rider_type             VARCHAR      -- 자체 / 계약업체
  vehicle_type           VARCHAR      -- [추가] 자전거/오토바이/보도/자동차 (배차 가중치 계산)
  activity_region        VARCHAR      -- [추가] 활동 지역 (가변 - 이동 가능)
  status                 VARCHAR      -- [추가] 활성/비활성/정지 (라이더 계정 상태, 가변)
  delivery_status        VARCHAR      -- [추가] 대기중/배달중/휴식/오프라인 (가변)
  rating                 FLOAT        -- [추가] 라이더 평점 (가변 - 누적 갱신)
  consecutive_deliveries INT          -- 연속 배달 수 (피로도 proxy)
  current_lat            FLOAT
  current_lng            FLOAT
  call_informed_at       TIMESTAMP    -- [추가] 배차 요청 알림 시각 (가설 4-2-B: 완료→다음콜 간격 핵심)
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
  rider_id               INT          -- [추가] FK → riders (라이더 귀책 CS 식별)
  issue_type             VARCHAR      -- ETA지연/음식불량/배달오류/취소요청/라이더불만
  issue_detail           TEXT
  is_eta_related         BOOLEAN      -- ETA 관련 여부 (분석 핵심 필터)
  is_rider_fault         BOOLEAN      -- [추가] 라이더 귀책 여부 (배달료-품질 연관 분석 핵심)
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


rider_fee_rules [신규] — 배달료 책정 규칙 (프라이싱 정책 이력)
  rule_id                INT          -- PK
  rule_name              VARCHAR      -- 규칙 식별명 (예: 기본요금_v1, 악천후할증_v2)
  distance_band          VARCHAR      -- 0-1km / 1-3km / 3-5km / 5km+
  time_band              VARCHAR      -- 피크(11-13, 18-21) / 비피크 / 심야(23-06)
  weather_grade          VARCHAR      -- 맑음 / 흐림 / 비 / 폭우 / 눈 (NULL = 상시)
  vehicle_type           VARCHAR      -- 자전거 / 오토바이 / 자동차 (NULL = 전체)
  base_fee               INT          -- 기본 배달료
  dynamic_rate           FLOAT        -- 동적 할증률 (수급 Gap 연동)
  max_surcharge          INT          -- 최대 할증 상한
  is_active              BOOLEAN
  start_at               TIMESTAMP
  end_at                 TIMESTAMP    -- NULL = 현재 적용 중
  created_by             VARCHAR      -- 규칙 생성 주체 (정책 변경 이력 추적)
```

**분석에서 어디에 쓰이나?**

| 분석 목적 | 활용 테이블 |
|---------|-----------|
| 배달료 구간별 수락률 분석 | deliveries.commission_amount + deliveries.is_rejected |
| 거리별 배달료 차등 효과 | deliveries.distance_km + deliveries.base_fee + deliveries.is_rejected |
| 할증 효율 측정 | promotions.incentive_amount + deliveries.is_rejected + weather.weather_grade |
| 피로도-품질 연관성 | riders.consecutive_deliveries + deliveries.delivery_quality_score + cs_tickets.is_rider_fault |
| 배달료 ROI | deliveries.total_fee + deliveries.eta_gap_min + cs_tickets.is_rider_fault |
| 동적 배달료 규칙 효과 비교 | rider_fee_rules × deliveries (fee_rule_id) 시나리오별 수락률·품질 비교 |
| Dead Zone: 수요지 vs 라이더 완료 좌표 | user_addresses.lat/lng + deliveries.dropoff_lat/lng + restaurants.lat/lng |
| 신규 vs 기존 고객의 ETA 불만 차이 | users.signup_at + cs_tickets |
| 고객 이탈 - ETA 지연 연관성 | users.last_order_at + deliveries.eta_gap_min |
| 활성 라이더 공급 현황 | riders.status + riders.delivery_status + riders.activity_region |


---

## 6. 올바른 분석 프로세스 흐름

> ⚠ **설계 원칙:** DB/파이프라인(그릇)이 먼저 구축되어야 한다. 외부 리서치 데이터는 시뮬레이션 생성 이후 파라미터 검증·보정 용도로 활용한다. 둘 다 feature/02-data-collection 브랜치에서 처리한다.

| Step | 작업 내용 | Git 브랜치 | Airflow 역할 |
|------|----------|-----------|------------|
| Step 1 | 도메인 분석 + 문제 정의 + 가설 문서화 | feature/01-domain-analysis | - |
| Step 2 | [DB/인프라] Docker Compose 작성 + PostgreSQL 컨테이너 + OLTP DDL 작성 및 적재 | feature/02-data-collection | - (1회성 셋업, 자동화 불필요) |
| Step 3 | [시뮬레이션] Python 스크립트로 orders / restaurants / deliveries / riders / weather / rider_fee_rules 다중 시나리오 생성 및 OLTP 적재 (1차) | feature/02-data-collection | - (1회성 실행) |
| Step 4 | [외부 데이터] 기상청·통계청 공개데이터 수집 → 시뮬레이션 파라미터와 비교 검증 → 필요시 파라미터 조정 후 재적재 | feature/02-data-collection | O [DAG-1] 외부 API 스케줄 수집 DAG (오케스트레이션: 수집→검증→저장 순서를 자동 스케줄링) |
| Step 5 | [선행검증 EDA] commission_amount 구간별 수락률 분해 쿼리 → 배달료 임계점 탐색 + 탐색적 분석 | feature/03-eda | - |
| Step 6 | [마트 ETL] OLTP → 4개 데이터 마트 변환·적재 | feature/04-modeling | O [DAG-2] 마트 ETL 오케스트레이션 DAG (오케스트레이션: 마트 간 의존 순서 보장 - rider_fee/rider_rejection/weather → fee_roi) |
| Step 7 | [분석] KPI 지표 9개 쿼리 + 가설 A/B/C 검증 | feature/04-modeling | - |
| Step 8 | [보고] seaborn / folium 시각화 + Tableau 대시보드 + 포트폴리오 9챕터 정리 | feature/05-reporting | - |

**Airflow 역할 상세**

| DAG 위치 (브랜치) | 트리거 방식 | 태스크 구성 |
|-----------------|-----------|-----------|
| DAG-1: 외부 데이터 수집 — feature/02-data-collection | 스케줄 (@daily 또는 1회 실행) | ① 기상청 API 호출 → ② 통계청 파일 다운로드 → ③ 파라미터 비교 테이블 생성 → ④ PostgreSQL 적재 |
| DAG-2: 마트 ETL — feature/04-modeling | 수동 트리거 or @once | ① mart_rider_fee → ② mart_rider_rejection → ③ mart_weather_supply_gap → ④ mart_fee_roi (순서 의존성 보장) |

> 💡 **오케스트레이션(Orchestration)이란:** 여러 태스크(작업)의 실행 순서, 의존성, 재시도, 스케줄을 중앙에서 자동으로 관리하는 것. 예: 마트 A가 완료된 후에만 마트 B를 실행 → Airflow DAG가 이 순서를 보장해준다.

---

## 7. Git 브랜치 전략

**저장소:** baemin_rider_fee_optimization | **현재 작업:** feature/01-domain-analysis

```
master
├── feature/01-domain-analysis   ← [현재] 도메인 이해, 문제 정의
├── feature/02-data-collection   ← 스키마 설계, Docker + DDL,
│                                   시뮬레이션 생성 & 적재 (rider_fee_rules 다중 시나리오 포함),
│                                   외부 리서치 수집 (Airflow DAG-1),
│                                   파라미터 검증 & 조정
├── feature/03-eda               ← 배달료-수락률 선행검증, 탐색 분석
├── feature/04-modeling          ← 마트 ETL (Airflow DAG-2), KPI 쿼리, 가설 검증
└── feature/05-reporting         ← 시각화, 대시보드, 포트폴리오 정리
```

**브랜치 핵심 산출물**

| 브랜치 | 핵심 산출물 |
|--------|-----------|
| feature/01-domain-analysis | 계획서 v5.0, ERD 분석 노트, 문제 정의 문서 |
| feature/02-data-collection | docker-compose.yml, ddl.sql, simulate_data.py, simulate_fee_rules.py, airflow/dag_external_collect.py |
| feature/03-eda | eda_notebook.ipynb, fee_acceptance_analysis.sql |
| feature/04-modeling | mart_*.sql, airflow/dag_mart_etl.py, hypothesis_test.ipynb |
| feature/05-reporting | dashboard.twb, charts/, README.md (포트폴리오 최종) |

> 💡 **브랜치 병합 규칙:** 각 브랜치 작업 완료 후 master로 PR → 머지. master는 항상 완성 상태만 유지.

---

## 8. 데이터 마트 설계 (4개)

```
mart_rider_fee          ─┐
mart_rider_rejection    ─┼──▶ mart_fee_roi ──▶ Tableau 대시보드
mart_weather_supply_gap ─┘
```

### 마트 1: mart_rider_fee

| 항목 | 내용 |
|-----|------|
| 목적 | 배달료 구조 분석 → 수락률 임계점 도출 + 거리·시간대·차량 유형별 최적 배달료 설계 근거 (가설 1-A/B/C 검증) |
| Grain | 배달 건 × 거리 구간 × 시간대 × 날씨 등급 |
| 핵심 지표 | 배달료-수락률 탄력성, 거리 구간별 콜사율, 차량 유형별 수락률, 배달료 임계점, 건당 실수익 |
| 핵심 공식 | 배달료 탄력성 = (수락률 변화율) / (commission_amount 변화율) \| 건당 실수익 = total_fee - 이동비용 추정(distance_km × 단위비용) |

> ⚠️ **분석 순서 정당화**: 이 마트는 파이프라인 Step 5(선행검증 EDA)에서 가장 먼저 운영한다. 배달료가 수락률의 주된 결정 요인이라는 주장은 CS 경험 기반 가정이므로, mart_rider_fee에서 commission_amount 구간별 수락률 분포를 먼저 확인해야 이후 마트들의 심층 분석이 정당화된다. 탄력성이 낮다면 배달료 외 다른 요인(Dead Zone, 피로도)을 메인 타겟으로 재설정해야 한다.

### 마트 2: mart_rider_rejection

| 항목 | 내용 |
|-----|------|
| 목적 | 콜사 패턴 분석 + Dead Zone 탐지 → 거리·지역별 배달료 차등화 근거 도출 + 계약업체 의존도 수치화 (가설 1-B/C, 2-A 검증) |
| Grain | 라이더 × 완료 좌표(격자) × 배달 유형 |
| 핵심 지표 | 콜사율, Dead Zone 발생률, 라이더 유형별 콜사율, 피로도-콜사율, 거리 구간별 콜사율 |
| 시각화 | folium Dead Zone 히트맵 + 배달료 구간 오버레이 |

**계약업체 의존도 수치화 분석 방향**

```
계약업체 라이더 콜사율  vs  자체 라이더 콜사율
→ 격차가 크다면: 계약업체 의존도 리스크의 정량적 근거
→ riders.rider_type 기준으로 분리 집계
→ commission_amount 구간별 격차 변화 → 배달료가 계약업체 콜사율에 미치는 영향
```

> ⚠ 시뮬레이션에서 두 유형의 콜사율 파라미터를 동일하게 설정하거나 다중 시나리오로 분석한다. 결론은 "만약 실데이터에서 이 격차가 관찰된다면"으로 프레임을 전환한다. (순환논리 방지 — 13장 결함 #1 참조)

**Dead Zone 분석 데이터 소스**

```
라이더 완료 좌표: deliveries.dropoff_lat / dropoff_lng
수요지 좌표:     user_addresses.lat / lng
식당 좌표:       restaurants.lat / lng

→ 라이더 완료 지점과 인근 수요지(배달 목적지, 식당) 밀집도를 비교해
   수요가 적은 지역에 고립된 라이더를 Dead Zone으로 정의한다.
→ Dead Zone 발생 지역의 평균 commission_amount를 교차 분석해
   배달료 차등화로 Dead Zone을 해소할 수 있는지 검증한다.
```

### 마트 3: mart_weather_supply_gap

| 항목 | 내용 |
|-----|------|
| 목적 | 악천후 수급 불균형 + 할증 효율 분석 → 수급 연동 동적 할증 구조 설계 근거 (가설 2-A/B/C 검증) |
| Grain | 시간대 × 날씨 등급 × 지역 |
| 핵심 지표 | 수급 Gap, 날씨 등급별 콜사율, 할증 효율 지수, 할증 ROI |
| 핵심 공식 | 수급 Gap = (실제 주문 수 / 평시 평균) - (실제 라이더 공급 수 / 평시 평균) \| 할증 효율 지수 = 수락률 상승폭 / incentive_amount 증가폭 |

### 마트 4: mart_fee_roi (통합 마트)

| 항목 | 내용 |
|-----|------|
| 목적 | 배달료 프라이싱 개선의 ROI 통합 검증 → 수락률·품질·비용 트레이드오프 분석 + Executive Summary 소스 |
| Grain | 주문 단위 |
| 핵심 공식 | ROI = (수락률 개선 효과 × 건수 × 건당 마진) / 배달료 인상 총비용 |
| 역할 | 3개 마트 인사이트의 배달료 개선 효과 최종 검증 + Executive Summary 소스 |

**ROI 분해 공식 상세**

```
Total ETA = Kitchen Wait Time       (order_accepted_at → pickup_ready_at)
          + Pickup Wait Time        (pickup_ready_at → 라이더 픽업 완료)
          + Last-mile Time          (픽업 완료 → 배달 완료)

ETA 지연 측정 = eta_gap_min (actual_eta_min - expected_eta_min)
              양수 = 지연 / 음수 = 조기 도착

배달료 프라이싱 ROI =
  [수락률 개선으로 절감된 재배차 비용]
  + [ETA 개선으로 감소한 CS 처리 비용]
  + [배달품질 향상으로 증가한 재주문율 × 건당 마진]
  - [배달료 인상 총비용]

개선 효과 측정 기준:
  수락률 개선 → 콜사율 감소폭 (mart_rider_rejection)
  ETA 개선   → eta_gap_min 감소폭 (deliveries)
  품질 개선  → CS 접수율 감소폭 (cs_tickets.is_rider_fault)
```

> ⚠️ **분석 순서 정당화**: mart_rider_fee에서 배달료-수락률 탄력성이 확인된 후 이 마트의 ROI 분석이 정당화된다. 탄력성이 미미하다면 ROI 계산의 전제가 무너지므로 분석 방향을 재설정해야 한다.

---

## 9. 핵심 KPI 지표 9개

| # | 지표명 | 원천 마트 | 선정 이유 |
|---|-------|---------|---------|
| 1 | 배달료-수락률 탄력성 | mart_rider_fee | 배달료 설계의 핵심 파라미터 — 가설 1-A 검증 핵심 |
| 2 | 거리 구간별 콜사율 | mart_rider_fee | 거리 차등화 배달료 설계 근거 — 가설 1-B 검증 핵심 |
| 3 | 할증 효율 지수 | mart_weather_supply_gap | 현행 할증 구조의 실효성 측정 |
| 4 | 수급 Gap | mart_weather_supply_gap | 악천후 배달료 비효율의 직접 원인 |
| 5 | Dead Zone 발생률 | mart_rider_rejection | 지역별 배달료 차등화 필요성의 선행 지표 |
| 6 | 피로도-품질 상관계수 | mart_rider_rejection | 연속배달 인센티브 설계 근거 — 가설 3-A 검증 핵심 |
| 7 | 라이더 CS율 | mart_fee_roi | JD 핵심 KPI — 배달품질의 최종 결과 지표 |
| 8 | ETA 지연율 (eta_gap_min > 0 비율) | mart_fee_roi | 배달료 개선 효과의 결과 지표 |
| 9 | 배달료 프라이싱 ROI | mart_fee_roi | JD 명시 KPI — 프라이싱 정책 변경의 비용·효과 통합 측정 |

> **KPI #9 추가 이유:** 기존 KPI들은 수락률·콜사율 등 운영 지표 중심이다. ROI는 "배달료를 얼마나 올려야 수락률이 얼마나 오르고, 그 비용이 수익 개선보다 작은가"라는 의사결정의 최종 기준이 된다.

**cs_tickets 분석 연결**

cs_tickets는 독립 마트를 갖지 않고 아래 방식으로 기존 마트와 교차 분석한다.

| 분석 목적 | 연결 마트 | 조인 방법 |
|---------|---------|---------|
| 라이더 귀책 CS → 피로도 연관성 | mart_rider_rejection | rider_id + consecutive_deliveries |
| 배달료 구간별 CS 발생 패턴 | mart_rider_fee | delivery_id → commission_amount |
| 악천후 시 라이더 CS 폭증 | mart_weather_supply_gap | recorded_at 시간 조인 + is_rider_fault |
| ETA 지연 → 고객 이탈 연관성 | mart_fee_roi | order_id → user_id → users.last_order_at |

---

## 10. 기술 스택

| 도구 | 역할 | 적용 브랜치 |
|-----|------|-----------|
| Docker | 환경 세팅 / PostgreSQL + Python 컨테이너 | feature/02 |
| PostgreSQL | OLTP 저장 + 데이터 마트 저장 | feature/02, 04 |
| Python | 시뮬레이션 데이터 생성 / ETL 스크립트 / 분석 시각화 | feature/02, 03, 04 |
| Apache Airflow | DAG-1 외부 데이터 수집 스케줄링 \| DAG-2 마트 ETL 오케스트레이션 | feature/02, 04 |
| SQL | 데이터 조회 / KPI 지표 쿼리 / 마트 생성 | feature/03, 04 |
| seaborn / folium | 분석 시각화 + Dead Zone 히트맵 + 배달료 구간 오버레이 | feature/05 |
| Tableau | 최종 대시보드 (배달료팀 운영 모니터링 + 경영진 보고용) | feature/05 |

---

## 11. 솔루션 방향

| 시기 | 솔루션 | 근거 마트 |
|-----|-------|---------|
| 단기 | 배달료 임계점 기반 최소 보장 배달료 설정 | mart_rider_fee |
| 단기 | 악천후 할증 선행 적용 (수급 Gap 발생 전 예측 트리거) | mart_weather_supply_gap |
| 중기 | 거리 구간별 배달료 차등화 → Dead Zone 해소 | mart_rider_rejection |
| 중기 | 연속배달 N건 이후 피로도 인센티브 구조 설계 | mart_rider_rejection |
| 장기 | 거리·시간대·날씨·피로도 통합 반영 동적 배달료 책정 시스템 | mart_fee_roi |
| 장기 | 자체 라이더 비율 확대 + 배달료 내재화 → 계약업체 의존도 탈피 | 구조적 문제 정량 근거 |

---

## 12. 포트폴리오 목차 (9챕터)

1. Executive Summary
2. 시장 현황 분석 (외부 공개데이터 활용)
   - 2-1. 시장 규모 및 성장 추이 (공정위)
   - 2-2. 플랫폼별 점유율 비교 (모바일인덱스, 와이즈앱)
   - 2-3. 라이더 공급 현황 (통계청, 고용노동부)
   - 2-4. 배달료 프라이싱이 수락률에 미치는 영향 (DoorDash/Uber Eats 논문)
3. 문제 정의 (3층 구조)
4. 가설 수립 및 측정 설계
5. 데이터 설계 및 수집
   - 5-1. 분석용 OLTP 스키마
   - 5-2. 외부 공개데이터 파라미터 근거
   - 5-3. 시뮬레이션 데이터 생성 방법론
6. 분석
   - 6-1. 배달료-수락률 분해 (선행 검증)
   - 6-2. 배달료 임계점 및 거리 차등화 분석 (가설 1-A/B/C)
   - 6-3. 할증 구조 효율 분석 (가설 2-A/B/C)
   - 6-4. 피로도-배달품질 분석 (가설 3-A/B/C)
7. 인사이트 (So what? 관점)
8. 솔루션 제안 (단기/중기/장기)
9. 한계 및 후속 과제

---

## 13. 논리 검토 및 한계 명시

### 13-1. 발견된 논리적 결함 및 해결 방향

| # | 결함 유형 | 내용 | 해결 방향 |
|---|---------|------|---------|
| 1 | 순환논리 | 시뮬레이션에서 배달료 낮은 구간의 콜사율을 높게 설정 → 그 데이터로 '배달료가 낮아서 콜사가 발생한다'를 검증하는 건 순환논리 | 배달료-콜사율 파라미터를 독립적으로 설정하거나 다중 시나리오. 결론은 '만약 실데이터에서…' 프레임으로 전환 |
| 2 | 검증 순서 역전 | 배달료가 수락률의 주된 결정 요인이라고 전제하고 분석 | 파이프라인 Step 5에 배달료-수락률 분해 선행 배치 (v5.0 반영 완료) |
| 3 | 임계값 근거 부재 | 최소 보장 배달료 임계점, 피로도 N건 기준, Slot Congestion 임계값 설정 근거 없음 | CS 경험 기반임을 명시 + 민감도 테스트(sensitivity test) 수행 |
| 4 | ERD-스키마 출처 혼용 | 학습용 ERD를 실제 배민 구조인 것처럼 오해할 소지 | ERD는 도메인 이해 목적, 분석 스키마는 별도 설계 명시 (2장/5장 처리) |
| 5 | 외부 데이터 매핑 불완전 | 외부 데이터가 어떤 파라미터를 뒷받침하는지 연결 고리 미흡 | 시뮬레이션 파라미터 설정 시 각 수치의 근거 출처를 테이블로 명시 |
| 6 | ROI 계산 단순화 | 배달료 인상 비용과 수락률 개선 효과를 선형 관계로 가정 | 수확체감 구간 탐색 (가설 2-C) + 비선형 모델 적용 검토 명시 |
| 7 | rider_fee_rules 현실성 | 실제 배민의 배달료 책정 규칙은 외부에 공개되지 않음 | rider_fee_rules는 시뮬레이션 설계 목적임을 명시. 실데이터에서는 commission_amount 분포 역추적으로 규칙을 추정하는 방식이 현실적 대안임을 명시 |

### 13-2. 구조적 한계

**한계 1: 시뮬레이션 데이터의 근본 한계**
파라미터 설정에 따라 결론이 달라질 수 있음. '분석 구조 설계 능력'을 보여주는 것이 목적임을 명확히 할 것.

**한계 2: 배달료-수락률 인과관계 식별 어려움**
배달료 외에 날씨·시간대·피로도 등 다른 요인이 동시에 수락률에 영향을 준다. 통제변수 설정 없이는 배달료의 단독 효과를 분리하기 어려움 → 다변량 분석 또는 A/B 테스트 설계 검토 필요.

**한계 3: Dead Zone 공간 분석**
단순 랜덤 좌표 사용 시 신뢰도 저하. 서울 행정동 경계 공공데이터로 현실적인 좌표 분포 구성 권장.

**한계 4: 계약업체 의존도 단일 지표 부재**
정성적 개념 → mart_rider_rejection에서 복수 지표(콜사율, 피로도, 배달시간, commission_amount 구간별 격차)로 보완 필요.

---

## 14. 다음 액션 플랜

| 순서 | 내용 | 브랜치 |
|-----|------|-------|
| Step 1 | 도메인 분석 완료 + 문제 정의 문서화 + 계획서 v5.0 커밋 | feature/01-domain-analysis |
| Step 2 | Docker Compose 작성 (PostgreSQL + Python 컨테이너) + OLTP DDL 작성 | feature/02-data-collection |
| Step 3 | Python 시뮬레이션 스크립트 작성 (orders, restaurants, deliveries, riders, weather, rider_fee_rules 다중 시나리오) | feature/02-data-collection |
| Step 4 | PostgreSQL 스키마 DDL + 시뮬레이션 데이터 1차 적재 | feature/02-data-collection |
| Step 5 | 외부 공개데이터 수집 (기상청, 통계청) + Airflow DAG-1 작성 + 파라미터 검증 및 조정 | feature/02-data-collection |
| Step 6 | 배달료-수락률 분해 쿼리 → 임계점 선행 확인 + EDA | feature/03-eda |
| Step 7 | ETL + 데이터 마트 4개 구축 + Airflow DAG-2 작성 | feature/04-modeling |
| Step 8 | KPI 쿼리 + 가설 A/B/C 검증 분석 + ROI 계산 | feature/04-modeling |
| Step 9 | seaborn / folium 시각화 + Tableau 대시보드 + 포트폴리오 9챕터 최종 정리 | feature/05-reporting |

---

> 💡 **본 계획서는 Living Document입니다.** 분석 진행에 따라 섹션을 업데이트하고 버전을 관리합니다. (현재: v5.0)
