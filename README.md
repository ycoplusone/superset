# Apache Superset 완전 개발/운영/테스트 가이드 📊

## 목차

1. [개발 환경 구성](#개발-환경-구성)
2. [쿼리 생성 및 관리](#쿼리-생성-및-관리)
3. [데이터셋 운영 관리](#데이터셋-운영-관리)
4. [차트 개발](#차트-개발)
5. [대시보드 구축](#대시보드-구축)
6. [테스트 전략](#테스트-전략)
7. [운영 및 배포](#운영-및-배포)
8. [성능 최적화](#성능-최적화)
9. [보안 및 권한 관리](#보안-및-권한-관리)
10. [모니터링 및 문제 해결](#모니터링-및-문제-해결)

---

# 1️⃣ 개발 환경 구성

## 1.1 로컬 개발 환경 설정

### Docker로 개발 환경 구성

```bash
#!/bin/bash

# dev-superset 컨테이너 생성 (개발용)
docker run -d \
  -p 8088:8088 \
  --name superset \
  -e SUPERSET_SECRET_KEY="dev-secret-key-12345" \
  -e SUPERSET_LOAD_EXAMPLES="yes" \
  -e SUPERSET_ENV="development" \
  -v superset-data:/var/lib/superset \
  -v $(pwd)/queries:/superset_queries \
  -v $(pwd)/dashboards:/superset_dashboards \
  apache/superset:3.1.0

# 초기화
sleep 30
docker exec superset superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname man \
  --email dev@example.com \
  --password admin --overwrite

docker exec superset-dev superset db upgrade
docker exec superset-dev superset load_examples
```
### 개발 환경 설정(초기화 문제시)
#!/bin/bash

echo "🔄 Superset 에러 해결 중..."

# 1. 캐시 정리
echo "[1/5] 캐시 정리 중..."
docker exec superset superset cache-warm-up 2>/dev/null || true

# 2. DB 업그레이드
echo "[2/5] 데이터베이스 업그레이드 중..."
docker exec superset superset db upgrade

# 3. 초기화
echo "[3/5] Superset 초기화 중..."
docker exec superset superset init 2>/dev/null || true

# 4. 컨테이너 재시작
echo "[4/5] 컨테이너 재시작 중..."
docker restart superset
sleep 10

# 5. 완료
echo "[5/5] 완료!"
echo ""
echo "✅ 에러 해결 완료!"
echo "🌐 http://localhost:8088 에서 새로고침 해주세요"
echo "⏳ 페이지가 로드될 때까지 10초 정도 기다려주세요"


# 2️⃣ 쿼리 생성 및 관리

## 2.1 쿼리 개발 프로세스

### Step 1: SQL Lab에서 쿼리 작성

```
상단 메뉴 → SQL Lab
```

### Step 2: 쿼리 작성 및 테스트

```sql
-- [Step 1] 데이터 탐색 쿼리
SELECT 
    COUNT(*) as total_records,
    COUNT(DISTINCT user_id) as unique_users,
    MIN(created_date) as earliest_date,
    MAX(created_date) as latest_date
FROM orders;

-- [Step 2] 필터 조건 추가
SELECT 
    o.order_id,
    o.user_id,
    u.user_name,
    o.order_date,
    o.total_amount,
    o.status
FROM orders o
JOIN users u ON o.user_id = u.user_id
WHERE 1=1
    AND DATE(o.order_date) >= '{{ start_date | default("2024-01-01") }}'
    AND DATE(o.order_date) <= '{{ end_date | default("2024-12-31") }}'
    {% if status %}
        AND o.status = '{{ status }}'
    {% endif %}
    {% if user_id %}
        AND o.user_id = {{ user_id }}
    {% endif %}
ORDER BY o.order_date DESC
LIMIT {{ limit | default(1000) }};

-- [Step 3] 성능 최적화 (EXPLAIN 확인)
EXPLAIN
SELECT 
    DATE(o.order_date) as date,
    o.status,
    COUNT(*) as order_count,
    SUM(o.total_amount) as total_sales
FROM orders o
WHERE DATE(o.order_date) >= '{{ start_date }}'
GROUP BY 1, 2
ORDER BY 1 DESC;
```

### 쿼리 작성 체크리스트

```
- [ ] 쿼리 실행 성공
- [ ] 결과 데이터 확인
- [ ] 변수 기본값 설정
- [ ] 성능 테스트 (EXPLAIN)
- [ ] 인덱스 활용 여부 확인
- [ ] NULL 값 처리
- [ ] 데이터 타입 일치 확인
- [ ] 쿼리 최적화 완료
```

## 2.2 Saved Query 관리

### 쿼리 저장

```
SQL Lab → Save Query as...
```

**저장 정보**:

```
Query name: "User Order Analysis"
Description: "사용자별 주문 데이터 분석"
Label: ["analytics", "orders"]
```

### 쿼리 버전 관리

```bash
# 쿼리 백업
docker cp superset-dev:/var/lib/superset/superset.db ./superset_backup_$(date +%Y%m%d).db

# 쿼리 내보내기 (API)
curl -X GET http://localhost:8088/api/v1/saved_query/ \
  -H "Authorization: Bearer $TOKEN"
```

### Saved Query 명명 규칙

```
[영역]_[용도]_[버전]
- analytics_sales_daily_v1
- operational_orders_status_v2
- finance_revenue_analysis_v1
```

---

## 2.3 쿼리 테스트

### 테스트 쿼리 예제

```sql
-- 1. 데이터 품질 테스트
SELECT 
    'Record Count' as test_name,
    COUNT(*) as result,
    CASE 
        WHEN COUNT(*) > 0 THEN 'PASS'
        ELSE 'FAIL'
    END as status
FROM orders
UNION ALL
SELECT 
    'Null Check on user_id',
    COUNT(*) as null_count,
    CASE 
        WHEN COUNT(*) = 0 THEN 'PASS'
        ELSE 'FAIL'
    END
FROM orders
WHERE user_id IS NULL
UNION ALL
SELECT 
    'Duplicate Check',
    COUNT(*) - COUNT(DISTINCT order_id) as duplicate_count,
    CASE 
        WHEN COUNT(*) - COUNT(DISTINCT order_id) = 0 THEN 'PASS'
        ELSE 'FAIL'
    END
FROM orders;

-- 2. 날짜 범위 테스트
SELECT 
    MIN(created_date) as min_date,
    MAX(created_date) as max_date,
    DATEDIFF(DAY, MIN(created_date), MAX(created_date)) as days_span
FROM orders;

-- 3. 데이터 분포 테스트
SELECT 
    CASE 
        WHEN total_amount < 100 THEN 'Low'
        WHEN total_amount < 500 THEN 'Medium'
        WHEN total_amount < 1000 THEN 'High'
        ELSE 'Premium'
    END as amount_range,
    COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM orders
GROUP BY CASE 
    WHEN total_amount < 100 THEN 'Low'
    WHEN total_amount < 500 THEN 'Medium'
    WHEN total_amount < 1000 THEN 'High'
    ELSE 'Premium'
END;
```

---

# 3️⃣ 데이터셋 운영 관리

## 3.1 데이터셋 생성

### Step 1: 데이터베이스 연결

```
상단 메뉴 → Data → Databases → + Database
```

**MySQL 연결 설정**:

```
Database name: production_db
SQLAlchemy URI: mysql+pymysql://user:password@host:3306/database
```

### Step 2: 데이터셋 생성

```
Data → Datasets → + Dataset
```

**옵션**:

```
Dataset name: orders
Database: production_db
Schema: public
Table: orders
```

## 3.2 데이터셋 구성 (매우 중요!)

### 컬럼 설정

```
각 컬럼별로:
- 컬럼명
- 데이터 타입 (string, numeric, temporal, etc.)
- 포맷팅 규칙
- 표시 여부
- 필터 활성화
```

### 메트릭 정의

```sql
-- 메트릭 추가 (Dataset 편집 → Metrics)

메트릭 1: Total Sales
SQL: SUM(total_amount)
Function: Sum

메트릭 2: Average Order Value
SQL: AVG(total_amount)
Function: Avg

메트릭 3: Order Count
SQL: COUNT(*)
Function: Count

메트릭 4: Unique Users
SQL: COUNT(DISTINCT user_id)
Function: Count Distinct
```

### 테이블 계산 추가

```
Data → Datasets → [Dataset 편집] → Calculated columns
```

**예제**:

```
Calculated Column 1:
Label: order_year
SQL: YEAR(order_date)
Type: Numeric

Calculated Column 2:
Label: status_category
SQL: CASE 
    WHEN status = 'completed' THEN 'Complete'
    WHEN status = 'pending' THEN 'Pending'
    ELSE 'Other'
END
Type: String
```

## 3.3 데이터셋 캐싱 전략

```python
# superset_config.py에서 캐시 설정

# 데이터셋별 캐시 기간
CACHE_DATASET_TIMEOUT = 3600  # 1시간

# 쿼리 캐시
SQLLAB_CTAS_NO_LIMIT = True

# 캐시 워밍업
SUPERSET_CACHE_WARMING = {
    'CACHE_WARMING_ENABLED': True,
    'CACHE_WARMING_TIMEOUT': 60,
}
```

## 3.4 데이터셋 버전 관리

### 데이터셋 백업

```bash
# 데이터셋 메타데이터 내보내기
docker exec superset-dev superset export_datasets \
  --dataset_ids 1,2,3 \
  --output_file /tmp/datasets_backup.json

# 볼륨 백업
docker run --rm \
  -v superset-dev-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/superset-backup-$(date +%Y%m%d).tar.gz -C /data .
```

### 데이터셋 변경 로그

```
Dataset 편집 시마다:
- 변경 내용 기록
- 버전 번호 업데이트
- 영향받는 차트 확인
```

---

# 4️⃣ 차트 개발

## 4.1 차트 생성 프로세스

### Step 1: 차트 생성

```
+ New → Chart
```

### Step 2: 기본 설정

```
Dataset: orders
Visualization Type: Bar Chart
```

### Step 3: 데이터 설정

```
시각화 유형별 설정:

[Bar Chart]
X-axis: category (또는 DATE(order_date))
Y-axis: SUM(total_amount)
Color: status
Aggregation: Sum

[Line Chart]
X-axis: DATE(order_date)
Y-axis: COUNT(*)
Color: status
Trend line: Yes

[Pie Chart]
Labels: category
Values: SUM(total_amount)

[Table]
Columns: order_id, user_name, status, total_amount
Sorting: order_date DESC
Pagination: 100
```

## 4.2 차트 커스터마이징

### 스타일 설정

```
Customize 탭에서:

- Title: "Daily Order Summary"
- Subtitle: "Last 30 days"
- Color Scheme: "supersetColors"
- Show Legend: True
- X-axis Label: "Date"
- Y-axis Label: "Sales Amount"
- Show Tooltip: True
```

### 고급 설정

```
Advanced 탭에서:

- Query Limit: 10000
- Row Limit: 1000
- Apply Rolling Mean: Yes
- Rolling Window: 7
```

## 4.3 차트 필터 설정

```
Filters 섹션:

Filter 1: Time Range
- Column: order_date
- Operator: Time range
- Default: Last 30 days

Filter 2: Status
- Column: status
- Operator: ==
- Options: completed, pending, shipped

Filter 3: Amount Range
- Column: total_amount
- Operator: >=, <=
- Default: 0, 10000
```

## 4.4 차트 테스트 및 검증

### 테스트 체크리스트

```
- [ ] 데이터 로드 성공
- [ ] 필터 작동 확인
- [ ] 성능 테스트 (5초 이내)
- [ ] 모바일 반응성 확인
- [ ] 다크 모드 확인
- [ ] 다양한 데이터 범위 테스트
- [ ] NULL 값 처리 확인
- [ ] 범례 표시 확인
- [ ] 툴팁 정보 정확성
- [ ] 색상 구분 명확성
```

## 4.5 차트 명명 규칙

```
[영역]_[메트릭]_[차트유형]

예시:
- sales_daily_revenue_bar
- operations_order_status_pie
- finance_quarterly_growth_line
- analytics_user_segments_table
```

---

# 5️⃣ 대시보드 구축

## 5.1 대시보드 생성

### Step 1: 대시보드 생성

```
Dashboards → + DASHBOARD
```

**기본 정보**:

```
Dashboard name: "Sales Operations Dashboard"
Description: "일일 매출 및 운영 현황"
```

### Step 2: 차트 추가

```
Edit Dashboard 모드
→ 좌측 패널에서 차트 선택
→ 드래그 & 드롭으로 배치
```

## 5.2 대시보드 레이아웃 설계

### 12 컬럼 그리드 시스템

```
[ Header - Full Width 12 ]
┌─────────────────────────┐
│   Title & KPI Metrics   │
└─────────────────────────┘

[ Row 1 - 3개 차트 ]
┌─────────┬─────────┬─────────┐
│Chart 1  │Chart 2  │Chart 3  │
│(4 col)  │(4 col)  │(4 col)  │
└─────────┴─────────┴─────────┘

[ Row 2 - 2개 차트 ]
┌───────────────┬───────────────┐
│Chart 4        │Chart 5        │
│(6 col)        │(6 col)        │
└───────────────┴───────────────┘

[ Row 3 - 전체 너비 ]
┌─────────────────────────┐
│Chart 6 (Table)          │
│(12 col)                 │
└─────────────────────────┘
```

### 예제: Sales Operations Dashboard

```
┌─────────────────────────────────────────────┐
│        Sales Operations Dashboard           │
│        Last Updated: 2024-01-15 10:30 AM   │
└─────────────────────────────────────────────┘

┌──────────────┬──────────────┬──────────────┐
│ Total Sales  │ Orders       │ Unique Users │
│ $1.2M        │ 5,234        │ 1,245        │
└──────────────┴──────────────┴──────────────┘

┌──────────────────────┬──────────────────────┐
│ Daily Revenue        │ Order Status         │
│ (Line Chart)         │ (Pie Chart)          │
│ [그래프]             │ [파이]               │
└──────────────────────┴──────────────────────┘

┌──────────────────────┬──────────────────────┐
│ Top Categories       │ Payment Methods      │
│ (Bar Chart)          │ (Bar Chart)          │
│ [그래프]             │ [그래프]             │
└──────────────────────┴──────────────────────┘

┌──────────────────────────────────────────────┐
│ Recent Orders (Table)                        │
│ Order ID | User | Status | Amount | Date    │
│ [테이블]                                     │
└──────────────────────────────────────────────┘
```

## 5.3 대시보드 필터 연결

### 필터 생성

```
Edit Dashboard → Filter 추가
```

**필터 설정**:

```
Filter 1: Date Range
- 적용 대상: 모든 차트
- 기본값: Last 30 days

Filter 2: Status
- 적용 대상: sales, order_status, recent_orders 차트
- 기본값: All

Filter 3: Category
- 적용 대상: top_categories, revenue 차트
- 기본값: All
```

### 필터 매핑

```
각 차트별로:
- SQL 변수와 필터 연결
- 예: {{ date_range }} ↔ Date Range Filter
```

## 5.4 대시보드 성능 최적화

```
# 성능 권장사항

1. 차트 개수 제한: 최대 8-10개
2. 자동 새로고침: 필요한 경우만 활성화
3. 캐시 활용: 자주 변하지 않는 데이터는 캐시 설정
4. 쿼리 최적화: LIMIT, 인덱스 확인
5. 차트 타입 선택: 복잡도 낮은 타입 우선
```

## 5.5 대시보드 공유 및 권한

```
Dashboard 공유:

1. 링크 공유
   - 특정 사용자에게만 공유 가능
   
2. 권한 설정
   - Viewer: 조회만 가능
   - Editor: 편집 가능
   - Admin: 모든 권한

3. 공개 링크
   - Dashboard Settings → Link Sharing
```

---

# 6️⃣ 테스트 전략

## 6.1 쿼리 테스트

### 단위 테스트

```sql
-- Test 1: 데이터 존재 여부
SELECT COUNT(*) as record_count
FROM orders
WHERE 1=1;
-- Expected: > 0

-- Test 2: 필수 컬럼 NULL 체크
SELECT COUNT(*) as null_count
FROM orders
WHERE user_id IS NULL 
   OR order_date IS NULL 
   OR total_amount IS NULL;
-- Expected: 0

-- Test 3: 데이터 타입 확인
SELECT 
    typeof(user_id) as user_id_type,
    typeof(order_date) as date_type,
    typeof(total_amount) as amount_type
FROM orders
LIMIT 1;
-- Expected: numeric, date, decimal

-- Test 4: 날짜 범위 확인
SELECT 
    MIN(order_date) as min_date,
    MAX(order_date) as max_date
FROM orders;
-- Expected: 유효한 날짜 범위
```

### 성능 테스트

```bash
# 쿼리 실행 시간 측정
#!/bin/bash

QUERY="SELECT COUNT(*) FROM orders WHERE order_date >= '2024-01-01'"

# 시간 측정
time mysql -h localhost -u user -p database -e "$QUERY"

# EXPLAIN으로 실행 계획 확인
mysql -h localhost -u user -p database -e "EXPLAIN $QUERY"
```

## 6.2 차트 테스트

### 테스트 케이스

```
Test Case 1: 기본 데이터 표시
- 입력: 기본 필터값
- 기대: 차트 정상 표시
- 확인: 데이터 개수, 색상, 레이블

Test Case 2: 필터 적용
- 입력: 다양한 필터 조합
- 기대: 데이터 동적 변경
- 확인: 필터 적용 정확성

Test Case 3: 응답성
- 입력: 큰 데이터셋 (100K+ 레코드)
- 기대: 5초 이내 로드
- 확인: 차트 렌더링 시간

Test Case 4: 모바일 호환성
- 입력: 다양한 화면 크기
- 기대: 반응형 레이아웃
- 확인: 모바일, 태블릿, 데스크톱
```

## 6.3 대시보드 테스트

### 통합 테스트

```
Test 1: 대시보드 로드
- 모든 차트 정상 로드
- 필터 옵션 표시
- 성능 측정 (10초 이내)

Test 2: 필터 통합
- 필터 변경 시 관련 차트 업데이트
- 필터 기본값 설정 확인
- 필터 초기화 기능 테스트

Test 3: 사용자 권한
- Viewer: 조회만 가능
- Editor: 편집 가능
- Anonymous: 공개 대시보드만 접근

Test 4: 브라우저 호환성
- Chrome, Firefox, Safari 테스트
- 다크 모드 지원 확인
```

### 자동화 테스트 (선택사항)

```python
# pytest 예제
import pytest
from selenium import webdriver

@pytest.fixture
def driver():
    driver = webdriver.Chrome()
    yield driver
    driver.quit()

def test_dashboard_loads(driver):
    driver.get("http://localhost:8088/dashboard/1")
    assert driver.find_element("class name", "dashboard") is not None

def test_filter_changes_chart(driver):
    driver.find_element("id", "status-filter").send_keys("completed")
    driver.find_element("id", "apply-filter").click()
    # 차트 데이터 변경 확인
    assert driver.find_element("class name", "chart-container") is not None
```

---

# 7️⃣ 운영 및 배포

## 7.1 개발 → 테스트 → 프로덕션

### 환경 구성

```
개발 (dev-superset)
↓ (쿼리/차트 검증)
테스트 (test-superset)
↓ (성능/통합 테스트)
프로덕션 (superset-prod)
```

### 배포 체크리스트

```
[ Development ]
- [ ] 쿼리 작성 및 테스트
- [ ] 변수 및 기본값 설정
- [ ] 데이터 품질 검증

[ Testing ]
- [ ] 성능 테스트
- [ ] 데이터 정확성 확인
- [ ] 필터 동작 검증
- [ ] 보안 권한 테스트

[ Production ]
- [ ] 대시보드 배포
- [ ] 사용자 권한 설정
- [ ] 캐시 전략 적용
- [ ] 모니터링 활성화
```

## 7.2 설정 내보내기/임포트

### 대시보드 내보내기

```bash
# API를 통한 대시보드 내보내기
curl -X GET http://localhost:8088/api/v1/dashboard/1 \
  -H "Authorization: Bearer $TOKEN" \
  > dashboard_backup.json

# 메타데이터 내보내기
docker exec superset-prod superset export_dashboards \
  --dashboard_ids 1,2,3 \
  --output_file /tmp/dashboards.json
```

### 대시보드 임포트

```bash
# 대시보드 복사
docker exec superset-dev superset import_dashboards \
  --path /tmp/dashboards.json
```

## 7.3 마이그레이션 전략

### 데이터베이스 마이그레이션

```bash
# 1. 현재 버전 확인
docker exec superset-prod superset version

# 2. 데이터 백업
docker exec superset-prod superset db upgrade --status

# 3. 데이터 마이그레이션
docker exec superset-prod superset db upgrade

# 4. 데이터 검증
docker exec superset-prod superset db upgrade --revision
```

---

# 8️⃣ 성능 최적화

## 8.1 쿼리 최적화

### 인덱스 전략

```sql
-- 자주 필터링되는 컬럼
CREATE INDEX idx_order_status ON orders(status);
CREATE INDEX idx_order_date ON orders(order_date);
CREATE INDEX idx_user_id ON orders(user_id);

-- 복합 인덱스
CREATE INDEX idx_order_date_status ON orders(order_date, status);

-- 인덱스 확인
SHOW INDEX FROM orders;
```

### 쿼리 최적화 기법

```sql
-- ❌ 나쁜 쿼리
SELECT * FROM orders o
JOIN users u ON o.user_id = u.user_id
WHERE YEAR(o.order_date) = 2024;

-- ✅ 좋은 쿼리
SELECT 
    o.order_id,
    o.user_id,
    u.user_name,
    o.order_date,
    o.total_amount
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id
WHERE o.order_date >= '2024-01-01' 
  AND o.order_date < '2025-01-01'
LIMIT 10000;

-- 이유:
-- 1. YEAR() 함수 제거 (인덱스 활용 불가)
-- 2. 필요한 컬럼만 선택
-- 3. INNER JOIN 사용 (성능 개선)
-- 4. DATE 범위 필터 (인덱스 활용 가능)
```

## 8.2 캐싱 전략

### 데이터셋 캐싱

```python
# superset_config.py

# 데이터셋 캐시 설정
SQLLAB_CTAS_NO_LIMIT = True
CACHE_DATASET_TIMEOUT = 3600  # 1시간

# 쿼리 결과 캐싱
SUPERSET_SQLLAB_ASYNC_TIME_LIMIT_SEC = 300
SQLLAB_DEFAULT_FETCH_LIMIT = 1000
```

### 캐시 워밍업

```bash
# 자주 사용되는 쿼리 캐시 미리 생성
docker exec superset-prod superset cache-warm-up

# 특정 차트의 캐시만 워밍
docker exec superset-prod superset refresh-dataset \
  --dataset-id 1,2,3
```

## 8.3 성능 모니터링

### 쿼리 성능 모니터링

```bash
# 로그에서 느린 쿼리 확인
docker logs superset-prod | grep -i "slow" | tail -20

# 데이터베이스 슬로우 로그 확인
mysql -u user -p -e "SET GLOBAL slow_query_log = 'ON';"
```

---

# 9️⃣ 보안 및 권한 관리

## 9.1 사용자 및 역할 관리

### 사용자 생성

```
Settings → Users → + User
```

**사용자 정보**:

```
Username: john_doe
Email: john@example.com
First Name: John
Last Name: Doe
Role: Editor
Active: Yes
```

### 역할 정의

```
Settings → Roles → + Role

Role: Sales Manager
Permissions:
- Datasource Access: sales_data
- Dashboard: sales_dashboard (edit)
- Chart: all charts (view)
```

## 9.2 행 수준 보안 (RLS)

### Row Level Security 설정

```
Data → Datasets → [dataset] → Row Level Security
```

**RLS 규칙**:

```
Rule 1:
- Clause: user_id = {{ current_user_id }}
- User: All users

Rule 2:
- Clause: region = '{{ current_user_region }}'
- User: regional_managers

Rule 3:
- Clause: 1=1 (제한 없음)
- User: admin
```

## 9.3 데이터 마스킹

### 민감한 데이터 보호

```
Dataset 편집 → Columns → [민감한 컬럼]
- Certification: Sensitive
- Description: "PII - 제한됨"
- Filterable: No (필터링 불가)
```

---

# 🔟 모니터링 및 문제 해결

## 10.1 로그 모니터링

### 로그 확인

```bash
# 실시간 로그
docker logs -f superset-prod

# 특정 오류 필터링
docker logs superset-prod | grep -i "error"

# 경고 메시지
docker logs superset-prod | grep -i "warning"

# 마지막 100줄
docker logs superset-prod | tail -100
```

### 로그 분석 예제

```bash
# 느린 쿼리 찾기
docker logs superset-prod | grep "Query duration"

# 실패한 쿼리
docker logs superset-prod | grep -i "failed\|error" | grep -i "query"

# 캐시 관련 로그
docker logs superset-prod | grep -i "cache"
```

## 10.2 성능 문제 해결

### 느린 대시보드

```
1. 문제 진단
   - 각 차트의 로드 시간 측정
   - 데이터베이스 쿼리 시간 확인
   - 캐시 히트율 확인

2. 해결 방법
   - 쿼리 최적화 (LIMIT, 인덱스)
   - 캐시 설정 조정
   - 차트 개수 감소
   - 필터 기본값 변경
```

### 메모리 부족

```bash
# Docker 메모리 제한 확인
docker stats superset-prod

# 메모리 제한 설정
docker update --memory 4g superset-prod
docker update --memory-swap 6g superset-prod
```

### 데이터베이스 연결 오류

```bash
# 1. 데이터베이스 연결 테스트
docker exec superset-prod superset test-connection \
  --database_id 1

# 2. 연결 문자열 확인
# Data → Databases → [database] → Test Connection

# 3. 네트워크 확인
docker network inspect superset_default
```

## 10.3 데이터 정확성 검증

### 데이터 검증 쿼리

```sql
-- 1. 일일 데이터 검증
SELECT 
    DATE(created_date) as date,
    COUNT(*) as record_count,
    COUNT(DISTINCT user_id) as unique_users,
    SUM(amount) as total_amount
FROM orders
GROUP BY DATE(created_date)
ORDER BY date DESC
LIMIT 30;

-- 2. 데이터 이상 감지
SELECT 
    DATE(created_date) as date,
    COUNT(*) as count,
    LAG(COUNT(*)) OVER (ORDER BY DATE(created_date)) as prev_count,
    ROUND(100.0 * (COUNT(*) - LAG(COUNT(*)) OVER (ORDER BY DATE(created_date))) / LAG(COUNT(*)) OVER (ORDER BY DATE(created_date)), 2) as change_percent
FROM orders
GROUP BY DATE(created_date)
HAVING ABS(ROUND(100.0 * (COUNT(*) - LAG(COUNT(*)) OVER (ORDER BY DATE(created_date))) / LAG(COUNT(*)) OVER (ORDER BY DATE(created_date)), 2)) > 50
ORDER BY date DESC;

-- 3. 데이터 품질 리포트
SELECT 
    'Total Records' as metric,
    COUNT(*) as value
FROM orders
UNION ALL
SELECT 
    'Records with NULL user_id',
    COUNT(*)
FROM orders
WHERE user_id IS NULL
UNION ALL
SELECT 
    'Duplicate Records',
    COUNT(*) - COUNT(DISTINCT order_id)
FROM orders;
```

---

# 📋 통합 운영 체크리스트

## 일일 작업

```
[ 08:00 AM ]
- [ ] 시스템 상태 확인
- [ ] 새로운 에러 로그 검토
- [ ] 대시보드 성능 확인

[ 09:00 AM ]
- [ ] 데이터 신선도 확인
- [ ] 쿼리 실행 시간 모니터링
- [ ] 캐시 상태 확인

[ 05:00 PM ]
- [ ] 일일 사용자 로그 분석
- [ ] 성능 메트릭 수집
- [ ] 아직 해결되지 않은 이슈 검토
```

## 주간 작업

```
[ 월요일 ]
- [ ] 주간 계획 검토
- [ ] 새로운 쿼리/차트 검증

[ 목요일 ]
- [ ] 성능 리포트 생성
- [ ] 사용자 피드백 검토

[ 금요일 ]
- [ ] 주간 백업 확인
- [ ] 다음 주 계획 수립
```

## 월간 작업

```
- [ ] 전체 시스템 성능 리뷰
- [ ] 사용자 권한 감사
- [ ] 데이터 품질 감사
- [ ] 보안 업데이트 확인
- [ ] 용량 계획 검토
- [ ] 아키텍처 최적화 검토
```

---

# 🎯 모범 사례 (Best Practices)

## 쿼리 작성

```
1. SQL 코멘트 추가
2. 변수 이름은 명확하게
3. 기본값 항상 설정
4. LIMIT 값 설정
5. NULL 처리
6. 인덱스 활용
```

## 데이터셋 관리

```
1. 명확한 컬럼명 사용
2. 메트릭 미리 정의
3. 캐시 전략 수립
4. 주기적 검증
5. 문서화
```

## 차트 설계

```
1. 단순성 우선
2. 색상 일관성 유지
3. 레이블 명확하게
4. 성능 고려
5. 모바일 반응성
```

## 대시보드 구성

```
1. 정보 위계 설정
2. 필터 위치 통일
3. 새로고침 주기 최적화
4. 접근성 고려 (색맹 등)
5. 반응형 레이아웃
```

---

# 📞 트러블슈팅 가이드

## 문제: "Unable to connect to database"

```bash
# 해결 순서
1. 데이터베이스 서버 상태 확인
   docker exec superset-prod ping database-host

2. 네트워크 연결 확인
   docker exec superset-prod nc -zv database-host:3306

3. 자격증명 확인
   Data → Databases → [database] → Test Connection

4. 로그 확인
   docker logs superset-prod | grep -i "database"
```

## 문제: "Query timeout"

```bash
# 해결 방법
1. 쿼리 최적화 (인덱스, LIMIT 추가)
2. 타임아웃 값 증가
3. 캐시 활성화
4. 데이터 파티션
```

## 문제: "Out of memory"

```bash
# 해결 방법
1. 메모리 한계 증가
   docker update --memory 8g superset-prod

2. LIMIT 값 감소
   SET SUPERSET_SQLLAB_DEFAULT_FETCH_LIMIT = 500

3. 캐시 정리
   docker exec superset-prod superset cache-clear
```

---

# 📚 참고 자료

- 공식 문서: https://superset.apache.org/docs/
- GitHub: https://github.com/apache/superset
- API 문서: http://localhost:8088/swagger/v1
- 커뮤니티: https://github.com/apache/superset/discussions

---

**문제가 발생하거나 궁금한 점이 있으면 로그를 확인하고 위의 트러블슈팅 가이드를 참고하세요!** 🚀