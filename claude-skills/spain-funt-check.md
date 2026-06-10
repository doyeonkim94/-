# 스페인 FUNT 외부지면 주간 점검

스페인 FUNT 외부지면(네이버 쇼핑광고 + 파워링크) 주간 점검을 실행한다.
BQ 쿼리 기반으로 데이터를 조회하고, 구글 시트를 업데이트한다.

## 시트 정보
- 시트 키: `1JaJGbIp5-e0dtSMwOg95-sNZ2NGa7BZJpH6XsMg80Ck`
- 탭: `스페인 FUNT - 외부지면` (gid=813930738)
- 서비스 계정 키: `C:/Users/User/.claude/claude-sheets-key.json`

## 시트 구조 (2026/4/22 기준)

### 섹션 1. 파워링크 (Row 4~49)
- B=도시, C=n_keyword, D=웹순위, E=랜딩 검색어, F=CVR, G=랜딩URL, H=클로드 이슈
- I열~ = **수동 입력 영역 (절대 건드리지 않음)**
- CVR: 랜딩 URL에서 GID 추출 → BQ CVR WoW로 갱신
- 이슈 자동 판별: 파워링크 미노출 / 키워드-랜딩 불일치 / LLP 랜딩 / q= 너무 넓음

### 섹션 2. SA 쇼핑광고 성과 (Row 53~)
- Row 53: 클로드 자동 업데이트 헤더
- Row 54: B=도시, C=adgroup, D=GID, E=상품명, F=SA imp, G=SA clk, H=SA cost, I=avgrnk, J=최초 노출, K=최종 노출, L=CVR, M=이슈, N=*수동 액션
- Row 55~: SA 데이터 (투어티켓만, 숙박 제외)
- N열 = **수동 입력 영역 (절대 건드리지 않음)**

### 섹션 3. 지면 노출 현황 (SA 섹션 이후)
- 기존 쇼핑 크롤링 데이터. 크롤링 불가하므로 수동 업데이트 영역.
- CVR만 BQ에서 자동 갱신 (G열)
- 수동 액션 열 보존

## 실행 단계

### 날짜 계산
- 오늘 기준으로 직전 완료 주간(월~일)과 그 전주를 자동 계산
- 예: 오늘이 4/22(화) → 이번주 = 4/14(월)~4/20(일), 지난주 = 4/7(월)~4/13(일)

### 1단계: SA 쇼핑광고 성과 (BQ)
```
테이블: ST_ADS_STAT_NAVER_AD + DW_ADS_STAT_NAVER_MASTER_REPORT_SHOPPING_PRODUCT_AD + MART_PRODUCT_D
컬럼명 주의: impCnt(not imp), clkCnt(not click), salesAmt(not cost)
도시 필터: MART_PRODUCT_D.CITY_NM IN ('Barcelona','Madrid','Mallorca')
adgroup 필터: 투어티켓만 (adgroup_name LIKE '%투어티켓%'), 숙박 제외
지표: imp, click, cost, avgrnk(가중평균), 구매수, GMV, ROAS
```

### 2단계: 파워링크 성과 (BQ)
```
테이블: ST_ADS_STAT_NAVER_KEYWORD + MART_SALE_D(RESVE_UTM_SOURCE='NAD')
컬럼명 주의: impCnt, clkCnt, salesAmt
캠페인 필터: campaign_name LIKE '%파워%'
키워드 그룹 필터: adgroup_name 기반 (마케팅팀 키워드 그룹 체계 활용)
  - adgroup_name LIKE '%스페인_%' OR adgroup_name LIKE '%유럽_%'
  - adgroup_name 구조: 파워{M/P}_{지역}_{도시}_{키워드그룹}_{서브그룹}
  - 예: 파워M_스페인_바르셀로나_상품명_가우디투어
  - 이렇게 하면 마케팅팀이 키워드 추가/변경 시 자동 반영
집계 단위: adgroup_name별 (= 키워드 그룹별)
구매 JOIN: CAST(nccKeywordId AS STRING) = RESVE_N_KEYWORD_ID
지표: imp, CTR, CPC, click, cost, ROAS, purchase_cnt
```

### 3단계: CVR WoW (BQ)
```
테이블: MART_BIZ_LOG_PID_CONVERSION_D
키: ITEM_ID = GID (PID가 아님!)
CVR = purchase_uv(CHECKOUT_COMPLETE_FLAG=1) / detail_uv(OFFER_DETAIL_FLAG=1) * 100
대상: SA 섹션의 모든 GID + 지면 노출 섹션의 MRT GID
비교: 직전 완료 주간 vs 그 전주
```

### 4단계: 시트 업데이트

#### 파워링크 (섹션 1)
- B2: 점검 날짜 갱신
- F열(CVR): 랜딩 URL의 GID 기반 CVR WoW 라벨
- H열(이슈): 자동 판별 결과
- I열~ 수동 영역 절대 보존

#### SA 쇼핑광고 (섹션 2) — B안 히스토리 로직
1. BQ에서 이번주 SA 집행 상품(투어티켓, cost>0) 조회
2. 시트의 기존 SA 행 읽기 (D열=GID 기준)
3. **기존에 있는 GID** → F~I열(imp/clk/cost/avgrnk) + K열(최종 노출) 갱신, L열(CVR) 갱신, N열(수동) 보존
4. **신규 GID** → 행 추가, J열(최초 노출)=오늘, K열(최종 노출)=오늘
5. **이번주 안 나온 GID** → 행 삭제하지 않음, 최종 노출 그대로 유지 (히스토리 보존)
6. SA 링크: `https://www.myrealtrip.com/experiences/products/{GID}?utm_campaign=NshoppingTour&utm_medium=CPC&utm_source=NaverShopping`

#### 지면 노출 CVR (섹션 3)
- MRT 상품 링크에서 GID 추출 → G열(CVR) 갱신
- 수동 영역 보존

### 5단계: 이슈 플래그
- 🔴 CVR 3%p 이상 하락 → M열에 마킹
- ⚠️ CVR 0% (상세조회 있는데 구매 0) → 캘린더/상품 상태 점검
- ⚠️ 키워드-랜딩 불일치 (파워링크)
- 파워링크 미노출

### 6단계: 아카이빙
같은 스프레드시트의 아카이브 탭에 이번주 데이터를 append한다.
- `FUNT_Archive_SA` (gid=284386697): 점검일, 주간범위, 도시, adgroup, GID, 상품명, imp, clk, cost, avgrnk, CVR(이번주), CVR(전주), CVR diff
- `FUNT_Archive_PL` (gid=53300849): 점검일, 주간범위, 키워드, 캠페인, keyword_id, imp, clk, cost, CTR%, CPC, avgrnk, 구매수, GMV, ROAS%
- `FUNT_Archive_KW` (gid=1231083317): 점검일, 주간범위, 도시, 키워드그룹, 서브그룹, adgroup_name, 키워드, imp, clk, cost (키워드 그룹별 상세)
- `FUNT_Archive_CVR` (gid=1364418285): 점검일, 주간범위, GID, 이번주 상세UV, 구매UV, CVR%, 전주 상세UV, 구매UV, CVR%, diff
- 기존 데이터 아래에 append (덮어쓰지 않음)
- BQ 쿼리 결과는 반드시 파일로 저장 후 아카이빙에 재사용

### 7단계: 결과 리포트
이슈 요약을 대화로 출력:
- 🔴 CVR 급락 상품 리스트
- ⚠️ CVR 0% 상품 리스트
- 파워링크 이슈 키워드
- SA 신규 진입/이탈 상품

## BQ 컬럼 참고
- `ST_ADS_STAT_NAVER_AD`: basis_dt, nccAdId, campaign_name, adgroup_name, impCnt, clkCnt, salesAmt, avgRnk
- `ST_ADS_STAT_NAVER_KEYWORD`: basis_dt, nccKeywordId, keyword, campaign_name, impCnt, clkCnt, salesAmt, avgRnk
- `DW_ADS_STAT_NAVER_MASTER_REPORT_SHOPPING_PRODUCT_AD`: ad_id, product_id_of_mall
- `MART_PRODUCT_D`: GID, CITY_NM, PRODUCT_NM (PRODUCT_TITLE 아님!)
- `MART_SALE_D`: RESVE_UTM_SOURCE, RESVE_N_AD, RESVE_N_KEYWORD_ID, RESVE_N_CAMPAIGN_TYPE, SALES_KRW_PRICE (RESVE_AMT 아님!)
- `MART_BIZ_LOG_PID_CONVERSION_D`: ITEM_ID(=GID), OFFER_DETAIL_FLAG, CHECKOUT_COMPLETE_FLAG, USER_ID
- `MART_FPNA_NONAIR_PROFIT_D`: BASIS_DATE(not BASIS_DT), RECENT_STATUS(not SALE_STATUS), CON_MARGIN

## 실행 도구
- BQ CLI: `bq query --use_legacy_sql=false --format=csv`
- Python: `python` (not python3, Windows 환경)
- gspread + 서비스 계정 인증
