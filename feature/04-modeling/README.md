# 4단계: 분석 모델링

> Branch: `feature/04-modeling`

---

## 목표

EDA 결과를 기반으로 주문 예측 및 고객 이탈 분석 모델을 개발합니다.

---

## 작업 항목

- [ ] 주문 예측 모델 (Order Demand Forecasting)
  - 시간대별 주문량 예측
  - 지역별 수요 예측
  - 특수일(비, 명절 등) 보정
- [ ] 고객 이탈 분석 (Churn Analysis)
  - 이탈 정의 기준 수립 (최근 N일 미주문)
  - 이탈 예측 피처 엔지니어링
  - 이탈 위험 고객 세그먼트 분류
- [ ] 배달 시간 예측 모델 (Delivery Time Estimation)
  - 거리, 라이더 위치, 시간대 기반 예측
- [ ] 모델 성능 평가
  - RMSE / MAE (회귀)
  - AUC-ROC / Precision-Recall (분류)

---

## 산출물

- `order_forecast_model.ipynb` - 주문 예측 모델
- `churn_analysis_model.ipynb` - 고객 이탈 분석 모델
- `model_evaluation.md` - 모델 성능 평가 결과
