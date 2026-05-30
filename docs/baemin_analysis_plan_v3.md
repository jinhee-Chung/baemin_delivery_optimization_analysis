# 배달의민족 배달 최적화 분석 프로젝트
**종합 계획서 v3.0 | 2026-05-28**
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

---

## 5. 데이터 전략

### 5-1. 분석용 OLTP 스키마

> **출처 구분:** ERD(erdcloud)와 무관하게 분석 목적으로 독자 설계. 실제 배민 DB 구조와 다를 수 있으며, 포트폴리오 시뮬레이션용임을 명시.

```sql
orders
  order_id              INT
  restaurant_id         INT
  order_accepted_at     TIMESTAMP  -- 식당 주문 수락 시각
  cooking_started_at    TIMESTAMP  -- 조리 시작 시각
  pickup_ready_at       TIMESTAMP  -- 픽업 준비 완료 시각
  order_type            VARCHAR    -- 한집 / 알뜰
  is_converted_offline  BOOLEAN    -- 오프라인→온라인 전환 추정
  created_at            TIMESTAMP

restaurants
  restaurant_id         INT
  category              VARCHAR    -- 한식 / 치킨 / 피자 / 분식
  region                VARCHAR
  avg_cooking_time      INT
  max_capacity          INT        -- 동시 처리 가능 주문 수 (오버수락 측정 핵심)

deliveries
  delivery_id           INT
  order_id              INT
  rider_id              INT
  assigned_at           TIMESTAMP
  pickup_at             TIMESTAMP
  completed_at          TIMESTAMP
  pickup_lat/lng        FLOAT
  dropoff_lat/lng       FLOAT
  distance_km           FLOAT
  is_rejected           BOOLEAN
  rejection_reason      VARCHAR    -- 거리 / 시간 / 날씨

riders
  rider_id                  INT
  rider_type                VARCHAR  -- 자체 / 계약업체
  consecutive_deliveries    INT      -- 연속 배달 수 (피로도 proxy)
  current_lat/lng           FLOAT

weather
  weather_id          INT
  recorded_at         TIMESTAMP
  precipitation       FLOAT    -- 강수량 (mm)
  wind_speed          FLOAT    -- 풍속 (m/s)
  weather_grade       VARCHAR  -- 맑음 / 흐림 / 비 / 폭우 / 눈
  is_adverse          BOOLEAN
  demand_multiplier   FLOAT
  supply_multiplier   FLOAT
```

**테이블 관계**
- restaurants 1:N orders
- orders 1:1 deliveries
- riders 1:N deliveries
- weather 1:N deliveries (recorded_at 시간 조인)

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
```

> ⚠️ **분석 순서 정당화**: 이 마트는 파이프라인 Step 5(선행검증 EDA)에서 가장 먼저 운영한다.
> Kitchen Delay(마트 1)를 메인으로 설정한 것은 현재 CS 경험 기반 가정이므로,
> mart_eta_decomposition에서 Kitchen Wait Time이 Total ETA에서 차지하는 비중이
> 가장 크다는 것이 먼저 확인되어야 마트 1의 심층 분석이 논리적으로 정당화된다.
> 반대로 Pickup Wait Time이나 Last-mile Time이 더 크게 나온다면,
> 분석 메인 타겟을 재설정해야 한다.

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

> 💡 **본 계획서는 Living Document입니다.** 분석 진행에 따라 섹션을 업데이트하고 버전을 관리합니다. (현재: v3.0)

---

## v2 → v3 변경 분석

> **비교 기준**: v2 = `baemin_plan_v2.md` (종합 계획서 v1.0) / v3 = `baemin_plan_v3.md` (종합 계획서 v3.0)

---

### 🔴 v2에 있었으나 v3에서 삭제된 항목

---

**① 문서 상단 목차 섹션**

v2에는 1~11장 링크가 달린 목차가 문서 최상단에 있었다.

**삭제 이유:** v3는 챕터 수가 14개로 늘어났고, Living Document로 구조가 계속 바뀌는 특성상 목차를 별도로 유지하면 섹션 변경 시마다 목차도 함께 수정해야 하는 관리 부담이 생긴다. 깃허브 등 마크다운 렌더러에서는 헤딩 기반 자동 목차가 제공되므로 수동 목차는 실익이 적다.

---

**② 분석 파이프라인의 브랜치 구조 전면 개편**

v2의 브랜치 구조는 인프라·구현 중심이었다.

```
feature/01-infra-setup       ← Docker + PostgreSQL
feature/02-data-generation   ← 시뮬레이션 + OLTP 적재
feature/03-eda
feature/04-data-mart
feature/05-analysis
feature/06-visualization
```

v3에서는 분석 단계 중심으로 재편됐다.

```
feature/01-domain-analysis
feature/02-data-collection
feature/03-eda
feature/04-modeling
feature/05-reporting
```

**변경 이유:** v2의 브랜치명은 기술 작업 단위(infra-setup, data-generation)로 나뉘어 있어, 같은 브랜치(feature/02)에 시뮬레이션 생성과 OLTP 저장이 함께 섞이고 외부 데이터 수집 단계가 누락되어 있었다. v3는 분석 흐름 단위(domain-analysis, data-collection, modeling, reporting)로 재편해 각 브랜치의 책임 범위를 명확히 하고, 외부 데이터 수집과 파라미터 검증 단계를 feature/02-data-collection에 통합했다. 브랜치 수도 6개 → 5개로 줄어 관리가 단순해졌다.

---

**③ 프로모션 수락률 예측 모델 피처 설명 (4-3절)**

v2에는 악천후 수급 분석 가설 C(프로모션 효과) 아래에 예측 모델의 입력 피처와 출력을 설명하는 블록이 있었다.

```
[입력 피처] 날씨 강도, 배달 거리, 완료 후 콜 밀도, 할증 금액,
            연속 배달 수, 시간대, 배달 유형 (한집/알뜰)
[출력] 수락률 (0~1) → 피처 중요도 → 프로모션 설계 근거
```

**삭제 이유:** 모델 설계 상세는 feature/04-modeling 브랜치의 분석 단계에서 다루는 것이 구조상 맞다는 판단으로 보인다. 그러나 이 피처 목록은 가설 C("현재 할증이 수락률에 실제 효과가 있는가")의 측정 방법을 구체화하는 내용이므로, 가설 설계 단계인 4장에 있는 것이 논리적으로 더 자연스럽다. 향후 복원을 검토할 필요가 있다.

---

**④ 핵심 지표 8개가 마트 설계 섹션 내에 통합 배치 (v2의 7장 구조)**

v2에서는 데이터 마트 설계(7장) 안에 "핵심 지표 8개 선정" 소섹션이 포함되어 있었다.

**분리 이유:** v3에서는 핵심 KPI 지표 8개를 9장으로 독립시켰다. 마트 설계는 ETL 구조(목적·Grain·공식)를 다루고, KPI 선정은 분석 관점에서 "무엇을 측정할 것인가"를 다루는 별개 레이어다. 섹션을 분리하면 각각의 목적이 명확해지고, 이후 KPI가 변경될 때 마트 설계에 영향 없이 독립적으로 수정할 수 있다.

---

**⑤ 기술 스택이 분석 파이프라인 섹션 내에 포함 (v2의 6장 구조)**

v2에서 기술 스택은 6장(분석 파이프라인) 안의 소섹션이었다.

**분리 이유:** v3에서는 10장으로 독립시키고 "적용 브랜치" 열을 추가했다. 기술 스택은 파이프라인 실행 방법이 아니라 프로젝트 전반에서 사용하는 도구 목록이므로, 독립 섹션으로 분리하는 것이 문서 구조상 더 적합하다. 브랜치 열 추가로 어떤 도구가 어느 단계에서 쓰이는지 한눈에 파악 가능해졌다.

---

### 🟢 v3에서 새로 추가된 항목

---

**① Airflow DAG 설계 (6장 신규)**

v3에는 DAG-1(외부 데이터 수집)과 DAG-2(마트 ETL)의 트리거 방식, 태스크 구성, 순서 의존성이 상세히 추가됐다.

**추가 이유:** v2의 파이프라인은 단순 단계 나열이었고, 외부 데이터 수집과 마트 ETL의 자동화·스케줄링 구조가 없었다. 실무에서 데이터 파이프라인은 오케스트레이션 없이는 재현성과 순서 보장이 어렵다. Airflow DAG 설계를 계획서에 포함함으로써 기술 깊이가 높아졌다.

---

**② Git 브랜치 전략 독립 섹션 (7장 신규)**

v3에는 브랜치 트리 구조, 각 브랜치별 핵심 산출물 표(docker-compose.yml, ddl.sql, simulate_data.py 등)가 독립 섹션으로 추가됐다.

**추가 이유:** v2에서 브랜치는 파이프라인 표의 한 열로만 존재했다. 독립 섹션으로 분리하면서 "브랜치별 무엇을 만드는가"를 산출물 단위로 명시했다. 깃 저장소 구조를 사전에 파악할 수 있어 가독성이 높아졌다.

---

**③ DB/파이프라인 설계 원칙 명시 (6장 신규)**

> "DB/파이프라인(그릇)이 먼저 구축되어야 한다. 외부 리서치 데이터는 시뮬레이션 생성 이후 파라미터 검증·보정 용도로 활용한다."

**추가 이유:** v2에서는 외부 데이터 수집과 시뮬레이션 생성의 선후 관계가 명확하지 않았다. 이 원칙을 명시함으로써 "외부 데이터로 시뮬레이션 결과를 검증하는 것이 아니라, 파라미터 설정 근거를 보정하는 것"이라는 분석 철학이 문서에 확립됐다.

---

**④ 논리 검토 결함 #2 해결 방향에 "(v3.0 반영 완료)" 명시**

v2의 논리 검토에서 "검증 순서 역전" 결함의 해결 방향은 "파이프라인 4단계에 ETA 분해 선행 배치"로만 적혀 있었다. v3에서는 "(v3.0 반영 완료)"가 추가됐다.

**추가 이유:** 단순히 계획으로 남겨두지 않고 실제 파이프라인(Step 5)에 반영됐음을 명시함으로써, 논리 검토 섹션이 "발견된 문제"에서 "해결된 문제"로 전환됐다는 것을 확인할 수 있다. Living Document의 버전 관리 취지에도 부합한다.

---

### 🟡 종합 평가

v2 → v3는 단순 내용 추가가 아니라 **문서 구조 자체를 재설계**한 버전업이다. 브랜치 전략 개편, Airflow 도입, 섹션 분리는 모두 구조를 개선하는 방향이다. v2에서 삭제됐던 프로모션 수락률 예측 모델 피처 설명은 가설 C의 측정 방법을 구체화하는 내용이므로 4-3절에 복원했다.
