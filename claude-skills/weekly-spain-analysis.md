# 스페인 주간 지표 분석

> 스페인 5개 도시(바르셀로나, 마드리드, 세비야, 그라나다, 마요르카)의 주간 지표를 분석하여, 이번 주 액션 아이템을 도출하고 Confluence 위키에 발행한다.

## 사용법

```
/weekly-spain-analysis
/weekly-spain-analysis W17
```

## 핵심 원칙

1. **FP&A 시트를 선행 참조한다.** GMV, CGMV, 확정률, 공헌이익, 유입, 전환 등 수치는 FP&A 시트 기준이 1차 소스다.
2. **상품 단위 분석은 FUNT 대시보드 카테고리 기준**으로 한다.
3. **도시별로 독립 정리**한다. 국가 합산은 참고용.
4. **증가 카테고리와 감소 카테고리 모두, 빠짐없이 전체** 기재한다. CM 변동이 소규모(1만 미만)여도 생략하지 않는다. FUNT 대시보드 카테고리 기준으로 모든 카테고리를 나열할 것.
5. **위키 양식에 맞춰** Confluence에 자동 발행한다.

---

## 데이터 소스

### 1차 소스: FP&A 구글시트

**시트 URL**: `https://docs.google.com/spreadsheets/d/1ael4JHUI1LjwxxxO-4-f30JTRO1SDkzjfpxi88eh-cQ/edit?gid=0#gid=0`

| 탭 | 용도 | 주요 지표 |
|----|------|----------|
| `R_손익_WoW` | 도시별 WoW 손익 | GMV, CGMV, 확정률, **공헌이익(CM)** |
| `R_카테고리_WoW` | 카테고리별 WoW | 카테고리별 GMV, CGMV, UV, CVR, CM |

> **CM(공헌이익)은 반드시 FP&A 시트 `R_손익_WoW` 기준**으로 사용한다. BQ의 CM 컬럼(`CON_MARGIN`)이 아님.
> FP&A 시트의 CM은 채널수수료, 쿠폰(CPD), 마케팅파트너비 등을 별도 수식으로 추가 차감하기 때문에 BQ `CON_MARGIN`보다 낮다.

**BQ와 FP&A 시트 값 일치 여부:**

| 지표 | BQ = FP&A? | 비고 |
|------|:---:|------|
| GMV | △ | 근사치이나 소폭 차이 있음 (타이밍/필터 차이) |
| CGMV | **X** | 시트가 기준. BQ와 상당 차이 (세비야 등 최대 6% 차이) |
| 확정률 | **X** | CGMV 차이로 인해 BQ와 큰 차이 (마요르카 65.7% vs BQ 78.7%) |
| UV/CVR | **X** | 시트 UV는 BQ BIZ_LOG PID UV와 정의 다름. 시트 기준 사용 |
| **CM(공헌이익)** | **X** | FP&A 시트가 기준. BQ `CON_MARGIN` 사용 금지 |
| 예약 수 | **X** | 시트와 BQ `ORDER_ID` 카운트 정의 다름. 시트 기준 사용 |

> **결론: 모든 지표를 FP&A 시트에서 선행 조회한다.** BQ는 상품 단위 분석(FUNT 카테고리별 GMV WoW, 상품별 CFR 등)에만 보조적으로 사용한다.

**시트 접근 방법:**
- 서비스 계정: `claude-sheets@hellofirebase-e3a6a.iam.gserviceaccount.com`
- 접근 불가 시: 유저에게 CSV 다운로드 또는 값 공유 요청
- Google Drive API로 시트 읽기 시, 계층 구조(병합 셀) 때문에 도시별 행 식별이 어려움 → 유저에게 도시별 CM 값 직접 확인 요청

### 2차 소스: BigQuery (상품 단위 분석)

**BQ wrapper**: `./.claude/hooks/run-bq-readonly.sh bq query --use_legacy_sql=false --location=asia-northeast3 --format=prettyjson --max_rows=200 "쿼리"`

#### 핵심 테이블

| 테이블 | 용도 |
|--------|------|
| `edw_fpna.MART_FPNA_NONAIR_PROFIT_D` | 주문/GMV/CGMV/CFR/마진 (FP&A 원천) |
| `edw_mart.MART_PRODUCT_AGG_D` | 상품별 UV/체크아웃/결제 퍼널 |
| `edw_mart.MART_SALE_D` | 예약/주문 상세 |
| `edw_mart.MART_PRODUCT_D` | 상품 마스터 (도시, 카테고리, 파트너) |
| `business.TNA_FUN_T_category_WoW` | FUNT 카테고리별 주간 집계 |
| `business.TNA_FUN_T_category_DoD` | FUNT 카테고리별 일간 집계 |
| `business.TNA_dashboard_category_product_list_v1` | 상품→FUNT 카테고리 매핑 |
| `business.tna_calendar_open_rate_history` | 캘린더 오픈율 이력 |
| `edw_mart.MART_BIZ_LOG_PID_CONVERSION_D` | 채널별 유입/전환 (UTM) |
| `business.mkt_dashboard_raw_data` | 광고비/ROAS |

#### TNA 필터 조건

- `FPNA_DOMAIN_NM = 'TNA'`
- 숙박/항공/열차 카테고리 제외
- '민박', '현지민박', 'B2C 비노출 숙소' 제외
- 확정 건: `RECENT_STATUS IN ('confirm', 'finish')` — finish 반드시 포함
- 마요르카 도시: `CITY_NM IN ('Palma de Mallorca', 'Majorca')` 합산

---

## FUNT 대시보드 카테고리

카테고리 분석 시 FPNA `CATEGORY_NM`이 아닌, **FUNT 대시보드 카테고리**를 사용한다.

| 바르셀로나 | 마드리드 | 세비야 | 그라나다 | 마요르카 |
|-----------|---------|--------|---------|---------|
| 가우디 투어 | 세고비아&톨레도 | 론다 별투어 | 알함브라입장권 | 근교투어 |
| 몬세라트 | 프라도미술관 | 그라나다 이동투어 | 알함브라투어 | 시내투어 |
| 바르셀로나 티켓 | 마드리드 왕궁 | 시내투어&대성당 | 이동투어 | 액티비티 |
| 바르셀로나 시내/야경/박물관 투어 | 축구 | 플라멩고 티켓 | 시내/야경투어 | 스냅 |
| 축구 | 입장권 | 세비야 스냅 | 플라멩고 | 티켓/입장권 |
| 바르셀로나 스냅 | 플라멩고 티켓 | 입장권 | etc | etc |
| 근교투어(몬세라트 제외) | 시내투어 | 1박 이상 | | |
| 클래스 | 1박 이상 | etc | | |
| 교통 | etc | | | |
| 액티비티 | | | | |
| 유심/이심 | | | | |
| etc | | | | |

---

## FUNT 상품 단위 분석 쿼리 (참고 쿼리 #19787)

상품 단위로 유입/전환/캘린더/예약을 조회할 때 아래 쿼리 구조를 참고한다.

### 쿼리 파라미터

| 파라미터 | 설명 | 예시 |
|---------|------|------|
| `{{region/country/city}}` | 도시/국가 필터 | `'Barcelona'`, `'Madrid'` 등 |
| `{{category_nm}}` | FUNT 대시보드 카테고리 필터 | `'ALL'` 또는 특정 카테고리 |
| `{{offer_id}}` | 상품 GID 필터 | `'ALL'` 또는 특정 GID |
| `{{start_date}}` | 분석 시작일 | 비교 대상 주 월요일 |
| `{{end_date}}` | 분석 종료일 | 분석 대상 주 일요일 |
| `{{trunc_by}}` | 집계 단위 | `WEEK(MONDAY)` (주간) 또는 `DAY` (일간) |

### 쿼리 구조

```sql
-- ① 상품 목록 (FUNT 카테고리 매핑 포함)
WITH product AS (
    SELECT DISTINCT mp.* EXCEPT(filter_product_id)
    FROM business.TNA_dashboard_category_product_list_v1 AS mp
    WHERE CASE WHEN 'ALL' IN ({{region/country/city}}) THEN 1=1
            ELSE (country_nm IN ({{region/country/city}}) OR region_nm IN ({{region/country/city}}) OR city_nm IN ({{region/country/city}})) END
        AND CASE WHEN 'ALL' IN ({{category_nm}}) THEN 1=1
            ELSE dashboard_category IN ({{category_nm}}) END
        AND CASE WHEN 'ALL' IN ({{offer_id}}) THEN 1=1
            ELSE mp.filter_product_id IN ({{offer_id}}) END
)

-- ② 2.0→3.0 마이그레이션 매핑
, tna_migration AS (
    SELECT CAST(p.id AS STRING) AS gid_30, CAST(m.offer_id AS STRING) AS id_20
    FROM mrtdata.edw.DW_MRT_EXPERIENCES_PRODUCTS AS p
    LEFT JOIN mrtdata.edw.DW_MRT_EXPERIENCES_PRODUCT_MIGRATION_MAP AS m
        ON p.id = m.product_id AND m.deleted_at IS NULL
    WHERE m.offer_id IS NOT NULL AND p.status IN ('ONSALE')
)

-- ③ 옵션 수량
, option AS (
    SELECT DISTINCT reservation_id, SUM(quantity) quantity
    FROM edw.DW_MRT_ORDERS_OPTION_RESERVATIONS
    WHERE deleted_at IS NULL
    GROUP BY 1
)

-- ④ 리뷰
, review AS (
    SELECT r.product_no, COUNT(DISTINCT r.id) AS cum_review_cnt, AVG(score) AS cum_avg_score
    FROM edw.DW_MRT_REVIEWS_REVIEWS AS r
    LEFT JOIN edw.DW_MRT_REVIEWS_REVIEW_RESERVATION_MAPPINGS AS rm ON r.id = rm.review_id
    WHERE DATE(r.created_at) <= '{{end_date}}'
    GROUP BY 1
)

-- ⑤ 마진 (FPNA 기준)
, margin AS (
    SELECT DISTINCT s.* EXCEPT(PRODUCT_ID),
        m.* EXCEPT(RESVE_ID),
        COALESCE(p.gid_30, s.PRODUCT_ID) AS PRODUCT_ID
    FROM mrtdata.edw_mart.MART_SALE_D AS s
    LEFT JOIN (
        SELECT RESVE_ID, SUM(CON_MARGIN) AS CON_MARGIN
        FROM edw_fpna.MART_FPNA_NONAIR_PROFIT_D
        GROUP BY 1
    ) AS m ON s.RESVE_ID = m.resve_id
    LEFT JOIN tna_migration AS p ON s.PRODUCT_ID = p.id_20
    WHERE s.product_id IN (SELECT PRODUCT_ID FROM product)
)

-- ⑥ 일별/주별 상품별 예약
, sales_data AS (
    SELECT BASIS_DATE, PRODUCT_ID, gmv, confirm_gmv, order_count, resve_count, confirm_resve_cnt,
        quantity, confirm_quantity,
        SAFE_DIVIDE(gmv, order_count) AS ASP,
        avg_user_resve_cnt, avg_resve_prsnl_cnt,
        SAFE_DIVIDE(margin, confirm_gmv) * 100 AS CM
    FROM (
        SELECT DATE_TRUNC(BASIS_DATE, {{trunc_by}}) BASIS_DATE,
            s.PRODUCT_ID,
            SUM(sales_krw_price) AS gmv,
            SUM(CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN sales_krw_price END) AS confirm_gmv,
            COUNT(DISTINCT s.ORDER_ID) AS order_count,
            COUNT(DISTINCT s.RESVE_ID) AS resve_count,
            COUNT(DISTINCT CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN s.RESVE_ID END) AS confirm_resve_cnt,
            COUNT(DISTINCT s.user_id) AS user_cnt,
            COUNT(DISTINCT CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN s.user_id END) AS confirm_user_cnt,
            SUM(quantity) AS quantity,
            SUM(CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN quantity END) AS confirm_quantity,
            SAFE_DIVIDE(COUNT(DISTINCT s.RESVE_ID), COUNT(DISTINCT s.user_id)) AS avg_user_resve_cnt,
            SAFE_DIVIDE(SUM(quantity), COUNT(DISTINCT s.RESVE_ID)) AS avg_resve_prsnl_cnt,
            SUM(CASE WHEN s.RECENT_STATUS IN ('confirm','finish') THEN CON_MARGIN END) AS margin
        FROM margin AS s
        LEFT JOIN mrtdata.edw.DW_MRT_ORDERS_RESERVATIONS AS r ON s.RESVE_ID = r.reservation_no
        LEFT JOIN option AS o ON r.id = o.reservation_id
        WHERE kind = 1
            AND basis_date BETWEEN '{{start_date}}'-7 AND '{{end_date}}'
        GROUP BY 1, 2
    )
    WHERE basis_date BETWEEN '{{start_date}}' AND '{{end_date}}'
    ORDER BY BASIS_DATE DESC, 2
)

-- ⑦ 일별/주별 상품별 유입
, log_data AS (
    SELECT DATE_TRUNC(basis_dt, {{trunc_by}}) basis_dt,
        GID,
        SUM(OFFER_DETAIL_UV) OFFER_DETAIL_UV,
        SUM(CHECKOUT_UV) CHECKOUT_UV,
        SUM(CHECKOUT_COMPLETE_UV) CHECKOUT_COMPLETE_UV,
        SAFE_DIVIDE(SUM(CHECKOUT_COMPLETE_UV), SUM(OFFER_DETAIL_UV)) * 100 AS CVR,
        SAFE_DIVIDE(SUM(CHECKOUT_UV), SUM(OFFER_DETAIL_UV)) * 100 AS DETAIL_TO_CHECKOUT,
        SAFE_DIVIDE(SUM(CHECKOUT_COMPLETE_UV), SUM(CHECKOUT_UV)) * 100 AS CHECKOUT_TO_COMPLETE
    FROM (
        SELECT basis_dt,
            COALESCE(m.gid_30, p.gid) AS gid,
            OFFER_DETAIL_UV, CHECKOUT_UV, CHECKOUT_COMPLETE_UV
        FROM mrtdata.edw_mart.MART_PRODUCT_AGG_D AS p
        LEFT JOIN tna_migration AS m ON p.gid = m.id_20
        WHERE basis_dt BETWEEN '{{start_date}}'-7 AND '{{end_date}}'
            AND GID IN (SELECT PRODUCT_ID FROM product)
    )
    WHERE basis_dt BETWEEN '{{start_date}}' AND '{{end_date}}'
    GROUP BY 1, 2
)

-- ⑧ 전주 대비 증감 (WoW)
, STAT AS (
    SELECT DISTINCT a.*, b.* EXCEPT(BASIS_DATE, PRODUCT_ID), c.* EXCEPT(basis_dt, GID)
    FROM (
        SELECT DISTINCT basis_dt, GID FROM log_data
        UNION ALL
        SELECT DISTINCT BASIS_DATE, PRODUCT_ID FROM sales_data
    ) AS a
    LEFT JOIN sales_data AS b ON a.basis_dt = b.basis_date AND a.gid = b.PRODUCT_ID
    LEFT JOIN log_data AS c ON a.basis_dt = c.basis_dt AND a.gid = c.gid
)

, STAT1 AS (
    SELECT *,
        SAFE_DIVIDE(gmv - last_week_gmv, last_week_gmv) * 100 AS last_week_gmv_diff,
        SAFE_DIVIDE(order_count - last_week_order_count, last_week_order_count) * 100 AS last_week_order_count_diff,
        SAFE_DIVIDE(resve_count - last_week_resve_count, last_week_resve_count) * 100 AS last_week_resve_count_diff,
        SAFE_DIVIDE(ASP - last_week_ASP, last_week_ASP) * 100 AS last_week_ASP_diff,
        SAFE_DIVIDE(OFFER_DETAIL_UV - last_week_OFFER_DETAIL_UV, last_week_OFFER_DETAIL_UV) * 100 AS last_week_OFFER_DETAIL_UV_diff,
        SAFE_DIVIDE(CHECKOUT_COMPLETE_UV - last_week_CHECKOUT_COMPLETE_UV, last_week_CHECKOUT_COMPLETE_UV) * 100 AS last_week_CHECKOUT_COMPLETE_UV_diff,
        SAFE_DIVIDE(cvr - last_week_cvr, last_week_cvr) * 100 AS last_week_cvr_diff
    FROM (
        SELECT a.*,
            b.gmv AS last_week_gmv,
            b.order_count AS last_week_order_count,
            b.resve_count AS last_week_resve_count,
            b.ASP AS last_week_ASP,
            b.avg_user_resve_cnt AS last_week_avg_user_resve_cnt,
            b.avg_resve_prsnl_cnt AS last_week_avg_resve_prsnl_cnt,
            b.OFFER_DETAIL_UV AS last_week_OFFER_DETAIL_UV,
            b.CHECKOUT_COMPLETE_UV AS last_week_CHECKOUT_COMPLETE_UV,
            b.cvr AS last_week_cvr
        FROM STAT AS a
        LEFT JOIN STAT AS b ON a.basis_dt - 7 = b.basis_dt AND a.gid = b.gid
    )
    ORDER BY basis_dt DESC, GMV DESC
)

-- ⑨ 상품별 최근 1개월 누적
, cumsum_sales AS (
    SELECT *,
        DENSE_RANK() OVER (ORDER BY month1_CUMSUM_gmv DESC) AS RANK_month1_CUMSUM_gmv,
        DENSE_RANK() OVER (ORDER BY month1_CUMSUM_order_count DESC) AS RANK_month1_CUMSUM_order_count
    FROM (
        SELECT s.PRODUCT_ID,
            SUM(CASE WHEN basis_date >= DATE_SUB('{{end_date}}', INTERVAL 1 MONTH) THEN sales_krw_price END) AS month1_CUMSUM_gmv,
            COUNT(DISTINCT CASE WHEN basis_date >= DATE_SUB('{{end_date}}', INTERVAL 1 MONTH) THEN ORDER_ID END) AS month1_CUMSUM_order_count,
            SUM(CASE WHEN basis_date >= DATE_SUB('{{end_date}}', INTERVAL 2 MONTH) THEN sales_krw_price END) AS month2_CUMSUM_gmv,
            COUNT(DISTINCT CASE WHEN basis_date >= DATE_SUB('{{end_date}}', INTERVAL 2 MONTH) THEN ORDER_ID END) AS month2_CUMSUM_order_count,
            AVG(CASE WHEN basis_date >= '{{start_date}}' THEN IFNULL(DATE_DIFF(TRAVEL_START_KST_DATE, BASIS_DATE, DAY), 0) END) AS avg_leadtime,
            APPROX_QUANTILES(CASE WHEN basis_date >= '{{start_date}}' THEN IFNULL(DATE_DIFF(TRAVEL_START_KST_DATE, BASIS_DATE, DAY), 0) END, 100)[OFFSET(50)] AS med_leadtime
        FROM margin AS s
        WHERE kind = 1
            AND BASIS_DATE BETWEEN DATE_SUB('{{end_date}}', INTERVAL 2 MONTH) AND '{{end_date}}'
            AND s.product_id IN (SELECT PRODUCT_ID FROM product)
        GROUP BY 1
    )
)

-- ⑩ 캘린더 오픈율 WoW
, calendar_stat AS (
    SELECT DISTINCT *,
        ROUND(open_rate_15 - last_week_open_rate_15, 2) AS last_week_open_rate_15_diff,
        ROUND(open_rate_16_30 - last_week_open_rate_16_30, 2) AS last_week_open_rate_16_30_diff,
        ROUND(open_rate_31_45 - last_week_open_rate_31_45, 2) AS last_week_open_rate_31_45_diff,
        ROUND(open_rate_46_60 - last_week_open_rate_46_60, 2) AS last_week_open_rate_46_60_diff,
        ROUND(open_rate_61_75 - last_week_open_rate_61_75, 2) AS last_week_open_rate_61_75_diff,
        ROUND(open_rate_76_90 - last_week_open_rate_76_90, 2) AS last_week_open_rate_76_90_diff
    FROM (
        SELECT *,
            LEAD(open_rate_15, 1) OVER (PARTITION BY offer_id ORDER BY basis_date DESC) AS last_week_open_rate_15,
            LEAD(open_rate_16_30, 1) OVER (PARTITION BY offer_id ORDER BY basis_date DESC) AS last_week_open_rate_16_30,
            LEAD(open_rate_31_45, 1) OVER (PARTITION BY offer_id ORDER BY basis_date DESC) AS last_week_open_rate_31_45,
            LEAD(open_rate_46_60, 1) OVER (PARTITION BY offer_id ORDER BY basis_date DESC) AS last_week_open_rate_46_60,
            LEAD(open_rate_61_75, 1) OVER (PARTITION BY offer_id ORDER BY basis_date DESC) AS last_week_open_rate_61_75,
            LEAD(open_rate_76_90, 1) OVER (PARTITION BY offer_id ORDER BY basis_date DESC) AS last_week_open_rate_76_90
        FROM (
            SELECT DATE_TRUNC(basis_date, {{trunc_by}}) basis_date,
                COALESCE(m.gid_30, offer_id) AS offer_id,
                AVG(open_rate_15) open_rate_15,
                AVG(open_rate_16_30) open_rate_16_30,
                AVG(open_rate_31_45) open_rate_31_45,
                AVG(open_rate_46_60) open_rate_46_60,
                AVG(open_rate_61_75) open_rate_61_75,
                AVG(open_rate_76_90) open_rate_76_90
            FROM mrtdata.business.tna_calendar_open_rate_history AS c
            LEFT JOIN tna_migration AS m ON c.offer_id = m.id_20
            WHERE basis_date BETWEEN '{{start_date}}'-7 AND '{{end_date}}'
            GROUP BY 1, 2
        )
    )
    WHERE basis_date BETWEEN '{{start_date}}' AND '{{end_date}}'
)

-- ⑪ 최종 JOIN
SELECT DISTINCT a.* EXCEPT(product_type),
    d.partner_id, d.partner_nm,
    b.* EXCEPT(gid),
    r.* EXCEPT(product_no),
    c.* EXCEPT(PRODUCT_ID),
    ca.* EXCEPT(basis_date, offer_id)
FROM product AS a
LEFT JOIN STAT1 AS b ON a.PRODUCT_ID = b.gid
LEFT JOIN cumsum_sales AS c ON a.product_id = c.product_id
LEFT JOIN review AS r ON a.product_id = r.product_no
LEFT JOIN calendar_stat AS ca ON b.gid = ca.offer_id AND b.basis_dt = ca.basis_date
LEFT JOIN edw_mart.MART_PRODUCT_D AS d ON a.PRODUCT_ID = d.product_id
WHERE b.gid IS NOT NULL OR c.product_id IS NOT NULL OR ca.offer_id IS NOT NULL
ORDER BY basis_dt DESC, GMV DESC
```

> **주의**: 이 쿼리는 FUNT 대시보드 참고 쿼리 #19787을 기반으로 한 것이며, BQ에서 직접 실행 시 파라미터를 실제 값으로 치환해야 한다.

---

## 분석 절차

### Phase 0: 기간 설정

1. **오늘 날짜 확인** (KST 기준). 배치 데이터에서 오늘 제외.
2. **분석 대상 주차 결정**:
   - 기본: 직전 완료 주차 (월~일)
   - 사용자 지정 시 해당 주차 사용
3. **비교 주차**: 분석 대상 주 바로 전주 (WoW)
4. **최근 추이**: 3~4주치 GMV/CM 추이 함께 조회

### Phase 1: 도시별 WoW 지표 (FP&A 시트 기준)

**FP&A 시트에서 먼저 조회:**
- `R_손익_WoW` 탭: 도시별 GMV, CGMV, 확정률, **공헌이익(CM)**
- GMV는 최근 3~4주치 추이 함께 표기

**도시별 기재 항목 (윤지우 양식 — 중첩 테이블):**

| 지표 | **LW** | **TW** | WoW |
|------|--------|--------|-----|
| **CM** (배경 #deebff 또는 증가시 #e3fcef / 하락시 #ffebe6) | `{LW금액}만` | `{TW금액}만` | `▼{%}%` |
| GMV | `{LW금액}` | `{TW금액}` | `{▲/▼}{%}%` |
| CGMV | `{LW금액}` | `{TW금액}` | `{▲/▼}{%}%` |
| CMR (CM% = CM/CGMV) | `{LW}%` | `{TW}%` | `{+/-}{pp}%p` |
| UV | `{LW수}` | `{TW수}` | `{▲/▼}{%}%` |
| CVR (핵심 하락시 #ffebe6) | `{LW}%` | `{TW}%` | `▼{pp}%p` |
| 예약수 | `{LW}건` | `{TW}건` | `{▲/▼}{%}%` |
| 확정률 (하락시 #ffebe6) | `{LW}%` | `{TW}%` | `{▲/▼}{pp}%p` |

[최근 추이]
CM: W{n-3} {금액} → W{n-2} {금액} → W{n-1} {금액} → W{n} {금액}

### Phase 2: 카테고리별 유입/전환/캘린더 이슈 (FUNT 대시보드 기준)

**FP&A 시트 `R_카테고리_WoW` 탭에서 조회 후**, 상품 단위는 BQ FUNT 쿼리로 보완.

**도시별로 아래 구분하여 기재:**

> **중요: 카테고리별 기재는 반드시 CM 절대금액(원) 기준으로 한다.** GMV가 아닌 CM 금액의 WoW 증감을 기재할 것.
> CM 금액 산출: BQ `TNA_FUN_T_category_WoW` 테이블의 `cat_confirm_gmv * cm / 100` (cm 컬럼은 CM%)
> 두 주간(W-1, W0)을 각각 조회하여 CM 금액 WoW를 비교한다.

#### CM 감소 카테고리 (▼)
```
{카테고리명}: {전주CM금액}→{금주CM금액} ({차이금액}) — UV {변화}, CVR {변화}. {원인 한 줄}
```

#### CM 증가 카테고리 (▲)
```
{카테고리명}: {전주CM금액}→{금주CM금액} ({차이금액}) — UV {변화}, CVR {변화}. {견인 요인}
```

**채널별 유입/전환 딥다이브:**
- 유입 채널: 네이버, 앱, 브레이즈, 구글, 직접유입 등
- 전환도 채널별로 분해
- UTM 기반 BQ 쿼리:

```sql
SELECT IFNULL(c.UTM_SOURCE, '(direct/none)') AS utm_source,
    COUNT(DISTINCT c.PID) AS uv,
    SUM(c.CHECKOUT_COMPLETE_FLAG) AS purchases,
    ROUND(SAFE_DIVIDE(SUM(c.CHECKOUT_COMPLETE_FLAG), COUNT(DISTINCT c.PID)) * 100, 2) AS cvr_pct
FROM mrtdata.edw_mart.MART_BIZ_LOG_PID_CONVERSION_D c
JOIN mrtdata.edw_mart.MART_PRODUCT_D p ON c.ITEM_ID = p.GID
WHERE c.OFFER_DETAIL_FLAG = 1
    AND p.COUNTRY_NM = 'Spain'
    AND p.CITY_NM = '{TARGET_CITY}'
    AND p.STANDARD_CATEGORY_LV_1_CD IN ('TOUR','TICKET','ACTIVITY','CLASS','CONVENIENCE','SNAP')
    AND c.BASIS_DATE BETWEEN '{WEEK_START}' AND '{WEEK_END}'
GROUP BY 1 HAVING uv >= 10
ORDER BY uv DESC
```

**캘린더 이슈:**
- `tna_calendar_open_rate_history`에서 15일/30일/45일/60일/75일/90일 오픈율 WoW 비교
- 오픈율 급락 상품 → 캘린더 부족/매진 이슈로 플래그

### Phase 3: 상품 단위 이슈 분석 (카테고리별 구조)

**특이 사항(Phase 2)에서 도출한 CM 감소 카테고리별로**, 해당 카테고리 CM 변동에 기여한 상품들을 상품 단위로 분석한다.

**구조: 특이 사항의 CM 감소 카테고리 순서대로 → 각 카테고리 내 상품 나열**

각 상품에 대해 반드시 기재:
- **상품명** / **상품 GID** / **파트너명** (3개 모두 필수)
- GMV 변동 (전주 → 금주, 증감%)
- CFR 변동 (전주 → 금주, pp 변화) + cancel / wait_confirm 건수 변화
- UV WoW + CVR WoW (체크아웃 진입율은 보지 않는 지표 — 기재하지 말 것)

> **중요**: GMV에 영향을 주는 요소는 UV, CVR, CFR 세 가지다. GMV 하락 상품은 반드시 UV/CVR/CFR을 모두 BQ로 확인하여 어느 요소가 하락 원인인지 특정해야 한다. CFR이 이상 없더라도 "CFR 양호" 라고 명시할 것. "별도 이슈 X"로 넘기지 말 것.

> **딥다이브 원칙**: 도시 단위에서 큰 변화가 있는 지표(예: 확정률 ▼3pp 이상, CVR ▼5% 이상 등)에 대해서는 해당 지표의 **하락 TOP 상품 테이블**을 반드시 추가한다.
> - CVR 하락이 유의미하면 → **CVR 하락 TOP (GMV 가중)** 테이블
> - CFR 하락이 유의미하면 → **CFR 하락 TOP (GM/CM 가중)** 테이블 (cancel/wait 건수 변화 포함)
> - 두 지표 모두 하락이면 → 두 테이블 모두 기재
> 이를 통해 도시 단위 이슈의 상품별 기여도를 한눈에 파악할 수 있다.

**이슈 원인 양식 (카테고리별):**
```
① [{카테고리명} — CM {증감금액}]
{상품명}({GID}/{파트너명}):
GMV {전주}만→{금주}만 ({증감%}) / CFR {전주}→{금주}%
UV {전주}→{금주} (▼{%}%) / CVR {전주}→{금주}% (▼{%}%) / cancel {전주}→{금주}건 / wait {전주}→{금주}건

② [{카테고리명} — CM {증감금액}]
...
```

**CM 증가 카테고리 중에서도 주의할 상품**이 있으면 (예: CFR 하락, 확정대기 누적) 별도 항목으로 추가:
```
③ [{카테고리명} — 확정대기 지속] (CM +{금액}이나 주의)
...
```

**⭐ 성장 상품**은 이슈 원인 마지막에 별도 표기:
```
⭐ [성장] {상품명}({GID}/{파트너명}):
GMV {전주}만→{금주}만 (▲{%}) / CFR {%}
```

### Phase 3-2: 성과 상품 트래킹

CM 감소 상품 분석 후, **CM 증가 카테고리 내 성과 상품**도 별도로 트래킹한다.

**발굴 기준 (OR 조건):**
- GMV WoW 50%↑ & W19 GMV 100만+ (급성장)
- CVR WoW 50%↑ & UV 50+ & GMV 50만+ (전환 개선)
- W18 GMV 0 → W19 100만+ (신규 진입)
- 이미 Phase 3에서 언급한 상품 제외

**각 성과 상품별 기재:**
```
⭐ {상품명} {GID} ({파트너명}/{카테고리}):
GMV {전주}만→{금주}만 (▲{%}) / CFR {%}
UV {전주}→{금주}(▲{%}), CVR {전주}→{금주}%(▲{%})
[채널] {주요 변화 채널}: UV {전주}→{금주}, 구매 {전주}→{금주}건
→ {성장 원인 한 줄}
```

**분석 관점:**
1. 채널별 UV + **구매건수** WoW — UV뿐 아니라 채널별 실제 전환 확인
2. 성장 원인 특정: 상품 자체 전환력 개선 / 특정 채널 유입 증가 / CRM(braze) 효과 등

### Phase 4: 외부 변수 확인

스페인 지역 관련 외부 변수를 웹 검색으로 확인:
- 한국인 스페인 관광객 동향
- 항공편 이슈 (파업, 노선 변경 등)
- 현지 이벤트/축제/공휴일
- 환율 변동 (EUR/KRW)
- 경쟁사 동향
- 기타 여행 수요에 영향을 줄 외부 요인

### Phase 5: 이번 주 액션 아이템 도출

Phase 1~4 분석 결과를 종합하여, **이번 주에 해야 할 액션**을 우선순위 순으로 정리:

- 파트너 컴케 (CFR 이슈, 최저가, 캘린더 등)
- 상품 운영 (메시지박스, 연관상품, 프로모션 페이지 등)
- CRM/마케팅 (발송 대상, 채널 최적화)
- 신규 소싱/온세일
- 기타 (이관, 버그, 프로모션 등)

### Phase 6: 신규 상품 트래킹

최근 등록된 신규 상품의 온세일 상태와 거래 발생 여부를 점검한다.

**조회 기준:**
- `DW_MRT_EXPERIENCES_PRODUCTS` 테이블의 `created_at` 기준 (FIRST_PUBLISHED_KST_DT 아님)
- FIRST_PUBLISHED_KST_DT는 미출시(TEMP) 상품에서 null이므로, 등록일 기준으로 조회해야 TEMP/IN_REVIEW 상품까지 포착 가능

**조회 쿼리:**
```sql
SELECT CAST(p.id AS STRING) AS gid, p.title, p.status,
  DATE(p.created_at) AS created_date,
  DATE(p.updated_at) AS updated_date
FROM mrtdata.edw.DW_MRT_EXPERIENCES_PRODUCTS p
WHERE p.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 4 WEEK)
  AND CAST(p.id AS STRING) IN (
    SELECT GID FROM mrtdata.edw_mart.MART_PRODUCT_D
    WHERE COUNTRY_NM = 'Spain'
      AND CITY_NM IN ('Barcelona','Madrid','Seville','Granada','Palma de Mallorca','Majorca')
      AND STANDARD_CATEGORY_LV_1_CD IN ('TOUR','TICKET','ACTIVITY','CLASS','CONVENIENCE','SNAP')
  )
ORDER BY p.created_at DESC
```

**점검 항목:**
1. 상태별 분류: ONSALE / TEMP / IN_REVIEW / READY_FOR_ONSALE / REJECT
2. ONSALE 상품 중 4월 GMV 발생 여부 → 미발생 시 노출/유입 이슈
3. TEMP/IN_REVIEW 상품 → 온세일 전환 필요 여부 판단
4. 이관 상품 여부: `DW_MRT_EXPERIENCES_PRODUCT_MIGRATION_MAP` 확인
5. 파트너별 분류 (GYG, Tiqets, Tripadvisor, 개인 파트너 등)

**위키 기재:**
- 액션 아이템에 "신규 온세일 상품 중 미거래 건 점검" 포함
- 주요 신규 상품 리스트 (상태별, 도시별)

---

## Confluence 위키 발행 양식

**발행 위치**: 김도연 개인 Confluence space
- Cloud ID: `48148651-ebd8-4545-8a44-ac48cfa693b6`
- Space ID (numeric): `5447550676`
- Space Key: `~7120204122619d974840488f2497e0ce5be72c`

**제목**: `[{YYYY}-{MM}-{DD}] 스페인 주간 지표 분석 (W{주차}: {시작일}~{종료일})`

**양식 참고 페이지**: `pageId: 5875270049` ([2026-05-19] 스페인 주간 지표 분석 W20) — 윤지우 양식 적용

### 위키 양식 (윤지우 양식)

남부유럽팀 주간 회의록의 **Team 윤지우** 양식을 따른다. 핵심 차이점:

1. **주간 지표 셀**: 중첩 테이블 (지표 | LW | TW | WoW) — 값은 `<code>` 태그로 감싸고, CM/CVR 등 핵심 지표 행에 배경색 적용
2. **특이 사항 셀**: **카테고리 단위만** — CM 분해 트리 + ▼ CM 감소 / ▲ CM 증가 카테고리. 상품 상세(GMV/CFR/UV/CVR)는 넣지 않음
3. **이슈 발생 원인 셀**: **상품 단위** — 채널 중첩 테이블 + braze 캠페인별 테이블 + CVR 하락 TOP 5 (GMV 가중) + 카테고리별 상품 드릴다운
4. **체크아웃 진입율은 보지 않는 지표** — CVR(종합)만 기재
5. **확정률**: 화요일 오전 기준으로 재조회하여 업데이트. 변동 있는 상품은 파란색(`<span style="color: #0747a6">`)으로 변경점 표기

### 화요일 오전 CFR 업데이트 (Phase 7)

월요일 발행 후, **화요일 오전에 확정률(CFR)을 재조회**하여 위키를 업데이트한다.
RECENT_STATUS는 현재 시점 편향이 있으므로, 화요일 오전 시점에서 TW·LW 모두 동일 경과 시점으로 비교해야 정확.

**대상 상품**: wait_confirm 건수가 유의미한(5건 이상) 모든 상품 + 훌라트래블 사그라다(3425987)는 매주 필수.

**업데이트 방식 — 블루 글꼴 (`<span style="color: #0747a6">`)**:

1. **특이 사항 셀 — 카테고리 CM 보정**:
```html
몬세라트: 709만→615만 (▼94만, <span style="color: #0747a6">+22만 5/20 → ▼72만</span>)
```

2. **이슈 발생 원인 셀 — 상품별 CFR 업데이트**:
```html
CFR <strong><span style="color: #0747a6">90.3%</span></strong> (5/19: 89.3%→<span style="color: #0747a6">5/20: 90.3%</span>)
wait <strong><span style="color: #0747a6">10→4건</span></strong> (+17만)
```

3. **동일시점 CFR 비교** (사그라다 등 핵심 상품):
```html
<strong><span style="color: #0747a6">화요일 오전 동일시점 CFR: W20 19.4% → W21 16.8%</span></strong>
```

4. **wait 전량 확정 시**:
```html
<strong><span style="color: #0747a6">wait 19→0건 (전부 확정, +22만)</span></strong>
```

**동일시점 CFR 쿼리** (updated_at 기반):
```sql
-- W21 예약 중 화요일 오전(+1일) 시점까지 확정된 건
SELECT
  COUNT(DISTINCT CASE WHEN RECENT_STATUS IN ('confirm','finish')
    AND updated_at <= TIMESTAMP_ADD(TIMESTAMP('2026-05-26'), INTERVAL 9 HOUR)
    THEN RESVE_ID END) AS confirm_by_tue,
  COUNT(DISTINCT RESVE_ID) AS total_resve,
  SUM(CASE WHEN RECENT_STATUS IN ('confirm','finish')
    AND updated_at <= TIMESTAMP_ADD(TIMESTAMP('2026-05-26'), INTERVAL 9 HOUR)
    THEN GMV END) AS cgmv_by_tue,
  SUM(GMV) AS total_gmv
FROM edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE BASIS_DATE BETWEEN '2026-05-19' AND '2026-05-25'
  AND PRODUCT_ID = '{GID}'
  AND FPNA_DOMAIN_NM = 'TNA'
```

**절차:**
1. wait_confirm 5건 이상인 상품 목록 조회
2. 각 상품의 현재 CFR을 월요일 발행 시점 CFR과 비교
3. 변동 있는 상품은 블루 글꼴로 위키 업데이트 (updateConfluencePage)
4. 사그라다(3425987)는 동일시점 CFR WoW 비교 필수 기재
5. 카테고리 단위 CM도 보정값 기재 (특이 사항 셀)

### HTML 발행 구조

`createConfluencePage` 또는 `updateConfluencePage` 사용 시 `contentFormat: "html"`:

```
<table data-width="2896">
  <thead><tr>
    <th colspan="7" data-background="#b3bac5">Team 김도연: 스페인, 마리특 + 상태 로젠지</th>
  </tr></thead>
  <tbody>
    <!-- 컬럼 헤더 행 (배경색 f0f1f2/deebff/e6fcff) -->
    <tr>
      <td>국가/어젠다</td> <td>도시</td>
      <td>주간 목표/달성 사항 (주간 지표) + LW/TW 기간</td>
      <td>특이 사항 (이슈 중심)</td> <td>이슈 발생 원인</td>
      <td>진행한 액션 & 결과</td> <td>이번 주 목표 & 액션 (우선순위 순)</td>
    </tr>
    <!-- 바르셀로나 행 (스페인 rowspan="3") -->
    <tr>
      <td rowspan="3">스페인</td> <td>바르셀로나</td>
      <td><!-- 중첩 테이블: 지표|LW|TW|WoW --></td>
      <td><!-- CM 분해 트리 + ▼▲ 카테고리 --></td>
      <td><!-- 채널 중첩 테이블 + 상품 드릴다운 --></td>
      <td><!-- 비워둠 --></td>
      <td><!-- 액션 ol 리스트 --></td>
    </tr>
    <!-- 마드리드 행, 마요르카 행 -->
    <!-- AI Native 행 (빈칸) -->
  </tbody>
</table>
```

**주간 지표 셀 — 중첩 테이블 형식:**
```html
<table>
  <thead><tr><th>지표</th><th><strong>LW</strong></th><th><strong>TW</strong></th><th>WoW</th></tr></thead>
  <tbody>
    <tr><td data-background="#deebff"><strong>CM</strong></td><td data-background="#deebff"><code>3,049만</code></td>...</tr>
    <!-- CM 증가시 #e3fcef, 하락시 #ffebe6, 핵심 하락지표 #ffebe6 -->
  </tbody>
</table>
<p>[최근 추이] CM: W17 → W18 → W19 → W20</p>
```

**특이 사항 셀 — CM 분해 트리 형식:**
```html
<p><strong>[CM 감소 원인: CVR 하락]</strong></p>
<p>CM ▼13.2% (▼401만) 분해<br>
├── CVR 하락 (▼1.9%p) ···· <strong>최대 기여</strong><br>
├── mktpartner UV↓ (3,019→2,691)<br>
└── UV(▲4%)/CMR(+0.4%p) ·· 상쇄(+)</p>
<p><strong>▼ CM 감소 카테고리</strong></p>
<p>카테고리: 전주CM→금주CM (<strong>▼차이</strong>) — CVR/UV 변화</p>
<p><strong>▲ CM 증가 카테고리</strong></p>
<!-- wait 적체 심각 시 형광펜: <span style="background-color: #f8e6a0">텍스트</span> -->
```

**이슈 발생 원인 셀 — 채널 + braze 캠페인 + CVR TOP5 + 상품 드릴다운:**
```html
<p><strong>[채널별 CVR 하락 분석]</strong></p>
<table>
  <thead><tr><th>UTM 채널</th><th>LW UV</th><th>TW UV</th><th>변화</th><th>CVR 변화</th></tr></thead>
  <tbody><!-- 채널별 행 --></tbody>
</table>

<p><strong>[braze 캠페인별 CVR 하락]</strong></p>
<table>
  <thead><tr><th>캠페인</th><th>LW UV</th><th>TW UV</th><th>LW CVR</th><th>TW CVR</th><th>구매 변화</th></tr></thead>
  <tbody><!-- 알림톡 UC, push-TA, 친구톡, mrt_coupon 등 --></tbody>
</table>

<p><strong>[CVR 하락 TOP 5 — GMV 가중 영향력]</strong></p>
<!-- 구매건수 감소 × ASP = 추정 GMV 손실 기준 정렬 -->
<table>
  <thead><tr><th>상품</th><th>GID</th><th>CVR 변화</th><th>구매 변화</th><th>추정 GMV 손실</th></tr></thead>
  <tbody><!-- TOP 5 --></tbody>
</table>

<p><strong>[CFR 하락 TOP — GM/CM 가중 영향력]</strong></p>
<!-- 확정률 하락이 유의미한(3pp 이상) 도시에서만 표시. cancel/wait 건수 변화 포함 -->
<table>
  <thead><tr><th>상품</th><th>GID</th><th>CFR 변화</th><th>cancel 변화</th><th>wait 변화</th><th>추정 CM 손실</th></tr></thead>
  <tbody><!-- 확정률 하락 기여 상품 --></tbody>
</table>

<p><strong>[카테고리명 — CM ▼금액]</strong></p>
<p>상품명(<code>GID</code>/파트너명):<br>
GMV/CFR/UV/CVR + cancel/wait<br>
→ <strong>원인 특정</strong></p>
```

**주간 분석 테이블 컬럼 설명:**

| 컬럼 | 내용 |
|------|------|
| 국가/어젠다 | `스페인` (rowspan="3"), 이후 빈칸 |
| 도시 | 바르셀로나 / 마드리드 / 마요르카 (세비야/그라나다는 별도 담당자) |
| 주간 지표 | Phase 1 결과 — **중첩 테이블** (지표/LW/TW/WoW) + 최근 추이 |
| 특이 사항 | Phase 2 결과 — **CM 분해 트리** + ▼ CM 감소 / ▲ CM 증가 카테고리 |
| 이슈 발생 원인 | Phase 3 결과 — **채널 중첩 테이블** + 카테고리별 상품 드릴다운 (상품명/GID/파트너명/GMV/CFR/UV/CVR/퍼널 + cancel/wait 건수) |
| 진행한 액션 & 결과 | 유저가 직접 기입 (분석 시점에는 비워둠) |
| 이번 주 목표 & 액션 | Phase 5 결과 (우선순위 순 `<ol>` 액션 리스트, 긴급 시 `<span data-type="status" data-color="red">긴급</span>`) |

**추가 행 (도시 분석 아래):**
- AI Native: 해당 시

---

## 공통 규칙

- 금액 단위: 1억 이상은 `억` 단위(소수 2자리), 1만 이상은 `만` 단위. 상품 단위에서는 원 단위 그대로.
- 비율: 소수 1자리 %
- 증감 표시: **▲**(증가), **▼**(감소), **pp**(퍼센트포인트)
- 배치 데이터에서 오늘 날짜 제외
- 확정 건: `RECENT_STATUS IN ('confirm', 'finish')` — finish 반드시 포함
- 마요르카: `Palma de Mallorca` + `Majorca` 합산
- 파트너 조회 시 도시 분리하지 말고 합산, GMV는 그냥 GMV로 표기
- 슬랙 전송 시 김도연 개인 DM으로만
