# 노다지 DB 재작업 로그

## 2026-08-14 (Day 1)

### 대상 파악
- DB: PostgreSQL, Django ORM
- 핵심 테이블: analysis 5개 (CommercialData, StreetCommercialData,
  ScoreData, StreetScoreData, StoreInfo)
- CommercialData: 408,576행 / 컬럼 약 100개 / 카테고리 51종 / 분기 27개
- StoreInfo: 534,987행

---

### 발견 #1: 지역 키 체계가 두 축으로 분리
**상황**
- 행정동 축: CommercialData (행정동코드 + 행정동명)
- 상권 축: StreetCommercialData, StreetScoreData, StoreInfo (상권_코드)
- 두 축을 잇는 FK나 매핑 테이블 없음
- ScoreData에는 행정동코드 컬럼 자체가 없고 행정동명만 존재

**검증**
- StoreInfo ↔ CommercialData 행정동명 매칭 실패 5개 동
  → 신설동 1,242 / 용두동 1,017 / 상일1동 671 / 개포3동 329 / 상일2동 144
  → 합계 3,403건 = 전체 534,987건의 0.64%
- 행정동명별 DISTINCT 행정동코드 집계 → '신사동'이 코드 2개 (강남구/관악구)

**판단**
매칭 실패보다 매칭 성공 쪽이 위험. 신사동은 조인이 성공하므로
에러 없이 다른 자치구 데이터가 결합됨.
views.py 확인 결과 실제 결합은 SQL JOIN이 아니라 Python dict로 발생:
  score_map = {row["행정동명"]: row for row in ScoreData...}
dict 키가 행정동명이므로 신사동 한쪽이 조용히 덮어써짐.
ScoreData에 코드 컬럼이 없어 데이터 차원에서 구분 불가.

**조치 (예정)**
ScoreData에 행정동코드 추가 → dict 키를 코드로 전환

---

### 발견 #2: 정규화 로직이 경계에만 있고 데이터에는 없음
views.py의 DONG_REMAP:
  {"신설동": "용신동", "상일제1동": "상일동", "상일제2동": "상일동"}

- 용두동, 상일1동, 상일2동, 개포3동 미포함
- normalize_dong()이 API 요청 파라미터에만 적용되고
  StoreInfo 테이블 데이터 자체에는 미적용
- 따라서 StoreInfo.행정동명을 직접 쓰는 API(subcategory_trend 등)는
  여전히 어긋남

**원인 추정**
법정동/행정동 혼용 (신설동·용두동은 동대문구 법정동, 행정동은 용신동)
+ 행정동 개편 시점 불일치 (상일동 분동, 개포동 개편)

---

### 발견 #3: 단일 컬럼 인덱스의 필터 낭비 → 개선 완료

**측정 쿼리**
WHERE 통합카테고리='카페' AND 기준_년분기_코드=20244

| 지표 | 개선 전 | 개선 후 |
|---|---|---|
| 실행 계획 | Index Scan (기준_년분기_코드 단일) | Bitmap Heap Scan (복합) |
| 반환 행 | 420 | 420 |
| Rows Removed by Filter | 14,663 | 0 |
| Buffers read | 1,260 | 3 |
| Execution Time | 39.235 ms | 0.840 ms |

**조치**
CREATE INDEX idx_cd_cat_quarter
ON analysis_commercialdata ("통합카테고리", "기준_년분기_코드");

**컬럼 순서 근거**
카디널리티: 통합카테고리 51 > 기준_년분기_코드 27
카테고리 선행 시 408,576/51 ≈ 8,011행, 분기 선행 시 ≈ 15,132행
선택도 높은 컬럼을 앞에 배치해 탐색 범위 절반으로 축소

**배운 것**
"인덱스가 없다"가 아니라 "인덱스가 조건을 다 못 덮는다"가 문제였음.
Rows Removed by Filter가 그 신호. 실행 시간은 캐시에 따라 흔들리지만
Buffers read는 흔들리지 않으므로 이쪽이 더 신뢰할 지표.

---

### 남은 작업
- [ ] 수동 DDL을 Django 마이그레이션으로 이전 (현재 코드에 없음 → 배포 시 소실)
- [ ] recommend_industry API 레벨 응답 시간 측정 (카테고리 51개 × 전체 스캔 루프)
- [ ] ScoreData에 행정동코드 추가

### 손대지 않기로 한 것 (문서에만 기록)
- 컬럼 100개 정규화 (반복 그룹 3세트) — 앱 코드 영향 범위 과다
- CommercialData/StreetCommercialData 테이블 통합 — 동일 사유
- 한글 컬럼명 — 전면 수정 시 코드 전체 파손
- gu_report의 빈 for 루프, _REPORT_CACHE 무한 증가,
  _NAVER_TREND_KEYWORDS의 "카페" 중복 키