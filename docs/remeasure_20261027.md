# 인덱스 재측정 원문 (2026-10-27)

worklog.md의 2026-10-27 정정 항목에 쓰인 측정의 실행 결과 원문이다. pgAdmin이 결과에 덧붙인 따옴표만 정리했고, 내용은 고치지 않았다.

- 환경: PostgreSQL 18.3 on x86_64-windows, compiled by msvc-19.44.35223, 64-bit / shared_buffers 128MB / pgAdmin Query Tool
- 각 측정 5회 실행. 1회차는 캐시를 데우는 용도라 비교에서 뺐다.
- 개선 전: BEGIN; DROP INDEX idx_cd_cat_quarter; → 측정 → ROLLBACK; (Django 서버 정지, 직전 pg_dump 백업)
- ROLLBACK 뒤 pg_indexes 확인 결과(두 번 모두 같음): analysis_commercialdata_pkey, analysis_co_행정동명_b26457_idx, analysis_co_행정동명_d32cd6_idx, analysis_co_행정동명_d97fed_idx, analysis_co_기준_년분기__ca7168_idx, idx_cd_cat_quarter

## 측정 1: 업종 + 분기 조회

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM analysis_commercialdata
WHERE "통합카테고리" = '카페' AND "기준_년분기_코드" = 20244;
```

### 개선 후 (idx_cd_cat_quarter 있음)

1회차 (캐시 빈 상태, 제외)
```
Bitmap Heap Scan on analysis_commercialdata  (cost=8.57..1494.96 rows=405 width=606) (actual time=3.734..41.831 rows=420.00 loops=1)
  Recheck Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
  Heap Blocks: exact=420
  Buffers: shared read=423
  ->  Bitmap Index Scan on idx_cd_cat_quarter  (cost=0.00..8.47 rows=405 width=0) (actual time=1.868..1.868 rows=420.00 loops=1)
        Index Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
        Index Searches: 1
        Buffers: shared read=3
Planning:
  Buffers: shared hit=292 read=41
Planning Time: 16.680 ms
Execution Time: 41.997 ms
```

2회차
```
Bitmap Heap Scan on analysis_commercialdata  (cost=8.57..1494.96 rows=405 width=606) (actual time=0.192..0.729 rows=420.00 loops=1)
  Recheck Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
  Heap Blocks: exact=420
  Buffers: shared hit=423
  ->  Bitmap Index Scan on idx_cd_cat_quarter  (cost=0.00..8.47 rows=405 width=0) (actual time=0.095..0.095 rows=420.00 loops=1)
        Index Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
        Index Searches: 1
        Buffers: shared hit=3
Planning Time: 0.292 ms
Execution Time: 0.806 ms
```

3회차
```
Bitmap Heap Scan on analysis_commercialdata  (cost=8.57..1494.96 rows=405 width=606) (actual time=0.160..0.477 rows=420.00 loops=1)
  Recheck Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
  Heap Blocks: exact=420
  Buffers: shared hit=423
  ->  Bitmap Index Scan on idx_cd_cat_quarter  (cost=0.00..8.47 rows=405 width=0) (actual time=0.079..0.079 rows=420.00 loops=1)
        Index Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
        Index Searches: 1
        Buffers: shared hit=3
Planning Time: 0.215 ms
Execution Time: 0.533 ms
```

4회차
```
Bitmap Heap Scan on analysis_commercialdata  (cost=8.57..1494.96 rows=405 width=606) (actual time=0.149..0.495 rows=420.00 loops=1)
  Recheck Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
  Heap Blocks: exact=420
  Buffers: shared hit=423
  ->  Bitmap Index Scan on idx_cd_cat_quarter  (cost=0.00..8.47 rows=405 width=0) (actual time=0.071..0.071 rows=420.00 loops=1)
        Index Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
        Index Searches: 1
        Buffers: shared hit=3
Planning Time: 0.209 ms
Execution Time: 0.556 ms
```

5회차
```
Bitmap Heap Scan on analysis_commercialdata  (cost=8.57..1494.96 rows=405 width=606) (actual time=0.158..0.478 rows=420.00 loops=1)
  Recheck Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
  Heap Blocks: exact=420
  Buffers: shared hit=423
  ->  Bitmap Index Scan on idx_cd_cat_quarter  (cost=0.00..8.47 rows=405 width=0) (actual time=0.071..0.071 rows=420.00 loops=1)
        Index Cond: ((("통합카테고리")::text = '카페'::text) AND ("기준_년분기_코드" = 20244))
        Index Searches: 1
        Buffers: shared hit=3
Planning Time: 0.239 ms
Execution Time: 0.535 ms
```

### 개선 전 (idx_cd_cat_quarter를 트랜잭션 안에서 DROP)

1회차 (캐시 빈 상태, 제외)
```
Index Scan using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..1693.94 rows=405 width=606) (actual time=1.545..21.713 rows=420.00 loops=1)
  Index Cond: ("기준_년분기_코드" = 20244)
  Filter: (("통합카테고리")::text = '카페'::text)
  Rows Removed by Filter: 14663
  Index Searches: 1
  Buffers: shared hit=420 read=850
Planning:
  Buffers: shared hit=5
Planning Time: 1.354 ms
Execution Time: 21.808 ms
```

2회차
```
Index Scan using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..1693.94 rows=405 width=606) (actual time=0.048..5.637 rows=420.00 loops=1)
  Index Cond: ("기준_년분기_코드" = 20244)
  Filter: (("통합카테고리")::text = '카페'::text)
  Rows Removed by Filter: 14663
  Index Searches: 1
  Buffers: shared hit=1270
Planning Time: 0.228 ms
Execution Time: 5.695 ms
```

3회차
```
Index Scan using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..1693.94 rows=405 width=606) (actual time=0.057..7.533 rows=420.00 loops=1)
  Index Cond: ("기준_년분기_코드" = 20244)
  Filter: (("통합카테고리")::text = '카페'::text)
  Rows Removed by Filter: 14663
  Index Searches: 1
  Buffers: shared hit=1270
Planning Time: 0.283 ms
Execution Time: 7.612 ms
```

4회차
```
Index Scan using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..1693.94 rows=405 width=606) (actual time=0.040..5.599 rows=420.00 loops=1)
  Index Cond: ("기준_년분기_코드" = 20244)
  Filter: (("통합카테고리")::text = '카페'::text)
  Rows Removed by Filter: 14663
  Index Searches: 1
  Buffers: shared hit=1270
Planning Time: 0.204 ms
Execution Time: 5.657 ms
```

5회차
```
Index Scan using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..1693.94 rows=405 width=606) (actual time=0.041..5.181 rows=420.00 loops=1)
  Index Cond: ("기준_년분기_코드" = 20244)
  Filter: (("통합카테고리")::text = '카페'::text)
  Rows Removed by Filter: 14663
  Index Searches: 1
  Buffers: shared hit=1270
Planning Time: 0.209 ms
Execution Time: 5.234 ms
```

## 측정 2: 업종만 거른 최신 분기 조회

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT MAX("기준_년분기_코드") FROM analysis_commercialdata
WHERE "통합카테고리" = '카페';
```

### 개선 후

1회차 (캐시 빈 상태, 제외)
```
Result  (cost=0.44..0.45 rows=1 width=4) (actual time=1.718..1.720 rows=1.00 loops=1)
  Buffers: shared hit=2 read=2
  InitPlan 1
    ->  Limit  (cost=0.42..0.44 rows=1 width=4) (actual time=1.714..1.714 rows=1.00 loops=1)
          Buffers: shared hit=2 read=2
          ->  Index Only Scan Backward using idx_cd_cat_quarter on analysis_commercialdata  (cost=0.42..230.37 rows=10854 width=4) (actual time=1.700..1.700 rows=1.00 loops=1)
                Index Cond: ("통합카테고리" = '카페'::text)
                Heap Fetches: 0
                Index Searches: 1
                Buffers: shared hit=2 read=2
Planning:
  Buffers: shared hit=36 dirtied=1
Planning Time: 3.301 ms
Execution Time: 1.821 ms
```

2회차
```
Result  (cost=0.44..0.45 rows=1 width=4) (actual time=0.043..0.043 rows=1.00 loops=1)
  Buffers: shared hit=4
  InitPlan 1
    ->  Limit  (cost=0.42..0.44 rows=1 width=4) (actual time=0.040..0.040 rows=1.00 loops=1)
          Buffers: shared hit=4
          ->  Index Only Scan Backward using idx_cd_cat_quarter on analysis_commercialdata  (cost=0.42..230.37 rows=10854 width=4) (actual time=0.039..0.039 rows=1.00 loops=1)
                Index Cond: ("통합카테고리" = '카페'::text)
                Heap Fetches: 0
                Index Searches: 1
                Buffers: shared hit=4
Planning Time: 0.274 ms
Execution Time: 0.067 ms
```

3회차
```
Result  (cost=0.44..0.45 rows=1 width=4) (actual time=0.041..0.042 rows=1.00 loops=1)
  Buffers: shared hit=4
  InitPlan 1
    ->  Limit  (cost=0.42..0.44 rows=1 width=4) (actual time=0.038..0.039 rows=1.00 loops=1)
          Buffers: shared hit=4
          ->  Index Only Scan Backward using idx_cd_cat_quarter on analysis_commercialdata  (cost=0.42..230.37 rows=10854 width=4) (actual time=0.037..0.037 rows=1.00 loops=1)
                Index Cond: ("통합카테고리" = '카페'::text)
                Heap Fetches: 0
                Index Searches: 1
                Buffers: shared hit=4
Planning Time: 0.244 ms
Execution Time: 0.065 ms
```

4회차
```
Result  (cost=0.44..0.45 rows=1 width=4) (actual time=0.042..0.042 rows=1.00 loops=1)
  Buffers: shared hit=4
  InitPlan 1
    ->  Limit  (cost=0.42..0.44 rows=1 width=4) (actual time=0.039..0.039 rows=1.00 loops=1)
          Buffers: shared hit=4
          ->  Index Only Scan Backward using idx_cd_cat_quarter on analysis_commercialdata  (cost=0.42..230.37 rows=10854 width=4) (actual time=0.038..0.038 rows=1.00 loops=1)
                Index Cond: ("통합카테고리" = '카페'::text)
                Heap Fetches: 0
                Index Searches: 1
                Buffers: shared hit=4
Planning Time: 0.248 ms
Execution Time: 0.065 ms
```

5회차
```
Result  (cost=0.44..0.45 rows=1 width=4) (actual time=0.043..0.044 rows=1.00 loops=1)
  Buffers: shared hit=4
  InitPlan 1
    ->  Limit  (cost=0.42..0.44 rows=1 width=4) (actual time=0.040..0.041 rows=1.00 loops=1)
          Buffers: shared hit=4
          ->  Index Only Scan Backward using idx_cd_cat_quarter on analysis_commercialdata  (cost=0.42..230.37 rows=10854 width=4) (actual time=0.039..0.039 rows=1.00 loops=1)
                Index Cond: ("통합카테고리" = '카페'::text)
                Heap Fetches: 0
                Index Searches: 1
                Buffers: shared hit=4
Planning Time: 0.250 ms
Execution Time: 0.069 ms
```

### 개선 전

1회차 (캐시 빈 상태, 제외)
```
Result  (cost=4.50..4.51 rows=1 width=4) (actual time=18.348..18.350 rows=1.00 loops=1)
  Buffers: shared hit=3 read=1
  InitPlan 1
    ->  Limit  (cost=0.42..4.50 rows=1 width=4) (actual time=18.341..18.343 rows=1.00 loops=1)
          Buffers: shared hit=3 read=1
          ->  Index Scan Backward using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..44238.38 rows=10854 width=4) (actual time=18.339..18.339 rows=1.00 loops=1)
                Filter: (("통합카테고리")::text = '카페'::text)
                Rows Removed by Filter: 7
                Index Searches: 1
                Buffers: shared hit=3 read=1
Planning:
  Buffers: shared hit=5
Planning Time: 1.210 ms
Execution Time: 18.382 ms
```

2회차
```
Result  (cost=4.50..4.51 rows=1 width=4) (actual time=0.041..0.042 rows=1.00 loops=1)
  Buffers: shared hit=4
  InitPlan 1
    ->  Limit  (cost=0.42..4.50 rows=1 width=4) (actual time=0.038..0.039 rows=1.00 loops=1)
          Buffers: shared hit=4
          ->  Index Scan Backward using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..44238.38 rows=10854 width=4) (actual time=0.037..0.037 rows=1.00 loops=1)
                Filter: (("통합카테고리")::text = '카페'::text)
                Rows Removed by Filter: 7
                Index Searches: 1
                Buffers: shared hit=4
Planning Time: 0.218 ms
Execution Time: 0.109 ms
```

3회차
```
Result  (cost=4.50..4.51 rows=1 width=4) (actual time=0.030..0.031 rows=1.00 loops=1)
  Buffers: shared hit=4
  InitPlan 1
    ->  Limit  (cost=0.42..4.50 rows=1 width=4) (actual time=0.028..0.028 rows=1.00 loops=1)
          Buffers: shared hit=4
          ->  Index Scan Backward using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..44238.38 rows=10854 width=4) (actual time=0.027..0.027 rows=1.00 loops=1)
                Filter: (("통합카테고리")::text = '카페'::text)
                Rows Removed by Filter: 7
                Index Searches: 1
                Buffers: shared hit=4
Planning Time: 0.226 ms
Execution Time: 0.050 ms
```

4회차
```
Result  (cost=4.50..4.51 rows=1 width=4) (actual time=0.030..0.031 rows=1.00 loops=1)
  Buffers: shared hit=4
  InitPlan 1
    ->  Limit  (cost=0.42..4.50 rows=1 width=4) (actual time=0.028..0.028 rows=1.00 loops=1)
          Buffers: shared hit=4
          ->  Index Scan Backward using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..44238.38 rows=10854 width=4) (actual time=0.027..0.027 rows=1.00 loops=1)
                Filter: (("통합카테고리")::text = '카페'::text)
                Rows Removed by Filter: 7
                Index Searches: 1
                Buffers: shared hit=4
Planning Time: 0.219 ms
Execution Time: 0.050 ms
```

5회차
```
Result  (cost=4.50..4.51 rows=1 width=4) (actual time=0.031..0.031 rows=1.00 loops=1)
  Buffers: shared hit=4
  InitPlan 1
    ->  Limit  (cost=0.42..4.50 rows=1 width=4) (actual time=0.028..0.029 rows=1.00 loops=1)
          Buffers: shared hit=4
          ->  Index Scan Backward using "analysis_co_기준_년분기__ca7168_idx" on analysis_commercialdata  (cost=0.42..44238.38 rows=10854 width=4) (actual time=0.027..0.027 rows=1.00 loops=1)
                Filter: (("통합카테고리")::text = '카페'::text)
                Rows Removed by Filter: 7
                Index Searches: 1
                Buffers: shared hit=4
Planning Time: 0.216 ms
Execution Time: 0.050 ms
```
