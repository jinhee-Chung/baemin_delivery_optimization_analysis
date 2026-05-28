# 2단계: 데이터 수집 및 적재 설계

> Branch: `feature/02-data-collection`

---

## 목표

배달의 민족 도메인 분석을 기반으로 원천 시스템에서 DW로의 데이터 수집 및 적재 파이프라인을 설계합니다.

---

## 작업 항목

- [ ] ETL 파이프라인 설계
  - 원장성 테이블 적재 전략 (시계열 구축)
  - 거래성 테이블 적재 전략 (1:1 적재, 파티셔닝)
  - 양면성 테이블 적재 전략 (점이력 구축)
- [ ] 데이터 소스 정의
  - `user_master`, `restaurant_master`, `rider_master`, `menu_master`
  - `order_transaction`, `payment_transaction`, `delivery_log`
- [ ] 적재 스케줄 설계 (배치 주기: 일/시간)
- [ ] 데이터 품질 검증 룰 정의

---

## 산출물

- `pipeline_design.md` - ETL 파이프라인 설계서
- `schema_definition.sql` - DW 테이블 스키마 정의
- `data_quality_rules.md` - 데이터 품질 검증 기준
