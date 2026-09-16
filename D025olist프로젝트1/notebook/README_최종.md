# Olist 배송 지연 → 리뷰 경험 분석 — README

## 1. 프로젝트 개요

- **핵심 질문**: 배송 지연과 리뷰 점수 간에 상관관계가 있는가? (지연될수록 리뷰 점수가 낮아지는가?)
- **보조 질문 1**: 어느 배송 단계(승인·발송준비·운송)에서 지연이 주로 발생하는가?
- **보조 질문 2**: 상품 카테고리별로 지연–리뷰 상관관계의 강도가 다른가?
- **외부 데이터 보강**: 성수기 캘린더(블랙프라이데이 등)가 배송 지연을 예측하는가? (Olist 데이터에
  없는 브라질 공휴일·상업 기념일 날짜를 결합 — §5 스크래핑 코드 참고)
- **후속 분석**: 이름 붙은 성수기 이벤트가 아닌데도, 2018년 2~3월에 또 다른 지연 스파이크가
  있었는가? 있었다면 원인은 무엇인가?
- 분석 노트북: `olist_delivery_review_analysis.ipynb`

## 2. 실행 환경

이 노트북은 아래 버전에서 개발·검증(Restart & Run All)되었습니다. 정확히 같은 버전이 아니어도
대부분 문제없이 동작하지만, 버전 차이로 결과가 달라질 경우 아래를 우선 확인하세요.

| 패키지 | 버전 |
| --- | --- |
| Python | 3.11.15 |
| pandas | 3.0.2 |
| numpy | 2.4.4 |
| scipy | 1.17.1 |
| statsmodels | 0.14.6 |
| scikit-learn | 1.8.0 |
| matplotlib | 3.10.9 |

설치 예시:

```bash
pip install pandas numpy scipy statsmodels scikit-learn matplotlib jupyter nbconvert
```

한글 차트 라벨을 정상적으로 보려면 시스템에 한글 폰트(예: Noto Sans CJK KR, 나눔고딕)가 설치되어
있어야 합니다. 폰트가 없어도 통계 분석 자체는 정상 동작하며 차트의 한글 라벨만 깨져 보입니다.

## 3. 데이터 다운로드 방법

1. Kaggle에서 **Brazilian E-Commerce Public Dataset by Olist**를 다운로드합니다.
   https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce
2. 압축을 풀어 아래 9개 CSV 파일을 노트북이 있는 폴더의 **한 단계 위(부모 폴더)**에
   `Olist_dataset/` 폴더를 만들어 그 안에 둡니다.

```
프로젝트 루트/
├── Olist_dataset/
│   ├── olist_customers_dataset.csv
│   ├── olist_geolocation_dataset.csv
│   ├── olist_order_items_dataset.csv
│   ├── olist_order_payments_dataset.csv
│   ├── olist_order_reviews_dataset.csv
│   ├── olist_orders_dataset.csv
│   ├── olist_products_dataset.csv
│   ├── olist_sellers_dataset.csv
│   └── product_category_name_translation.csv
└── notebook/
    ├── olist_delivery_review_analysis.ipynb
    └── README.md
```

파일명은 Kaggle 원본 그대로 사용합니다. 노트북 코드는 `DATA_DIR = "../Olist_dataset"` 상대경로로
파일을 읽으므로, 위 폴더 구조(노트북의 부모 폴더에 `Olist_dataset/`)를 그대로 맞춰야 합니다.
다른 경로를 쓰고 싶다면 노트북 맨 위 "환경 설정" 셀의 `DATA_DIR` 값만 바꾸면 됩니다.

> **주의 — 상대경로는 절대경로로 바꾸지 마세요.** `../Olist_dataset`는 각자의 컴퓨터에서 폴더
> 구조만 맞으면 누구에게나 동일하게 동작하는 방식입니다. 반대로 절대경로(예:
> `/Users/본인이름/...`)로 바꾸면 **작성자의 컴퓨터에서만 동작하고, 다른 사람이 받으면 반드시
> 오류가 납니다.** 경로 문제가 있다면 절대경로로 바꾸지 말고, 아래 4장의 "실행 위치" 안내를
> 확인하세요.

## 4. 노트북 실행 순서

1. 위 데이터 다운로드를 완료합니다.
2. Jupyter를 실행하고 `olist_delivery_review_analysis.ipynb`를 엽니다.

   **실행 위치(working directory) 확인 — 데이터로드 오류의 가장 흔한 원인입니다.**
   `DATA_DIR = "../Olist_dataset"`는 노트북이 실행되는 현재 작업 디렉토리를 기준으로 한 단계
   위 폴더를 찾습니다. 다음을 확인하세요.
   - **Jupyter Notebook/Lab (브라우저)**: 터미널에서 `jupyter notebook` 또는 `jupyter lab`을
     **이 노트북이 있는 폴더 안에서** 실행하세요 (예: `cd notebook폴더 && jupyter lab`). 다른
     폴더에서 실행한 뒤 파일 탐색기로 이동해 열면 작업 디렉토리가 달라져 경로가 깨집니다.
   - **VSCode**: 노트북이 있는 폴더(또는 그 상위 폴더 전체)를 "Open Folder"로 열고, 우측 상단
     커널 선택 옆 작업 디렉토리가 노트북 파일 위치와 같은지 확인하세요. VSCode는 보통 노트북
     파일의 폴더를 작업 디렉토리로 자동 설정하지만, 다른 워크스페이스가 열려 있으면 달라질 수
     있습니다.
   - 데이터로드 셀(0장)에서 `FileNotFoundError`가 나면 십중팔구 이 문제입니다. 절대경로로
     바꾸지 말고, 실행 위치를 맞추거나 노트북 상단 `DATA_DIR`을 본인 환경에 맞는 경로로
     일시적으로 수정해서 확인하세요.
3. **Kernel → Restart & Run All**로 처음부터 끝까지 순서대로 실행합니다. (섹션을 건너뛰거나
   순서를 바꾸면 이후 셀에서 오류가 날 수 있습니다 — 뒤 섹션이 앞 섹션에서 만든 변수를 사용합니다.)
4. 노트북 구성 순서:
   0. 데이터 로드
   1–2. 전처리 판단 (리뷰 1:N 처리, 주문 필터링)
   3. 핵심 지표 정의 (지연여부/지연일수/구간별 소요일수)
   4. 대표 카테고리·주문금액 산정
   4-1. EDA — 리뷰 점수 분포 확인 (천장효과 확인, 비모수 방법 선택 근거)
   5. 핵심질문 분석 (Spearman, Mann–Whitney U, Cliff's delta)
   6. 교란변수 통제 회귀 (OLS, 순서형 로지스틱)
   7-0. 전처리 판단 — 구간별 소요일수의 음수(데이터 오류) 처리
   7. 보조질문 1 — 배송 단계 분해
   8. 보조질문 2 — 카테고리별 차이 + 상호작용 검정
   9. 외부 데이터 보강 — 블랙프라이데이 등 성수기 캘린더가 배송 지연을 예측하는가
   9-1. 외부 데이터 보강 체크리스트 (발표자료 부록용)
   9-2. 후속 분석 — 2018년 2~3월, 이름 없는 지연 스파이크의 원인 소거
   10. Findings → Insights → Recommendations 요약
   11. 한계 및 후속 분석 제안

## 5. 스크래핑 코드

**웹 스크래핑은 수행하지 않았습니다.** 배송·리뷰·상품 등 핵심 분석은 Olist 공개 데이터셋 9개
테이블만 사용합니다.

다만 §9(외부 데이터 보강) 성수기 캘린더 분석에서는 Olist 데이터에 없는 **브라질 공휴일·상업
기념일 날짜**(블랙프라이데이·어머니날·연인의날·어린이날·크리스마스)를 결합했습니다. 이 날짜는
스크래핑이 아니라 **날짜 계산**(블랙프라이데이 = 미국 추수감사절 다음 날)과 **공개적으로 알려진
브라질 기념일 날짜**를 코드에 직접 입력하는 방식으로 확보했습니다. 수집 방식과 출처는 노트북
§9-1 "외부 데이터 보강 체크리스트"에 표로 정리되어 있습니다.

## 6. 재현성 확인

이 노트북은 제출 전 `jupyter nbconvert --to notebook --execute --inplace`로 **Restart & Run All**과
동일한 조건에서 전체 실행하여 오류가 없음을 확인했습니다.
