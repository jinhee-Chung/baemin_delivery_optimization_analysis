# 배달의 민족 배달 최적화 분석 프로젝트

> 배달의 민족(Baemin) 도메인 기반 데이터 분석 파이프라인 설계 프로젝트

---

## 프로젝트 개요

배달의 민족의 고객(User), 가맹점(Restaurant), 라이더(Rider) 3자 O2O 구조를 기반으로
DW(Data Warehouse) 설계부터 분석 모델링, 리포트까지 전체 데이터 파이프라인을 단계별로 구축합니다.

---

## 분석 파이프라인 구조

| 단계 | 브랜치 | 설명 |
|------|--------|------|
| 1단계 | `feature/01-domain-analysis` | 비즈니스 도메인 분석 및 테이블 설계 이해 |
| 2단계 | `feature/02-data-collection` | 데이터 수집 및 적재 설계 |
| 3단계 | `feature/03-eda` | 탐색적 데이터 분석 (EDA) |
| 4단계 | `feature/04-modeling` | 분석 모델링 (주문 예측, 이탈 분석 등) |
| 5단계 | `feature/05-reporting` | 결과 시각화 및 리포트 작성 |

---

## 브랜치 전략

각 분석 단계는 독립적인 `feature` 브랜치로 관리됩니다.
단계별 작업이 완료되면 `master`로 병합합니다.

```
master
├── feature/01-domain-analysis
├── feature/02-data-collection
├── feature/03-eda
├── feature/04-modeling
└── feature/05-reporting
```
