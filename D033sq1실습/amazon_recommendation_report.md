# Amazon 데이터셋 기반 추천 시스템 설계

Amazon Sales Dataset(1,465행)을 DuckDB SQL로 정제·분석하여 서로 다른 5가지 관점의 추천 시스템을 설계했다.

## 0. 데이터 전처리

원본 데이터를 그대로 쓸 수 없는 문제가 세 가지 있었다.

- `discounted_price` / `actual_price`에 ₹ 기호와 천 단위 콤마가 섞여 있어 숫자로 못 씀 → `REPLACE`로 제거 후 캐스팅
- `discount_percentage`에 % 기호가 붙어 있음 → 제거 후 캐스팅
- `rating` 컬럼에 "|"라는 깨진 값이 1건(`B08L12N5H1`) 있어서 일반 `CAST`를 쓰면 쿼리 전체가 에러로 멈춤 → `TRY_CAST`로 해당 값만 NULL 처리하고 나머지 데이터는 살림
- **product_id 중복 114건** 발견: 같은 상품이 리뷰 스냅샷만 미세하게(rating_count ±1 등) 다르게 중복 등록되어 있었음. 그대로 두면 인기도·유사도 계산이 왜곡되므로 `ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY rating_count DESC)`로 대표 행 하나만 남겨 1,465행 → **1,351개 고유 상품**으로 정리(`products_clean`)

```sql
CREATE OR REPLACE TABLE products_clean AS
WITH numbered AS (
    SELECT
        product_id,
        product_name,
        category,
        TRY_CAST(REPLACE(REPLACE(discounted_price, '₹',''), ',','') AS DOUBLE) AS discounted_price,
        TRY_CAST(REPLACE(REPLACE(actual_price, '₹',''), ',','') AS DOUBLE) AS actual_price,
        TRY_CAST(REPLACE(discount_percentage, '%','') AS DOUBLE) AS discount_percentage,
        TRY_CAST(rating AS DOUBLE) AS rating,
        TRY_CAST(REPLACE(rating_count, ',','') AS BIGINT) AS rating_count,
        about_product, user_id, user_name, review_id, review_title, review_content,
        ROW_NUMBER() OVER (
            PARTITION BY product_id
            ORDER BY TRY_CAST(REPLACE(rating_count, ',','') AS BIGINT) DESC NULLS LAST
        ) AS rn
    FROM amazon
)
SELECT * EXCLUDE (rn) FROM numbered WHERE rn = 1;
```

![전처리 결과](images/00_preprocessing.png)

이후 5개 추천 시스템은 전부 이 `products_clean` 테이블을 기준으로 한다.

---

## 추천 시스템 1

**1. 추천 시스템 이름**
➜ "이 리뷰, 진짜로 추천한다고 했어요"

**2. 추천 시스템의 테마**
➜ 별점만으로는 "진짜 만족도"를 다 담지 못한다. 리뷰 본문에 "recommend"라는 단어가 실제로 등장하는 상품을 추려내면, 숫자 평점 뒤에 숨은 실사용자의 생생한 긍정 언급을 근거로 한 추천을 만들 수 있다. 다만 "I would not recommend this"처럼 부정 표현 안에 들어간 경우까지 잡으면 오히려 역효과이므로, 부정 표현이 포함된 경우는 제외했다.

**3. 구현 로직**
➜ `user_id`/`review_id`는 콤마로 쪼개도 안전하지만, `review_content`는 리뷰 문장 자체에 콤마가 많이 섞여 있어(1,465행 중 89%에서 분리 시 배열 길이가 안 맞음) 개별 리뷰 단위로 쪼개는 대신 **상품 단위 텍스트 전체**에서 키워드를 검색하는 방식을 택했다. `rating >= 4.0` 조건을 더해 "recommend 언급 + 실제 평점"이라는 두 신호를 함께 요구했다.

```sql
SELECT product_id, product_name, category, rating, rating_count, discount_percentage
FROM products_clean
WHERE REGEXP_MATCHES(LOWER(review_content), 'recommend')
  AND NOT REGEXP_MATCHES(LOWER(review_content), '(not|don''t|dont|wouldn''t|never)\s+recommend')
  AND rating >= 4.0
ORDER BY rating_count DESC;
```

**4. 결과**
➜ 227개 상품이 매칭되었다. 1위는 리뷰 27만 건의 Pigeon 야채 다지기, 그 뒤로 JBL 이어폰, MI 파워뱅크 등이 이어진다. 부정 표현("don't recommend" 등)을 걸러낸 덕분에, 결과에 뜨는 상품들은 실제로 리뷰에서 긍정적으로 "추천한다"고 언급된 상품들이다.

![시스템 1 결과](images/01_recommend_keyword.png)

---

## 추천 시스템 2

**1. 추천 시스템 이름**
➜ "숫자 몇 개보다 확실한 인기 랭킹"

**2. 추천 시스템의 테마**
➜ 단순히 "평점 4.5 이상 중 리뷰 수 DESC"로 정렬하면, 리뷰가 적은데 우연히 평점이 높은 상품과 리뷰가 압도적으로 많은 상품을 같은 기준으로 비교하기 어렵다. IMDB의 가중 평점 공식처럼 리뷰 수(신뢰도)와 평점(만족도)을 하나의 점수로 합성한 **베이지안 가중 인기 점수**를 설계했다. 리뷰가 적은 상품은 전체 평균 쪽으로 점수가 당겨지고, 리뷰가 많을수록 그 상품 고유의 평점이 더 크게 반영된다.

**3. 구현 로직**
➜ 전체 평균 평점(`c_mean`)과 전체 평균 리뷰 수(`m_thresh`)를 구한 뒤, 각 상품의 `weighted_score = (v/(v+m))·R + (m/(v+m))·C` 공식으로 계산한다(v=해당 상품 리뷰 수, R=해당 상품 평점, m=전체 평균 리뷰 수, C=전체 평균 평점).

```sql
WITH stats AS (
    SELECT AVG(rating) AS c_mean, AVG(rating_count) AS m_thresh
    FROM products_clean
    WHERE rating IS NOT NULL AND rating_count IS NOT NULL
)
SELECT
    p.product_id, p.product_name, p.category, p.rating, p.rating_count,
    ROUND(
        (p.rating_count / (p.rating_count + s.m_thresh)) * p.rating
        + (s.m_thresh / (p.rating_count + s.m_thresh)) * s.c_mean
    , 3) AS weighted_score
FROM products_clean p, stats s
WHERE p.rating IS NOT NULL AND p.rating_count IS NOT NULL
ORDER BY weighted_score DESC;
```

**4. 결과**
➜ 1,348개 상품 중 1위는 평점 4.8·리뷰 53,803건의 Swiffer 전기온수기(weighted_score 4.625)였다. 리뷰가 20만 건 넘는 SanDisk 메모리카드보다 오히려 리뷰 수는 적지만 평점이 더 높은 상품이 상위에 오른 것을 보면, 단순 리뷰 수 정렬과는 다른 결과가 나온다는 걸 확인할 수 있다.

![시스템 2 결과](images/02_weighted_popularity.png)

---

## 추천 시스템 3

**1. 추천 시스템 이름**
➜ "이 상품, 나만 리뷰한 게 아니었어요"

**2. 추천 시스템의 테마**
➜ 협업 필터링의 핵심 아이디어인 "함께 소비된 상품"을 리뷰 데이터로 근사한다. 같은 사용자가 서로 다른 두 상품에 리뷰를 남겼다면, 그 두 상품은 어떤 형태로든 연관이 있다고 볼 수 있다. 다만 분석 결과, 리뷰어가 완전히 겹치는 상품쌍은 대부분 **같은 상품의 색상·용량 옵션**(같은 브랜드 케이블의 다른 색상 등)이었고, 진짜 서로 다른 카테고리 상품 간의 겹침은 리뷰어 1명 수준으로 약했다. 이 두 신호를 구분해서 라벨링했다.

**3. 구현 로직**
➜ `user_id`는 콤마로 쪼개도 안전하므로(리뷰당 리뷰어 8명이 오차 없이 매칭됨) 이것만 UNNEST해서 (상품, 리뷰어) 쌍을 만들고, 같은 리뷰어가 겹치는 상품쌍을 self join으로 찾는다. `category`가 같으면 "옵션 추천", 다르면 "교차 카테고리 발견형 추천"으로 라벨을 붙였다.

```sql
WITH product_user AS (
    SELECT product_id, UNNEST(STR_SPLIT(user_id, ',')) AS reviewer_id
    FROM products_clean
),
pair_counts AS (
    SELECT
        a.product_id AS base_product_id,
        b.product_id AS related_product_id,
        COUNT(DISTINCT a.reviewer_id) AS shared_reviewers
    FROM product_user a
    JOIN product_user b
      ON a.reviewer_id = b.reviewer_id
     AND a.product_id <> b.product_id
    GROUP BY a.product_id, b.product_id
)
SELECT
    p1.product_name AS base_product,
    p2.product_name AS related_product,
    pc.shared_reviewers,
    CASE WHEN p1.category = p2.category THEN '동일 카테고리(옵션 추천)' ELSE '교차 카테고리(발견형 추천)' END AS signal_type
FROM pair_counts pc
JOIN products_clean p1 ON p1.product_id = pc.base_product_id
JOIN products_clean p2 ON p2.product_id = pc.related_product_id
QUALIFY ROW_NUMBER() OVER (PARTITION BY pc.base_product_id ORDER BY pc.shared_reviewers DESC) <= 3
ORDER BY pc.shared_reviewers DESC;
```

**4. 결과**
➜ 1,045개 행이 매칭됐다. 상위권은 iQOO Z6 44W의 색상별 옵션들처럼 리뷰어 8명이 그대로 겹치는 "동일 카테고리(옵션 추천)" 케이스가 대부분이었는데, 이는 이 데이터셋에서 동일 상품 라인이 리뷰어 풀을 그대로 재사용하는 특성 때문으로 분석된다. 실제 서비스라면 "이 상품의 다른 옵션" 위젯으로, 그리고 겹침이 1명인 교차 카테고리 케이스는 약하지만 "이런 것도 관심 있을 수 있어요" 발견형 추천으로 구분해 활용할 수 있다.

![시스템 3 결과](images/03_co_review.png)

---

## 추천 시스템 4

**1. 추천 시스템 이름**
➜ "적당히 할인된 게 오히려 믿을 만해요"

**2. 추천 시스템의 테마**
➜ 할인율이 지나치게 높으면(70% 이상) 오히려 재고떨이나 낚시성 표시가격일 가능성을 의심하게 된다. 실제로 할인 구간별 평균 평점을 확인해보니 10~40%(4.13) > 40~70%(4.08) > 70%+(4.02) 순으로 낮아지는 경향이 나타나, "과도한 할인=의심"이라는 가설이 데이터로 뒷받침되었다. 이에 따라 메리트도 있고 신뢰도도 높은 10~40% 구간을 "안전존"으로 정의해 추천한다.

**3. 구현 로직**
➜ 단순 정렬이 아니라 `discount_percentage`를 구간으로 나눠 적정 구간만 필터링한 뒤, 그 안에서 평점과 리뷰 수로 다시 랭킹한다.

```sql
SELECT product_id, product_name, category, discount_percentage, rating, rating_count
FROM products_clean
WHERE discount_percentage BETWEEN 10 AND 40
  AND rating IS NOT NULL
ORDER BY rating DESC, rating_count DESC;
```

**4. 결과**
➜ 432개 상품이 안전존에 해당했다. 1위는 평점 4.8의 Swiffer 온수기(할인율 28%), 이어서 로지텍 마우스류가 다수 포진했다. 할인율 70% 이상 구간(211개 상품, 평균 평점 4.02)과 비교하면 이 구간(평균 평점 4.13)이 확실히 더 신뢰할 만하다는 걸 알 수 있다.

![시스템 4 결과](images/04_discount_safezone.png)

---

## 추천 시스템 5

**1. 추천 시스템 이름**
➜ "이 정도 품질이면, 이게 더 나아요" (동급 저가 대체재)

**2. 추천 시스템의 테마**
➜ 지금 보고 있는 상품과 평점이 비슷한데 더 저렴한 상품이 있다면, 굳이 비싼 걸 살 필요가 없다. 같은 품질 수준에서 더 합리적인 소비를 돕는 대체재 추천이다.

**3. 구현 로직**
➜ 처음에는 category를 대분류 3단계로 묶어서 매칭했더니 로봇청소기와 티스푼 받침대가 "대체재"로 잡히는 등 말이 안 되는 결과가 나왔다. category의 마지막 depth 하나만 잘라낸(`category_trim1`) 세부 카테고리로 좁히고, 평점 차이 0.2 이내, 그리고 가격은 원가의 50~90% 사이(너무 극단적인 절감은 오히려 다른 등급의 상품일 가능성이 높아 배제)로 조건을 걸었다.

```sql
WITH base AS (
    SELECT *,
        ARRAY_TO_STRING(LIST_SLICE(STR_SPLIT(category, '|'), 1, GREATEST(LEN(STR_SPLIT(category,'|'))-1,1)), '|') AS category_trim1
    FROM products_clean
    WHERE rating IS NOT NULL AND discounted_price IS NOT NULL
)
SELECT
    a.product_id AS base_id, a.product_name AS base_name, a.discounted_price AS base_price, a.rating AS base_rating,
    b.product_id AS alt_id, b.product_name AS alt_name, b.discounted_price AS alt_price, b.rating AS alt_rating,
    ROUND((a.discounted_price - b.discounted_price) / a.discounted_price * 100, 1) AS save_pct
FROM base a
JOIN base b
  ON a.category_trim1 = b.category_trim1
 AND a.product_id <> b.product_id
 AND ABS(a.rating - b.rating) <= 0.2
 AND b.discounted_price BETWEEN a.discounted_price * 0.5 AND a.discounted_price * 0.9
QUALIFY ROW_NUMBER() OVER (PARTITION BY a.product_id ORDER BY save_pct DESC) = 1
ORDER BY save_pct DESC;
```

**4. 결과**
➜ 966개 상품에 대해 대체재를 찾을 수 있었다. 예를 들어 Ambrane 20000mAh 파워뱅크(₹1,799, 평점 4.1) 대신 URBN 10000mAh 파워뱅크(₹900, 평점 4.0)가 제안되고, 47,990원짜리 삼성/LG 55인치 TV 대신 23,999원의 Acer 43인치 TV가 제안되는 등, 실제로 말이 되는 대체 소비 제안이 만들어졌다.

![시스템 5 결과](images/05_cheaper_substitute.png)
