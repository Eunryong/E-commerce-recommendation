# Week 1: 환경 구축 & 데이터 파이프라인 상세 가이드

---

## 📅 Day 1-2: 프로젝트 셋업

### Step 1: 프로젝트 디렉토리 구조 생성

```bash
# 프로젝트 루트 생성
mkdir ecommerce-recommendation
cd ecommerce-recommendation

# 디렉토리 구조 생성
mkdir -p airflow/{dags,plugins,logs,config}
mkdir -p data/{raw,processed,features}
mkdir -p models/{saved_models,experiments}
mkdir -p notebooks
mkdir -p api
mkdir -p tests
mkdir -p scripts
mkdir -p config
mkdir -p monitoring

# 확인
tree -L 2
```

**디렉토리 설명**:
- `airflow/dags`: Airflow DAG 파일들
- `airflow/plugins`: 커스텀 Airflow 플러그인
- `data/raw`: 원본 데이터
- `data/processed`: 전처리된 데이터
- `models`: 학습된 모델 저장
- `notebooks`: 탐색적 데이터 분석 (EDA)
- `api`: FastAPI 서비스 코드
- `scripts`: 유틸리티 스크립트

---

### Step 2: Docker Compose로 Airflow 환경 구축

```bash
# docker-compose.yml 생성
vi docker-compose.yml
```

```yaml
version: '3.8'

x-airflow-common:
  &airflow-common
  image: apache/airflow:2.8.0-python3.10
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth'
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'
    _PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:-}
  volumes:
    - ./airflow/dags:/opt/airflow/dags
    - ./airflow/logs:/opt/airflow/logs
    - ./airflow/plugins:/opt/airflow/plugins
    - ./data:/opt/airflow/data
    - ./models:/opt/airflow/models
    - ./scripts:/opt/airflow/scripts
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on:
    &airflow-common-depends-on
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5
      start_period: 5s
    restart: always
    ports:
      - "5432:5432"

  redis:
    image: redis:7.2-alpine
    expose:
      - 6379
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 50
      start_period: 30s
    restart: always

  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    command:
      - -c
      - |
        mkdir -p /sources/logs /sources/dags /sources/plugins
        chown -R "${AIRFLOW_UID}:0" /sources/{logs,dags,plugins}
        exec /entrypoint airflow version
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_MIGRATE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
      _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
    user: "0:0"
    volumes:
      - ./airflow:/sources

volumes:
  postgres-db-volume:
```

---

### Step 3: 환경 변수 설정

```bash
# .env 파일 생성
vi .env
```

```bash
# Airflow 설정
AIRFLOW_UID=50000
AIRFLOW_GID=0
_AIRFLOW_WWW_USER_USERNAME=airflow
_AIRFLOW_WWW_USER_PASSWORD=airflow1234

# PostgreSQL 설정
POSTGRES_USER=airflow
POSTGRES_PASSWORD=airflow
POSTGRES_DB=airflow

# 추가 패키지
_PIP_ADDITIONAL_REQUIREMENTS=scikit-surprise pandas numpy scipy scikit-learn
```

---

### Step 4: Airflow 실행

```bash
# Docker Compose 실행
docker-compose up -d

# 로그 확인
docker-compose logs -f airflow-webserver

# 서비스 상태 확인
docker-compose ps
```

**접속 확인**:
- Airflow UI: http://localhost:8080
- ID: airflow
- PW: airflow1234

---

### Step 5: Python 가상환경 및 라이브러리 설치

```bash
# 가상환경 생성
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# requirements.txt 생성
vi requirements.txt
```

```txt
# Data Processing
pandas==2.1.4
numpy==1.26.2
scipy==1.11.4

# ML & Recommendation
scikit-learn==1.3.2
scikit-surprise==1.1.3

# Airflow
apache-airflow==2.8.0
apache-airflow-providers-postgres==5.10.0

# Database
psycopg2-binary==2.9.9
SQLAlchemy==2.0.23

# API
fastapi==0.109.0
uvicorn==0.25.0
redis==5.0.1

# Monitoring & MLOps
mlflow==2.9.2

# Utilities
python-dotenv==1.0.0
pyyaml==6.0.1
```

```bash
# 설치
pip install -r requirements.txt
```

---

### Step 6: 데이터셋 다운로드

```bash
# Kaggle API 설치 (이미 설치했다면 스킵)
pip install kaggle

# Kaggle 인증 설정
mkdir -p ~/.kaggle
vi ~/.kaggle/kaggle.json
```

```json
{
  "username": "your_kaggle_username",
  "key": "your_kaggle_api_key"
}
```

```bash
# 권한 설정
chmod 600 ~/.kaggle/kaggle.json

# 데이터셋 다운로드
cd data/raw

# eCommerce behavior data from multi category store
kaggle datasets download -d mkechinov/ecommerce-behavior-data-from-multi-category-store

# 압축 해제
unzip ecommerce-behavior-data-from-multi-category-store.zip

# 확인
ls -lh
```

**데이터셋 구조**:
```
2019-Oct.csv  # 약 42GB
2019-Nov.csv  # 약 67GB
```

---

### Step 7: Git 초기화 및 기본 설정

```bash
# Git 초기화
cd ~/ecommerce-recommendation
git init

# .gitignore 생성
vi .gitignore
```

```txt
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
env/

# Airflow
airflow/logs/
airflow.db
airflow.cfg

# Data
data/raw/*.csv
data/processed/*.csv
*.parquet

# Models
models/saved_models/*
!models/saved_models/.gitkeep

# Jupyter
.ipynb_checkpoints/
*.ipynb

# Environment
.env
.env.local

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
```

```bash
# 첫 커밋
git add .
git commit -m "Initial project setup with Airflow and Docker"
```

---

## 📅 Day 3-4: 데이터 수집 DAG 구축

### Step 1: 데이터 탐색 (EDA)

```bash
# Jupyter Notebook 실행
jupyter notebook
```

**notebooks/01_eda.ipynb** 생성:

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# 데이터 로드 (샘플링)
df = pd.read_csv('../data/raw/2019-Oct.csv', nrows=1000000)

# 기본 정보
print(df.head())
print(df.info())
print(df.describe())

# 컬럼 확인
print(df.columns)
# ['event_time', 'event_type', 'product_id', 'category_id', 
#  'category_code', 'brand', 'price', 'user_id', 'user_session']

# 결측치 확인
print(df.isnull().sum())

# event_type 분포
print(df['event_type'].value_counts())
# view        ~89%
# cart        ~8%
# purchase    ~3%

# 시간대별 분포
df['event_time'] = pd.to_datetime(df['event_time'])
df['hour'] = df['event_time'].dt.hour
df['hour'].hist(bins=24)
plt.title('Events by Hour')
plt.xlabel('Hour')
plt.ylabel('Count')
plt.show()

# 유저당 이벤트 수
user_events = df.groupby('user_id').size()
print(f"평균 이벤트/유저: {user_events.mean():.2f}")
print(f"중앙값: {user_events.median():.2f}")

# 상품 인기도
top_products = df['product_id'].value_counts().head(20)
print("Top 20 제품:")
print(top_products)
```

---

### Step 2: 데이터베이스 스키마 설계

```bash
# SQL 스크립트 생성
vi scripts/create_tables.sql
```

```sql
-- 유저 행동 이벤트 테이블
CREATE TABLE IF NOT EXISTS user_events (
    event_id SERIAL PRIMARY KEY,
    event_time TIMESTAMP NOT NULL,
    event_type VARCHAR(20) NOT NULL,
    product_id BIGINT NOT NULL,
    category_id BIGINT,
    category_code VARCHAR(200),
    brand VARCHAR(100),
    price DECIMAL(10, 2),
    user_id BIGINT NOT NULL,
    user_session UUID NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 인덱스 생성
CREATE INDEX idx_user_events_user_id ON user_events(user_id);
CREATE INDEX idx_user_events_product_id ON user_events(product_id);
CREATE INDEX idx_user_events_event_time ON user_events(event_time);
CREATE INDEX idx_user_events_event_type ON user_events(event_type);

-- 상품 메타데이터 테이블
CREATE TABLE IF NOT EXISTS products (
    product_id BIGINT PRIMARY KEY,
    category_id BIGINT,
    category_code VARCHAR(200),
    brand VARCHAR(100),
    avg_price DECIMAL(10, 2),
    view_count INT DEFAULT 0,
    cart_count INT DEFAULT 0,
    purchase_count INT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 유저 통계 테이블
CREATE TABLE IF NOT EXISTS user_stats (
    user_id BIGINT PRIMARY KEY,
    total_views INT DEFAULT 0,
    total_carts INT DEFAULT 0,
    total_purchases INT DEFAULT 0,
    total_spent DECIMAL(12, 2) DEFAULT 0,
    first_event TIMESTAMP,
    last_event TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```bash
# PostgreSQL 접속하여 테이블 생성
docker exec -it ecommerce-recommendation-postgres-1 psql -U airflow -d airflow

# SQL 실행
\i /opt/airflow/scripts/create_tables.sql

# 테이블 확인
\dt

# 종료
\q
```

---

### Step 3: 데이터 수집 유틸리티 작성

```bash
# scripts/data_loader.py 생성
vi scripts/data_loader.py
```

```python
import pandas as pd
import psycopg2
from psycopg2.extras import execute_batch
from datetime import datetime
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DataLoader:
    def __init__(self, db_config):
        self.db_config = db_config
        self.conn = None
        
    def connect(self):
        """데이터베이스 연결"""
        try:
            self.conn = psycopg2.connect(**self.db_config)
            logger.info("Database connected successfully")
        except Exception as e:
            logger.error(f"Database connection failed: {e}")
            raise
    
    def close(self):
        """연결 종료"""
        if self.conn:
            self.conn.close()
            logger.info("Database connection closed")
    
    def load_csv_to_db(self, csv_path, table_name, batch_size=10000):
        """CSV 파일을 데이터베이스에 로드"""
        try:
            # CSV 읽기 (청크 단위)
            chunk_iter = pd.read_csv(csv_path, chunksize=batch_size)
            
            cursor = self.conn.cursor()
            total_rows = 0
            
            for i, chunk in enumerate(chunk_iter):
                # 데이터 전처리
                chunk = self._preprocess_chunk(chunk)
                
                # 배치 인서트
                insert_query = self._get_insert_query(table_name)
                records = chunk.to_records(index=False).tolist()
                
                execute_batch(cursor, insert_query, records, page_size=batch_size)
                self.conn.commit()
                
                total_rows += len(chunk)
                logger.info(f"Batch {i+1}: {total_rows} rows loaded")
                
                # 메모리 절약을 위한 주기적 커밋
                if (i + 1) % 10 == 0:
                    self.conn.commit()
            
            cursor.close()
            logger.info(f"Total {total_rows} rows loaded to {table_name}")
            
        except Exception as e:
            self.conn.rollback()
            logger.error(f"Error loading data: {e}")
            raise
    
    def _preprocess_chunk(self, df):
        """데이터 전처리"""
        # 컬럼명 변경
        df = df.rename(columns={
            'event_time': 'event_time',
            'event_type': 'event_type',
            'product_id': 'product_id',
            'category_id': 'category_id',
            'category_code': 'category_code',
            'brand': 'brand',
            'price': 'price',
            'user_id': 'user_id',
            'user_session': 'user_session'
        })
        
        # 결측치 처리
        df['category_id'] = df['category_id'].fillna(0)
        df['category_code'] = df['category_code'].fillna('unknown')
        df['brand'] = df['brand'].fillna('unknown')
        
        # 날짜 형식 변환
        df['event_time'] = pd.to_datetime(df['event_time'])
        
        return df
    
    def _get_insert_query(self, table_name):
        """인서트 쿼리 생성"""
        if table_name == 'user_events':
            return """
                INSERT INTO user_events 
                (event_time, event_type, product_id, category_id, 
                 category_code, brand, price, user_id, user_session)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON CONFLICT DO NOTHING
            """
        else:
            raise ValueError(f"Unknown table: {table_name}")

# 사용 예시
if __name__ == "__main__":
    db_config = {
        'host': 'localhost',
        'port': 5432,
        'database': 'airflow',
        'user': 'airflow',
        'password': 'airflow'
    }
    
    loader = DataLoader(db_config)
    loader.connect()
    
    # 샘플 데이터 로드 (전체는 너무 크므로 10월 첫 날만)
    loader.load_csv_to_db(
        csv_path='../data/raw/2019-Oct.csv',
        table_name='user_events',
        batch_size=10000
    )
    
    loader.close()
```

---

### Step 4: Airflow DAG 작성 - 데이터 수집

```bash
# airflow/dags/01_data_collection_dag.py 생성
vi airflow/dags/01_data_collection_dag.py
```

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from datetime import datetime, timedelta
import pandas as pd
import logging

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2025, 1, 1),
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'data_collection_pipeline',
    default_args=default_args,
    description='수집 및 적재 파이프라인',
    schedule_interval='@daily',  # 매일 실행
    catchup=False,
    tags=['data-collection', 'ecommerce'],
)

def extract_daily_events(**context):
    """일일 이벤트 데이터 추출"""
    execution_date = context['ds']  # YYYY-MM-DD
    logging.info(f"Extracting events for {execution_date}")
    
    # 실제로는 API나 로그 시스템에서 가져옴
    # 여기서는 CSV 샘플링
    df = pd.read_csv(
        '/opt/airflow/data/raw/2019-Oct.csv',
        nrows=100000  # 샘플
    )
    
    # 날짜 필터링 (실제 환경에서)
    df['event_time'] = pd.to_datetime(df['event_time'])
    df = df[df['event_time'].dt.date == pd.to_datetime(execution_date).date()]
    
    # 임시 저장
    output_path = f'/opt/airflow/data/raw/events_{execution_date}.csv'
    df.to_csv(output_path, index=False)
    
    logging.info(f"Extracted {len(df)} events to {output_path}")
    return output_path

def validate_data(**context):
    """데이터 품질 검증"""
    task_instance = context['task_instance']
    file_path = task_instance.xcom_pull(task_ids='extract_events')
    
    df = pd.read_csv(file_path)
    
    # 검증 규칙
    checks = {
        'row_count': len(df) > 0,
        'required_columns': all(col in df.columns for col in 
            ['event_time', 'event_type', 'product_id', 'user_id']),
        'no_null_user_id': df['user_id'].notnull().all(),
        'valid_price': (df['price'] >= 0).all(),
    }
    
    if not all(checks.values()):
        failed_checks = [k for k, v in checks.items() if not v]
        raise ValueError(f"Data validation failed: {failed_checks}")
    
    logging.info("Data validation passed")
    return True

def load_to_postgres(**context):
    """PostgreSQL에 적재"""
    task_instance = context['task_instance']
    file_path = task_instance.xcom_pull(task_ids='extract_events')
    
    df = pd.read_csv(file_path)
    
    # PostgresHook 사용
    pg_hook = PostgresHook(postgres_conn_id='postgres_default')
    engine = pg_hook.get_sqlalchemy_engine()
    
    # 배치 인서트
    df.to_sql(
        'user_events',
        engine,
        if_exists='append',
        index=False,
        method='multi',
        chunksize=1000
    )
    
    logging.info(f"Loaded {len(df)} rows to user_events table")

def update_product_stats(**context):
    """상품 통계 업데이트"""
    execution_date = context['ds']
    
    pg_hook = PostgresHook(postgres_conn_id='postgres_default')
    conn = pg_hook.get_conn()
    cursor = conn.cursor()
    
    # 상품별 집계
    query = f"""
        INSERT INTO products (product_id, category_id, category_code, brand, 
                             view_count, cart_count, purchase_count, avg_price)
        SELECT 
            product_id,
            MAX(category_id) as category_id,
            MAX(category_code) as category_code,
            MAX(brand) as brand,
            SUM(CASE WHEN event_type = 'view' THEN 1 ELSE 0 END) as view_count,
            SUM(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) as cart_count,
            SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) as purchase_count,
            AVG(price) as avg_price
        FROM user_events
        WHERE DATE(event_time) = '{execution_date}'
        GROUP BY product_id
        ON CONFLICT (product_id) 
        DO UPDATE SET
            view_count = products.view_count + EXCLUDED.view_count,
            cart_count = products.cart_count + EXCLUDED.cart_count,
            purchase_count = products.purchase_count + EXCLUDED.purchase_count,
            avg_price = (products.avg_price + EXCLUDED.avg_price) / 2,
            updated_at = CURRENT_TIMESTAMP;
    """
    
    cursor.execute(query)
    conn.commit()
    cursor.close()
    conn.close()
    
    logging.info("Product stats updated")

# Task 정의
extract_task = PythonOperator(
    task_id='extract_events',
    python_callable=extract_daily_events,
    dag=dag,
)

validate_task = PythonOperator(
    task_id='validate_data',
    python_callable=validate_data,
    dag=dag,
)

load_task = PythonOperator(
    task_id='load_to_postgres',
    python_callable=load_to_postgres,
    dag=dag,
)

stats_task = PythonOperator(
    task_id='update_product_stats',
    python_callable=update_product_stats,
    dag=dag,
)

# Task 의존성
extract_task >> validate_task >> load_task >> stats_task
```

---

### Step 5: Airflow Connection 설정

```bash
# Airflow UI에서 설정
# Admin > Connections > + 버튼 클릭

# Connection Id: postgres_default
# Connection Type: Postgres
# Host: postgres
# Schema: airflow
# Login: airflow
# Password: airflow
# Port: 5432
```

또는 CLI로:

```bash
docker exec -it ecommerce-recommendation-airflow-webserver-1 bash

airflow connections add 'postgres_default' \
    --conn-type 'postgres' \
    --conn-host 'postgres' \
    --conn-schema 'airflow' \
    --conn-login 'airflow' \
    --conn-password 'airflow' \
    --conn-port 5432
```

---

### Step 6: DAG 테스트

```bash
# DAG 구문 검사
docker exec -it ecommerce-recommendation-airflow-webserver-1 bash
airflow dags list | grep data_collection

# DAG 테스트 실행
airflow dags test data_collection_pipeline 2025-01-01

# 특정 Task만 테스트
airflow tasks test data_collection_pipeline extract_events 2025-01-01
```

---

## 📅 Day 5-7: 전처리 파이프라인

### Step 1: 전처리 함수 작성

```bash
vi scripts/preprocessing.py
```

```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import LabelEncoder
from datetime import datetime, timedelta
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DataPreprocessor:
    def __init__(self):
        self.label_encoders = {}
    
    def remove_duplicates(self, df):
        """중복 제거"""
        before = len(df)
        df = df.drop_duplicates(
            subset=['user_id', 'product_id', 'event_time', 'event_type']
        )
        after = len(df)
        logger.info(f"Removed {before - after} duplicates")
        return df
    
    def handle_missing_values(self, df):
        """결측치 처리"""
        # 카테고리 결측치
        df['category_code'] = df['category_code'].fillna('unknown')
        df['brand'] = df['brand'].fillna('unknown')
        df['category_id'] = df['category_id'].fillna(0)
        
        # 가격 결측치 (카테고리 평균으로 대체)
        df['price'] = df.groupby('category_code')['price'].transform(
            lambda x: x.fillna(x.median())
        )
        
        logger.info("Missing values handled")
        return df
    
    def filter_active_users(self, df, min_events=5):
        """활성 유저 필터링"""
        user_event_counts = df.groupby('user_id').size()
        active_users = user_event_counts[user_event_counts >= min_events].index
        
        before = len(df)
        df = df[df['user_id'].isin(active_users)]
        after = len(df)
        
        logger.info(f"Filtered to {len(active_users)} active users")
        logger.info(f"Removed {before - after} rows from inactive users")
        return df
    
    def filter_popular_products(self, df, min_interactions=10):
        """인기 상품 필터링"""
        product_counts = df.groupby('product_id').size()
        popular_products = product_counts[product_counts >= min_interactions].index
        
        before = len(df)
        df = df[df['product_id'].isin(popular_products)]
        after = len(df)
        
        logger.info(f"Filtered to {len(popular_products)} popular products")
        logger.info(f"Removed {before - after} rows from unpopular products")
        return df
    
    def encode_categorical_features(self, df):
        """카테고리 변수 인코딩"""
        categorical_cols = ['event_type', 'category_code', 'brand']
        
        for col in categorical_cols:
            if col not in self.label_encoders:
                self.label_encoders[col] = LabelEncoder()
                df[f'{col}_encoded'] = self.label_encoders[col].fit_transform(df[col])
            else:
                # 새로운 값 처리
                df[f'{col}_encoded'] = df[col].apply(
                    lambda x: self.label_encoders[col].transform([x])[0] 
                    if x in self.label_encoders[col].classes_ else -1
                )
        
        logger.info("Categorical features encoded")
        return df
    
    def create_temporal_features(self, df):
        """시간 기반 피처 생성"""
        df['event_time'] = pd.to_datetime(df['event_time'])
        
        df['hour'] = df['event_time'].dt.hour
        df['day_of_week'] = df['event_time'].dt.dayofweek
        df['is_weekend'] = (df['day_of_week'] >= 5).astype(int)
        df['month'] = df['event_time'].dt.month
        
        # 시간대 구분
        df['time_of_day'] = pd.cut(
            df['hour'],
            bins=[0, 6, 12, 18, 24],
            labels=['night', 'morning', 'afternoon', 'evening'],
            include_lowest=True
        )
        
        logger.info("Temporal features created")
        return df
    
    def create_user_features(self, df):
        """유저 피처 생성"""
        user_features = df.groupby('user_id').agg({
            'event_time': ['min', 'max', 'count'],
            'product_id': 'nunique',
            'price': ['mean', 'sum'],
            'event_type': lambda x: (x == 'purchase').sum()
        }).reset_index()
        
        user_features.columns = [
            'user_id', 'first_event', 'last_event', 'total_events',
            'unique_products', 'avg_price', 'total_spent', 'purchase_count'
        ]
        
        # 활동 기간
        user_features['active_days'] = (
            user_features['last_event'] - user_features['first_event']
        ).dt.days + 1
        
        # 일평균 이벤트
        user_features['events_per_day'] = (
            user_features['total_events'] / user_features['active_days']
        )
        
        # 구매 전환율
        user_features['purchase_rate'] = (
            user_features['purchase_count'] / user_features['total_events']
        )
        
        logger.info(f"Created features for {len(user_features)} users")
        return user_features
    
    def create_product_features(self, df):
        """상품 피처 생성"""
        product_features = df.groupby('product_id').agg({
            'event_type': [
                lambda x: (x == 'view').sum(),
                lambda x: (x == 'cart').sum(),
                lambda x: (x == 'purchase').sum()
            ],
            'user_id': 'nunique',
            'price': 'mean',
            'category_code': 'first',
            'brand': 'first'
        }).reset_index()
        
        product_features.columns = [
            'product_id', 'view_count', 'cart_count', 'purchase_count',
            'unique_users', 'avg_price', 'category_code', 'brand'
        ]
        
        # 전환율
        product_features['cart_to_view_rate'] = (
            product_features['cart_count'] / 
            (product_features['view_count'] + 1)
        )
        product_features['purchase_to_cart_rate'] = (
            product_features['purchase_count'] / 
            (product_features['cart_count'] + 1)
        )
        
        # 인기도 점수
        product_features['popularity_score'] = (
            product_features['view_count'] * 0.1 +
            product_features['cart_count'] * 0.3 +
            product_features['purchase_count'] * 0.6
        )
        
        logger.info(f"Created features for {len(product_features)} products")
        return product_features
    
    def create_interaction_matrix(self, df, event_type='purchase'):
        """유저-아이템 상호작용 매트릭스 생성"""
        # 특정 이벤트만 필터링
        interactions = df[df['event_type'] == event_type].copy()
        
        # 가중치 부여 (최근 이벤트일수록 높은 가중치)
        max_time = interactions['event_time'].max()
        interactions['days_ago'] = (
            max_time - interactions['event_time']
        ).dt.days
        interactions['weight'] = np.exp(-interactions['days_ago'] / 30)
        
        # 유저-아이템 쌍의 총 가중치
        matrix = interactions.groupby(
            ['user_id', 'product_id']
        )['weight'].sum().reset_index()
        
        matrix.columns = ['user_id', 'product_id', 'rating']
        
        logger.info(f"Created interaction matrix: {len(matrix)} interactions")
        return matrix
    
    def preprocess_pipeline(self, df):
        """전체 전처리 파이프라인"""
        logger.info("Starting preprocessing pipeline")
        
        df = self.remove_duplicates(df)
        df = self.handle_missing_values(df)
        df = self.filter_active_users(df, min_events=5)
        df = self.filter_popular_products(df, min_interactions=10)
        df = self.encode_categorical_features(df)
        df = self.create_temporal_features(df)
        
        logger.info("Preprocessing pipeline completed")
        return df

# 사용 예시
if __name__ == "__main__":
    preprocessor = DataPreprocessor()
    
    # 데이터 로드
    df = pd.read_csv('../data/raw/2019-Oct.csv', nrows=100000)
    
    # 전처리
    df_processed = preprocessor.preprocess_pipeline(df)
    
    # 피처 생성
    user_features = preprocessor.create_user_features(df_processed)
    product_features = preprocessor.create_product_features(df_processed)
    interaction_matrix = preprocessor.create_interaction_matrix(df_processed)
    
    # 저장
    df_processed.to_csv('../data/processed/events_processed.csv', index=False)
    user_features.to_csv('../data/features/user_features.csv', index=False)
    product_features.to_csv('../data/features/product_features.csv', index=False)
    interaction_matrix.to_csv('../data/features/interaction_matrix.csv', index=False)
    
    print("Preprocessing completed!")
```

---

### Step 2: 전처리 DAG 작성

```bash
vi airflow/dags/02_preprocessing_dag.py
```

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from datetime import datetime, timedelta
import pandas as pd
import sys
sys.path.append('/opt/airflow/scripts')
from preprocessing import DataPreprocessor
import logging

default_args = {
    'owner': 'data-team',
    'depends_on_past': True,
    'start_date': datetime(2025, 1, 1),
    'email_on_failure': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'data_preprocessing_pipeline',
    default_args=default_args,
    description='데이터 전처리 및 피처 엔지니어링',
    schedule_interval='@daily',
    catchup=False,
    tags=['preprocessing', 'feature-engineering'],
)

def extract_from_db(**context):
    """데이터베이스에서 원본 데이터 추출"""
    execution_date = context['ds']
    
    pg_hook = PostgresHook(postgres_conn_id='postgres_default')
    
    # 최근 30일 데이터 추출
    query = f"""
        SELECT *
        FROM user_events
        WHERE event_time >= '{execution_date}'::date - INTERVAL '30 days'
          AND event_time < '{execution_date}'::date + INTERVAL '1 day'
    """
    
    df = pg_hook.get_pandas_df(query)
    logging.info(f"Extracted {len(df)} rows from database")
    
    # 임시 저장
    output_path = f'/opt/airflow/data/raw/raw_{execution_date}.csv'
    df.to_csv(output_path, index=False)
    
    return output_path

def preprocess_data(**context):
    """데이터 전처리"""
    task_instance = context['task_instance']
    input_path = task_instance.xcom_pull(task_ids='extract_from_db')
    execution_date = context['ds']
    
    # 데이터 로드
    df = pd.read_csv(input_path)
    logging.info(f"Loaded {len(df)} rows for preprocessing")
    
    # 전처리 실행
    preprocessor = DataPreprocessor()
    df_processed = preprocessor.preprocess_pipeline(df)
    
    # 저장
    output_path = f'/opt/airflow/data/processed/processed_{execution_date}.csv'
    df_processed.to_csv(output_path, index=False)
    
    logging.info(f"Preprocessing completed: {len(df_processed)} rows")
    return output_path

def create_user_features(**context):
    """유저 피처 생성"""
    task_instance = context['task_instance']
    input_path = task_instance.xcom_pull(task_ids='preprocess_data')
    execution_date = context['ds']
    
    df = pd.read_csv(input_path)
    
    preprocessor = DataPreprocessor()
    user_features = preprocessor.create_user_features(df)
    
    # PostgreSQL에 저장
    pg_hook = PostgresHook(postgres_conn_id='postgres_default')
    engine = pg_hook.get_sqlalchemy_engine()
    
    user_features.to_sql(
        'user_features',
        engine,
        if_exists='replace',
        index=False
    )
    
    logging.info(f"Created features for {len(user_features)} users")

def create_product_features(**context):
    """상품 피처 생성"""
    task_instance = context['task_instance']
    input_path = task_instance.xcom_pull(task_ids='preprocess_data')
    execution_date = context['ds']
    
    df = pd.read_csv(input_path)
    
    preprocessor = DataPreprocessor()
    product_features = preprocessor.create_product_features(df)
    
    # PostgreSQL에 저장
    pg_hook = PostgresHook(postgres_conn_id='postgres_default')
    engine = pg_hook.get_sqlalchemy_engine()
    
    product_features.to_sql(
        'product_features',
        engine,
        if_exists='replace',
        index=False
    )
    
    logging.info(f"Created features for {len(product_features)} products")

def create_interaction_matrix(**context):
    """상호작용 매트릭스 생성"""
    task_instance = context['task_instance']
    input_path = task_instance.xcom_pull(task_ids='preprocess_data')
    execution_date = context['ds']
    
    df = pd.read_csv(input_path)
    
    preprocessor = DataPreprocessor()
    
    # 구매 기반 매트릭스
    purchase_matrix = preprocessor.create_interaction_matrix(df, 'purchase')
    purchase_matrix.to_csv(
        f'/opt/airflow/data/features/purchase_matrix_{execution_date}.csv',
        index=False
    )
    
    # 장바구니 기반 매트릭스
    cart_matrix = preprocessor.create_interaction_matrix(df, 'cart')
    cart_matrix.to_csv(
        f'/opt/airflow/data/features/cart_matrix_{execution_date}.csv',
        index=False
    )
    
    logging.info("Interaction matrices created")

# Task 정의
extract_task = PythonOperator(
    task_id='extract_from_db',
    python_callable=extract_from_db,
    dag=dag,
)

preprocess_task = PythonOperator(
    task_id='preprocess_data',
    python_callable=preprocess_data,
    dag=dag,
)

user_features_task = PythonOperator(
    task_id='create_user_features',
    python_callable=create_user_features,
    dag=dag,
)

product_features_task = PythonOperator(
    task_id='create_product_features',
    python_callable=create_product_features,
    dag=dag,
)

interaction_matrix_task = PythonOperator(
    task_id='create_interaction_matrix',
    python_callable=create_interaction_matrix,
    dag=dag,
)

# Task 의존성
extract_task >> preprocess_task >> [
    user_features_task,
    product_features_task,
    interaction_matrix_task
]
```

---

## ✅ Week 1 완료 체크리스트

```bash
# 1. Docker 서비스 확인
docker-compose ps

# 2. Airflow UI 접속 확인
# http://localhost:8080

# 3. PostgreSQL 테이블 확인
docker exec -it ecommerce-recommendation-postgres-1 psql -U airflow -d airflow -c "\dt"

# 4. DAG 목록 확인
docker exec -it ecommerce-recommendation-airflow-webserver-1 airflow dags list

# 5. 샘플 데이터 확인
docker exec -it ecommerce-recommendation-postgres-1 psql -U airflow -d airflow -c "SELECT COUNT(*) FROM user_events;"

# 6. 전처리된 파일 확인
ls -lh data/processed/
ls -lh data/features/
```

---

## 🎯 Week 1 최종 점검

### 달성 목표:
- ✅ Airflow + PostgreSQL 환경 구축 완료
- ✅ 데이터 수집 DAG 작성 및 테스트
- ✅ 전처리 파이프라인 구현
- ✅ 피처 엔지니어링 완료
- ✅ 유저-아이템 상호작용 매트릭스 생성

### 다음 주 준비사항:
- Week 2에서는 이 데이터를 기반으로 추천 모델 학습
- 협업 필터링 & 컨텐츠 기반 필터링 구현
- 모델 성능 평가 및 최적화

---

**Week 1 완료! 수고하셨습니다! 🎉**

궁금한 점이나 막히는 부분 있으면 언제든 물어보세요!