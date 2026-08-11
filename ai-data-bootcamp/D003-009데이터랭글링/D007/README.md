# 모두마켓 월간 로그 — 도구 선택 보고서

## 1. 데이터 개요
- 파일: web_logs.csv
- 크기: 32.8 MB, 500000 행
- 주요 컬럼: ts / user_id / page / device / status_code / response_ms / bytes_sent

## 2. 측정 결과
| 방식 | 소요 시간 | 메모리 피크 |
|---|---|---|
| pandas + dtype | 2.62초 | 16MB |
| pandas chunked | 4.41초 | 2MB |
| Polars lazy    | 0.73초 | 63MB |

## 3. 분석 결과 요약
- 페이지별 평균 응답 시간 최고: cart (160.27ms)
- 디바이스 점유율: mobile 25.1%, desktop 69.9%, tablet 4.9%
- 에러 1위 페이지: home (42898건)

## 4. 도구 선택 정당화 (한 단락)
이 분석에는 **pandas + dtype**을 선택했습니다. 이유는
(1) 측정 결과 속도와 메모리 사용량 사이에서 가장 균형 잡힌 지점에 있고, (2) 팀 환경상 동료들이 pandas에 익숙해 협업·유지보수 비용이 낮으며, (3) 현재 데이터 규모(수십 MB급)에서는 chunked의 메모리 절약이나 polars의 속도 이점이 실제 체감 효과로 이어지기엔 크지 않기 때문입니다.

## 5. 다음 단계 제안
- 데이터 크기가 **수백 MB~1GB 이상**까지 늘어나면, 메모리 제약이 심한 환경에서는 **chunked 처리**, 처리 속도가 SLA에 민감한 환경에서는 **Polars lazy**로 전환을 고려.
- 시각화·검증은 D+008에서 **Plotly 인터랙티브 시각화(scatter/bar/box 등 + subplot 대시보드)**를 활용해 진행.