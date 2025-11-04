# 이커머스 추천 시스템 - 세부 프로젝트 계획서

## 📋 프로젝트 개요

**목표**: 실시간 유저 행동 기반 상품 추천 시스템 구축
**기간**: 3-4주
**핵심 기술**: Airflow, Python, ML(협업필터링), A/B 테스트

---

## 🗓️ 주차별 상세 일정

### **Week 1: 환경 구축 & 데이터 파이프라인**

#### Day 1-2: 프로젝트 셋업
```bash
# 프로젝트 구조 생성
mkdir -p ecommerce-recommendation/{airflow/{dags,plugins,logs},data/{raw,processed},models,notebooks,tests}
cd ecommerce-recommendation

# Docker Compose로 Airflow 구축
vi docker-compose.yml
```

**작업 내용**:
- Airflow + PostgreSQL + Redis 환경 구성
- Git 저장소 초기화
- 가상 데이터셋 준비 (Kaggle: ecommerce behavior data)

#### Day 3-4: 데이터 수집 DAG 구축
```python
# dags/01_data_collection_dag.py
from airflow import DAG
from airflow.operators.python import PythonOperator

# 유저 행동 로그 수집
- 페이지 조회 (view)
- 장바구니 추가 (cart)
- 구매 (purchase)
- 평점/리뷰 (rating)
```

**구현 기능**:
- 시간별 배치 수집 (매 1시간)
- 증분 데이터 적재
- 데이터 품질 체크

#### Day 5-7: 전처리 파이프라인
```python
# dags/02_preprocessing_dag.py
# 작업 순서
1. 중복 제거
2. 결측치 처리
3. 유저-아이템 매트릭스 생성
4. Feature Engineering
   - 유저: 구매 빈도, 카테고리 선호도, 평균 구매 금액
   - 아이템: 인기도, 평균 평점, 카테고리
```

---

### **Week 2: 추천 모델 개발**

#### Day 8-10: 협업 필터링 구현
```python
# models/collaborative_filtering.py
from surprise import SVD, KNNBasic
from surprise.model_selection import cross_validate

# 1. User-based CF
# 2. Item-based CF  
# 3. Matrix Factorization (SVD)
```

**모델 비교 지표**:
- RMSE
- Precision@K
- Recall@K
- NDCG

#### Day 11-12: 컨텐츠 기반 필터링
```python
# models/content_based.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

# 상품 메타데이터 활용
- 카테고리, 브랜드, 가격대
- 상품 설명 텍스트 (TF-IDF)
```

#### Day 13-14: 하이브리드 모델 & 최적화
```python
# models/hybrid_recommender.py
# CF + 컨텐츠 기반 앙상블
weighted_score = alpha * cf_score + (1-alpha) * content_score

# Cold Start 문제 해결
- 신규 유저: 인기 상품 추천
- 신규 상품: 컨텐츠 기반 유사 상품 추천
```

---

### **Week 3: MLOps & 자동화**

#### Day 15-17: 모델 재학습 파이프라인
```python
# dags/03_model_training_dag.py
@task
def train_model():
    # 최근 30일 데이터로 학습
    # 성능 검증 (holdout set)
    # 성능 개선 시에만 모델 교체
    
@task  
def evaluate_model():
    # A/B 테스트 그룹용 예측 생성
    
@task
def save_model():
    # MLflow에 모델 버전 관리
```

**Airflow 스케줄**:
- 매주 일요일 03:00 전체 재학습
- 매일 자정 증분 업데이트

#### Day 18-19: 실시간 추천 API
```python
# api/recommendation_service.py
from fastapi import FastAPI
import redis

app = FastAPI()

@app.get("/recommend/{user_id}")
def get_recommendations(user_id: int, top_k: int = 10):
    # Redis 캐시 확인
    # 모델 예측 수행
    # 비즈니스 룰 적용 (재고, 할인)
    return {"items": [...]}
```

#### Day 20-21: 모니터링 & 로깅
```bash
# Prometheus + Grafana 대시보드
- 추천 요청 수 (TPS)
- 응답 시간 (latency)
- 추천 다양성 (diversity)
- 클릭률 (CTR)
```

---

### **Week 4: A/B 테스트 & 최종 정리**

#### Day 22-24: A/B 테스트 시뮬레이션
```python
# simulation/ab_test.py
import numpy as np
from scipy import stats

# 그룹 분할
- Control: 랜덤 추천
- Treatment A: 협업 필터링
- Treatment B: 하이브리드 모델

# 평가 지표
- 클릭률 (CTR)
- 전환율 (CVR)  
- 평균 주문 금액 (AOV)
- 매출 (Revenue)

# 통계적 유의성 검증
p_value = stats.ttest_ind(control, treatment).pvalue
```

#### Day 25-26: 성능 최적화
- 추천 결과 Redis 캐싱 (TTL: 1시간)
- 배치 예측으로 응답 속도 개선
- 인덱싱 최적화 (Faiss, Annoy)

#### Day 27-28: 문서화 & 발표 자료
```markdown
# README.md
1. 프로젝트 아키텍처 다이어그램
2. 설치 및 실행 방법
3. API 문서 (Swagger)
4. 성능 벤치마크 결과
5. A/B 테스트 결과 분석
```

---

## 🏗️ 시스템 아키텍처

```
[유저 행동 로그] 
    ↓
[Airflow DAG - 수집]
    ↓
[PostgreSQL - Raw Data]
    ↓
[Airflow DAG - 전처리]
    ↓
[Feature Store]
    ↓
[Airflow DAG - 학습] → [MLflow Model Registry]
    ↓
[FastAPI 추천 서비스] ← [Redis 캐시]
    ↓
[A/B 테스트 프레임워크]
    ↓
[Grafana 모니터링]
```

---

## 📊 데이터셋 구조

### 유저 행동 로그
```sql
CREATE TABLE user_events (
    event_id SERIAL PRIMARY KEY,
    user_id INT,
    item_id INT,
    event_type VARCHAR(20), -- view, cart, purchase, rating
    timestamp TIMESTAMP,
    session_id VARCHAR(50),
    rating FLOAT,
    price DECIMAL(10,2)
);
```

### 상품 메타데이터
```sql
CREATE TABLE items (
    item_id INT PRIMARY KEY,
    name VARCHAR(200),
    category VARCHAR(50),
    brand VARCHAR(100),
    price DECIMAL(10,2),
    description TEXT,
    image_url VARCHAR(500)
);
```

---

## 🔧 핵심 Airflow DAG 예시

```python
# dags/recommendation_pipeline.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.sensors.external_task import ExternalTaskSensor
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'recommendation_pipeline',
    default_args=default_args,
    schedule_interval='0 3 * * 0',  # 매주 일요일 03:00
    start_date=datetime(2025, 1, 1),
    catchup=False,
) as dag:
    
    # Task 1: 데이터 추출
    extract = PythonOperator(
        task_id='extract_user_behavior',
        python_callable=extract_data,
    )
    
    # Task 2: 전처리
    transform = PythonOperator(
        task_id='preprocess_data',
        python_callable=preprocess,
    )
    
    # Task 3: 모델 학습
    train = PythonOperator(
        task_id='train_model',
        python_callable=train_recommender,
    )
    
    # Task 4: 평가
    evaluate = PythonOperator(
        task_id='evaluate_model',
        python_callable=evaluate,
    )
    
    # Task 5: 배포 (조건부)
    deploy = PythonOperator(
        task_id='deploy_model',
        python_callable=deploy_to_production,
        trigger_rule='all_success',
    )
    
    extract >> transform >> train >> evaluate >> deploy
```

---

## 📈 A/B 테스트 설계

### 실험 설계
```python
# 트래픽 분할
- Control (30%): 랜덤 추천
- Variant A (35%): 협업 필터링
- Variant B (35%): 하이브리드

# 실험 기간: 2주
# 최소 샘플 수: 그룹당 1,000명

# 성공 지표 (Success Metrics)
1. Primary: 클릭률 (CTR) 5% 향상 목표
2. Secondary: 
   - 전환율 (CVR)
   - 평균 체류 시간
   - 재방문율
```

### 통계 검정
```python
from scipy.stats import chi2_contingency

# 클릭률 비교 (카이제곱 검정)
contingency_table = [
    [control_clicks, control_impressions - control_clicks],
    [variant_clicks, variant_impressions - variant_clicks]
]
chi2, p_value, dof, expected = chi2_contingency(contingency_table)

# p < 0.05: 통계적으로 유의미한 차이
```

---

## 🎯 핵심 성과 지표 (KPI)

1. **모델 성능**
   - RMSE < 1.0
   - Precision@10 > 15%
   - NDCG@10 > 0.4

2. **시스템 성능**
   - API 응답 시간 < 100ms (P95)
   - 일일 처리량 > 100만 건
   - Airflow DAG 성공률 > 99%

3. **비즈니스 임팩트**
   - CTR 10% 향상
   - CVR 5% 향상
   - 평균 주문 금액 15% 증가

---

## 💡 추가 개선 아이디어

### Phase 2 (확장 가능)
1. **실시간 스트리밍**
   - Kafka + Flink로 실시간 행동 반영
   
2. **딥러닝 모델**
   - Neural Collaborative Filtering
   - Transformer 기반 Sequential Recommendation

3. **다중 목적 최적화**
   - 매출 + 다양성 + 신선도 균형

4. **개인화 강화**
   - 컨텍스트 인식 (시간대, 요일, 디바이스)
   - 세션 기반 추천

---

## 📚 필요 라이브러리

```bash
# requirements.txt
apache-airflow==2.8.0
apache-airflow-providers-postgres
scikit-surprise==1.1.3
scikit-learn==1.3.2
pandas==2.1.4
numpy==1.26.2
fastapi==0.109.0
uvicorn==0.25.0
redis==5.0.1
mlflow==2.9.2
psycopg2-binary==2.9.9
scipy==1.11.4
```

---

## ✅ 체크리스트

- [ ] Week 1: Airflow 환경 구축 완료
- [ ] Week 1: 데이터 수집/전처리 DAG 완성
- [ ] Week 2: 협업 필터링 모델 학습
- [ ] Week 2: 하이브리드 모델 구현
- [ ] Week 3: 자동 재학습 파이프라인 구축
- [ ] Week 3: FastAPI 서비스 배포
- [ ] Week 4: A/B 테스트 시뮬레이션
- [ ] Week 4: 최종 문서 및 발표 자료

---

**질문 있으면 언제든 물어보세요!** 🚀