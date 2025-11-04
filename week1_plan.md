# Week 1: 환경 구축 & 데이터 파이프라인

## 목표
- Airflow 환경 구축 및 설정 완료
- 데이터 수집 파이프라인 구축
- 데이터 전처리 파이프라인 구축

---

## Day 1-2: 프로젝트 셋업

### 1. 프로젝트 구조 생성

```bash
# 프로젝트 디렉토리 구조
mkdir -p ecommerce-recommendation/{airflow/{dags,plugins,logs},data/{raw,processed},models,notebooks,tests}
cd ecommerce-recommendation
```

**디렉토리 구조**:
```
ecommerce-recommendation/
├── airflow/
│   ├── dags/           # Airflow DAG 파일들
│   ├── plugins/        # 커스텀 플러그인
│   └── logs/           # Airflow 로그
├── data/
│   ├── raw/            # 원본 데이터
│   └── processed/      # 전처리된 데이터
├── models/             # 학습된 모델 저장
├── notebooks/          # Jupyter 노트북 (EDA, 실험)
└── tests/              # 테스트 코드
```

### 2. Docker Compose로 Airflow 구축

**docker-compose.yml 구성 요소**:
- Apache Airflow (웹서버, 스케줄러, 워커)
- PostgreSQL (메타데이터 저장소)
- Redis (셀러리 브로커)

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  airflow-webserver:
    image: apache/airflow:2.8.0
    depends_on:
      - postgres
      - redis
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./airflow/logs:/opt/airflow/logs
      - ./airflow/plugins:/opt/airflow/plugins
    ports:
      - "8080:8080"
    command: webserver

  airflow-scheduler:
    image: apache/airflow:2.8.0
    depends_on:
      - postgres
      - redis
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./airflow/logs:/opt/airflow/logs
      - ./airflow/plugins:/opt/airflow/plugins
    command: scheduler

  airflow-worker:
    image: apache/airflow:2.8.0
    depends_on:
      - postgres
      - redis
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./airflow/logs:/opt/airflow/logs
      - ./airflow/plugins:/opt/airflow/plugins
    command: celery worker

volumes:
  postgres-db-volume:
```

### 3. Git 저장소 초기화

```bash
git init
git add .
git commit -m "Initial project setup with Airflow Docker Compose"
```

**.gitignore 설정**:
```
# Airflow
airflow/logs/
airflow/__pycache__/
*.pyc

# Data
data/raw/
data/processed/

# Models
models/*.pkl
models/*.h5

# Environment
.env
venv/
```

### 4. 가상 데이터셋 준비

**데이터셋 소스**:
- Kaggle: E-Commerce Behavior Data
- 예시: [eCommerce behavior data from multi category store](https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store)

**데이터 특징**:
- 이벤트 타입: view, cart, purchase
- 기간: 수개월 간의 유저 행동 로그
- 규모: 수백만 건의 이벤트 데이터

---

## Day 3-4: 데이터 수집 DAG 구축

### 1. DAG 파일 구조

```python
# airflow/dags/01_data_collection_dag.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.dates import days_ago
from datetime import timedelta
import pandas as pd
import psycopg2
from datetime import datetime

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'data_collection_pipeline',
    default_args=default_args,
    description='Collect user behavior data from e-commerce platform',
    schedule_interval='@hourly',  # 매 1시간마다 실행
    start_date=days_ago(1),
    catchup=False,
    tags=['data-collection', 'ecommerce'],
) as dag:

    def extract_user_events(**context):
        """
        유저 행동 로그 추출
        - view: 상품 조회
        - cart: 장바구니 추가
        - purchase: 구매
        - rating: 평점/리뷰
        """
        execution_date = context['execution_date']

        # 실제 환경에서는 API 또는 DB에서 데이터 추출
        # 여기서는 CSV 파일에서 증분 로드
        df = pd.read_csv('/opt/airflow/data/raw/user_events.csv')

        # 시간 범위 필터링 (증분 적재)
        df['timestamp'] = pd.to_datetime(df['timestamp'])
        start_time = execution_date
        end_time = execution_date + timedelta(hours=1)

        df_filtered = df[
            (df['timestamp'] >= start_time) &
            (df['timestamp'] < end_time)
        ]

        # 임시 저장
        output_path = f'/opt/airflow/data/raw/events_{execution_date.strftime("%Y%m%d_%H")}.csv'
        df_filtered.to_csv(output_path, index=False)

        print(f"Extracted {len(df_filtered)} events for {execution_date}")
        return output_path

    def validate_data_quality(**context):
        """
        데이터 품질 검증
        - 필수 컬럼 존재 확인
        - NULL 값 비율 체크
        - 데이터 타입 검증
        """
        ti = context['ti']
        file_path = ti.xcom_pull(task_ids='extract_events')

        df = pd.read_csv(file_path)

        # 필수 컬럼 체크
        required_columns = ['user_id', 'item_id', 'event_type', 'timestamp']
        missing_columns = [col for col in required_columns if col not in df.columns]

        if missing_columns:
            raise ValueError(f"Missing required columns: {missing_columns}")

        # NULL 값 비율 체크 (10% 이하)
        null_ratio = df.isnull().sum() / len(df)
        high_null_columns = null_ratio[null_ratio > 0.1].index.tolist()

        if high_null_columns:
            print(f"Warning: High NULL ratio in columns: {high_null_columns}")

        # 이벤트 타입 검증
        valid_event_types = ['view', 'cart', 'purchase', 'rating']
        invalid_events = df[~df['event_type'].isin(valid_event_types)]

        if len(invalid_events) > 0:
            print(f"Warning: Found {len(invalid_events)} invalid event types")

        print("Data quality check passed!")
        return True

    def load_to_database(**context):
        """
        PostgreSQL에 데이터 적재
        """
        ti = context['ti']
        file_path = ti.xcom_pull(task_ids='extract_events')

        df = pd.read_csv(file_path)

        # PostgreSQL 연결
        conn = psycopg2.connect(
            host='postgres',
            database='airflow',
            user='airflow',
            password='airflow'
        )

        cursor = conn.cursor()

        # 테이블 생성 (존재하지 않을 경우)
        create_table_query = """
        CREATE TABLE IF NOT EXISTS user_events (
            event_id SERIAL PRIMARY KEY,
            user_id INT,
            item_id INT,
            event_type VARCHAR(20),
            timestamp TIMESTAMP,
            session_id VARCHAR(50),
            rating FLOAT,
            price DECIMAL(10,2)
        );
        """
        cursor.execute(create_table_query)

        # 데이터 삽입 (중복 제거)
        for _, row in df.iterrows():
            insert_query = """
            INSERT INTO user_events (user_id, item_id, event_type, timestamp, session_id, rating, price)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT DO NOTHING;
            """
            cursor.execute(insert_query, (
                row['user_id'],
                row['item_id'],
                row['event_type'],
                row['timestamp'],
                row.get('session_id'),
                row.get('rating'),
                row.get('price')
            ))

        conn.commit()
        cursor.close()
        conn.close()

        print(f"Loaded {len(df)} events to database")

    # Task 정의
    extract_task = PythonOperator(
        task_id='extract_events',
        python_callable=extract_user_events,
        provide_context=True,
    )

    validate_task = PythonOperator(
        task_id='validate_data_quality',
        python_callable=validate_data_quality,
        provide_context=True,
    )

    load_task = PythonOperator(
        task_id='load_to_database',
        python_callable=load_to_database,
        provide_context=True,
    )

    # Task 의존성 정의
    extract_task >> validate_task >> load_task
```

### 2. 구현 기능 요약

**시간별 배치 수집**:
- 매 1시간마다 자동 실행
- 스케줄: `@hourly`

**증분 데이터 적재**:
- 이전에 수집한 데이터 중복 방지
- 시간 범위 기반 필터링

**데이터 품질 체크**:
- 필수 컬럼 존재 확인
- NULL 값 비율 모니터링
- 이벤트 타입 유효성 검증

---

## Day 5-7: 전처리 파이프라인

### 1. 전처리 DAG 구조

```python
# airflow/dags/02_preprocessing_dag.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.sensors.external_task import ExternalTaskSensor
from airflow.utils.dates import days_ago
from datetime import timedelta
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler
import psycopg2

default_args = {
    'owner': 'data-team',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'data_preprocessing_pipeline',
    default_args=default_args,
    description='Preprocess user behavior data for recommendation model',
    schedule_interval='@daily',  # 매일 자정 실행
    start_date=days_ago(1),
    catchup=False,
    tags=['preprocessing', 'ecommerce'],
) as dag:

    # 데이터 수집 DAG 완료 대기
    wait_for_collection = ExternalTaskSensor(
        task_id='wait_for_data_collection',
        external_dag_id='data_collection_pipeline',
        external_task_id='load_to_database',
        timeout=600,
        mode='poke',
    )

    def remove_duplicates(**context):
        """
        중복 데이터 제거
        - 동일 유저, 동일 상품, 동일 시간대의 중복 이벤트 제거
        """
        conn = psycopg2.connect(
            host='postgres',
            database='airflow',
            user='airflow',
            password='airflow'
        )

        query = """
        SELECT DISTINCT ON (user_id, item_id, event_type, DATE_TRUNC('minute', timestamp))
            *
        FROM user_events
        ORDER BY user_id, item_id, event_type, DATE_TRUNC('minute', timestamp), timestamp DESC;
        """

        df = pd.read_sql(query, conn)
        conn.close()

        # 중복 제거된 데이터 저장
        df.to_csv('/opt/airflow/data/processed/deduped_events.csv', index=False)

        print(f"Removed duplicates. Remaining events: {len(df)}")
        return len(df)

    def handle_missing_values(**context):
        """
        결측치 처리
        - rating: 결측시 해당 아이템의 평균 평점으로 대체
        - price: 결측시 해당 카테고리의 중간값으로 대체
        """
        df = pd.read_csv('/opt/airflow/data/processed/deduped_events.csv')

        # rating 결측치 처리
        df['rating'] = df.groupby('item_id')['rating'].transform(
            lambda x: x.fillna(x.mean())
        )

        # price 결측치 처리
        df['price'] = df.groupby('item_id')['price'].transform(
            lambda x: x.fillna(x.median())
        )

        # 여전히 결측인 경우 전체 평균/중간값 사용
        df['rating'].fillna(df['rating'].mean(), inplace=True)
        df['price'].fillna(df['price'].median(), inplace=True)

        df.to_csv('/opt/airflow/data/processed/cleaned_events.csv', index=False)

        print("Missing values handled")
        return True

    def create_user_item_matrix(**context):
        """
        유저-아이템 매트릭스 생성
        - 행: 유저
        - 열: 아이템
        - 값: 상호작용 점수 (구매=5, 장바구니=3, 조회=1)
        """
        df = pd.read_csv('/opt/airflow/data/processed/cleaned_events.csv')

        # 이벤트 타입별 가중치
        event_weights = {
            'purchase': 5,
            'cart': 3,
            'view': 1,
            'rating': 4
        }

        df['interaction_score'] = df['event_type'].map(event_weights)

        # 유저-아이템 매트릭스 생성 (집계)
        user_item_matrix = df.groupby(['user_id', 'item_id'])['interaction_score'].sum().reset_index()

        # Pivot 형태로 변환
        matrix_pivot = user_item_matrix.pivot(
            index='user_id',
            columns='item_id',
            values='interaction_score'
        ).fillna(0)

        matrix_pivot.to_csv('/opt/airflow/data/processed/user_item_matrix.csv')

        print(f"User-Item matrix created: {matrix_pivot.shape}")
        return matrix_pivot.shape

    def feature_engineering(**context):
        """
        Feature Engineering

        유저 피처:
        - 구매 빈도
        - 카테고리 선호도
        - 평균 구매 금액
        - 활동 시간대

        아이템 피처:
        - 인기도 (조회수, 구매수)
        - 평균 평점
        - 카테고리
        - 가격 구간
        """
        df = pd.read_csv('/opt/airflow/data/processed/cleaned_events.csv')

        # === 유저 피처 ===
        user_features = df.groupby('user_id').agg({
            'event_id': 'count',  # 총 활동 수
            'item_id': 'nunique',  # 본 상품 수
            'price': ['mean', 'sum'],  # 평균/총 구매 금액
        }).reset_index()

        user_features.columns = ['user_id', 'total_events', 'unique_items', 'avg_price', 'total_spent']

        # 구매 빈도
        purchase_freq = df[df['event_type'] == 'purchase'].groupby('user_id').size().reset_index(name='purchase_count')
        user_features = user_features.merge(purchase_freq, on='user_id', how='left')
        user_features['purchase_count'].fillna(0, inplace=True)

        # 활동 시간대 (새벽/오전/오후/저녁)
        df['timestamp'] = pd.to_datetime(df['timestamp'])
        df['hour'] = df['timestamp'].dt.hour
        df['time_period'] = pd.cut(
            df['hour'],
            bins=[0, 6, 12, 18, 24],
            labels=['dawn', 'morning', 'afternoon', 'evening'],
            include_lowest=True
        )

        # 가장 활동이 많은 시간대
        user_active_time = df.groupby('user_id')['time_period'].agg(lambda x: x.mode()[0] if len(x.mode()) > 0 else 'afternoon').reset_index()
        user_active_time.columns = ['user_id', 'preferred_time']
        user_features = user_features.merge(user_active_time, on='user_id', how='left')

        user_features.to_csv('/opt/airflow/data/processed/user_features.csv', index=False)

        # === 아이템 피처 ===
        item_features = df.groupby('item_id').agg({
            'event_id': 'count',  # 총 조회수
            'user_id': 'nunique',  # 고유 유저 수
            'rating': 'mean',  # 평균 평점
            'price': 'mean',  # 평균 가격
        }).reset_index()

        item_features.columns = ['item_id', 'total_views', 'unique_users', 'avg_rating', 'avg_price']

        # 구매 횟수
        purchase_count = df[df['event_type'] == 'purchase'].groupby('item_id').size().reset_index(name='purchase_count')
        item_features = item_features.merge(purchase_count, on='item_id', how='left')
        item_features['purchase_count'].fillna(0, inplace=True)

        # 인기도 점수 (조회수와 구매수의 가중 평균)
        item_features['popularity_score'] = (
            0.3 * item_features['total_views'] / item_features['total_views'].max() +
            0.7 * item_features['purchase_count'] / item_features['purchase_count'].max()
        )

        # 가격 구간
        item_features['price_range'] = pd.cut(
            item_features['avg_price'],
            bins=[0, 30, 100, 300, np.inf],
            labels=['budget', 'mid', 'premium', 'luxury']
        )

        item_features.to_csv('/opt/airflow/data/processed/item_features.csv', index=False)

        print(f"User features: {user_features.shape}")
        print(f"Item features: {item_features.shape}")

        return {
            'user_features_count': len(user_features),
            'item_features_count': len(item_features)
        }

    # Task 정의
    remove_dup_task = PythonOperator(
        task_id='remove_duplicates',
        python_callable=remove_duplicates,
    )

    handle_missing_task = PythonOperator(
        task_id='handle_missing_values',
        python_callable=handle_missing_values,
    )

    create_matrix_task = PythonOperator(
        task_id='create_user_item_matrix',
        python_callable=create_user_item_matrix,
    )

    feature_eng_task = PythonOperator(
        task_id='feature_engineering',
        python_callable=feature_engineering,
    )

    # Task 의존성
    wait_for_collection >> remove_dup_task >> handle_missing_task >> [create_matrix_task, feature_eng_task]
```

### 2. 전처리 단계 요약

**1단계: 중복 제거**
- 동일 유저, 상품, 이벤트 타입의 중복 제거
- 시간 기준 최신 데이터 유지

**2단계: 결측치 처리**
- Rating: 아이템별 평균으로 대체
- Price: 카테고리별 중간값으로 대체

**3단계: 유저-아이템 매트릭스 생성**
- 이벤트 타입별 가중치 적용
- Sparse Matrix 형태로 저장

**4단계: Feature Engineering**

유저 피처:
- 구매 빈도
- 카테고리 선호도
- 평균 구매 금액
- 선호 활동 시간대

아이템 피처:
- 인기도 점수
- 평균 평점
- 가격 구간
- 고유 구매자 수

---

## 데이터베이스 스키마

### user_events 테이블
```sql
CREATE TABLE user_events (
    event_id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    item_id INT NOT NULL,
    event_type VARCHAR(20) NOT NULL,  -- view, cart, purchase, rating
    timestamp TIMESTAMP NOT NULL,
    session_id VARCHAR(50),
    rating FLOAT,
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 인덱스 생성 (쿼리 성능 향상)
CREATE INDEX idx_user_events_user_id ON user_events(user_id);
CREATE INDEX idx_user_events_item_id ON user_events(item_id);
CREATE INDEX idx_user_events_timestamp ON user_events(timestamp);
CREATE INDEX idx_user_events_event_type ON user_events(event_type);
```

### items 테이블 (상품 메타데이터)
```sql
CREATE TABLE items (
    item_id INT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category VARCHAR(50),
    brand VARCHAR(100),
    price DECIMAL(10,2),
    description TEXT,
    image_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_items_category ON items(category);
CREATE INDEX idx_items_price ON items(price);
```

---

## Week 1 체크리스트

- [ ] **Day 1-2: 프로젝트 셋업**
  - [ ] 프로젝트 디렉토리 구조 생성
  - [ ] Docker Compose로 Airflow 환경 구축
  - [ ] PostgreSQL 및 Redis 설정 확인
  - [ ] Git 저장소 초기화
  - [ ] Kaggle에서 데이터셋 다운로드

- [ ] **Day 3-4: 데이터 수집 DAG**
  - [ ] `01_data_collection_dag.py` 작성
  - [ ] 데이터 추출 함수 구현
  - [ ] 데이터 품질 검증 로직 구현
  - [ ] PostgreSQL 적재 함수 구현
  - [ ] DAG 테스트 및 디버깅

- [ ] **Day 5-7: 전처리 파이프라인**
  - [ ] `02_preprocessing_dag.py` 작성
  - [ ] 중복 제거 로직 구현
  - [ ] 결측치 처리 로직 구현
  - [ ] 유저-아이템 매트릭스 생성
  - [ ] Feature Engineering 구현
  - [ ] 전체 파이프라인 통합 테스트

---

## 환경 설정 파일

### requirements.txt
```
apache-airflow==2.8.0
apache-airflow-providers-postgres==5.7.1
pandas==2.1.4
numpy==1.26.2
scikit-learn==1.3.2
psycopg2-binary==2.9.9
scipy==1.11.4
```

### .env (환경 변수)
```
POSTGRES_USER=airflow
POSTGRES_PASSWORD=airflow
POSTGRES_DB=airflow
REDIS_HOST=redis
REDIS_PORT=6379
```

---

## 실행 방법

### 1. Airflow 시작
```bash
cd ecommerce-recommendation
docker-compose up -d
```

### 2. Airflow 웹 UI 접속
```
http://localhost:8080
기본 계정: airflow / airflow
```

### 3. DAG 활성화
- 웹 UI에서 `data_collection_pipeline` 활성화
- `data_preprocessing_pipeline` 활성화

### 4. 수동 실행 (테스트)
```bash
# DAG 트리거
docker-compose exec airflow-webserver airflow dags trigger data_collection_pipeline
```

---

## 예상 산출물

Week 1 종료 시 다음 데이터가 준비됩니다:

1. **원본 데이터**
   - `/data/raw/user_events.csv`

2. **전처리된 데이터**
   - `/data/processed/deduped_events.csv` - 중복 제거
   - `/data/processed/cleaned_events.csv` - 결측치 처리
   - `/data/processed/user_item_matrix.csv` - 유저-아이템 매트릭스

3. **피처 데이터**
   - `/data/processed/user_features.csv` - 유저 피처
   - `/data/processed/item_features.csv` - 아이템 피처

---

## 트러블슈팅

### Airflow가 시작되지 않을 때
```bash
# 로그 확인
docker-compose logs airflow-webserver

# 데이터베이스 초기화
docker-compose run airflow-webserver airflow db init
```

### PostgreSQL 연결 오류
```bash
# PostgreSQL 상태 확인
docker-compose ps postgres

# 연결 테스트
docker-compose exec postgres psql -U airflow -d airflow
```

### DAG가 보이지 않을 때
```bash
# DAG 파일 위치 확인
ls -la airflow/dags/

# DAG 파싱 오류 확인
docker-compose exec airflow-webserver airflow dags list
```

---

## 다음 주 준비사항

Week 2에서는 전처리된 데이터를 기반으로 추천 모델을 개발합니다:

1. 협업 필터링 (User-based, Item-based, SVD)
2. 컨텐츠 기반 필터링
3. 하이브리드 모델

**필요한 추가 라이브러리**:
- scikit-surprise (협업 필터링)
- implicit (ALS 알고리즘)
- mlflow (모델 관리)

---

**Week 1 목표 달성 기준**:
- Airflow 환경이 정상 작동
- 데이터 수집 DAG가 매시간 자동 실행
- 전처리 DAG가 매일 자정 자동 실행
- 유저-아이템 매트릭스 및 피처 데이터 생성 완료
