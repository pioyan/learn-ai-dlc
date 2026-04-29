# モブワーク実践ガイド 2: Mob Construction実践例

## シナリオ
前セッションで確定した Unit-C（レコメンドエンジンコア）を、
Domain Design → Logical Design → Code & Test の流れで実装する 2 週間のプロセス

## 所要時間（週ごと）
- Week 1: Domain Design & Logical Design（Mob-driven, 4 日間、各日 6-8 時間）
- Week 2: Code & Test & Integration（Mob + 個別作業の混在、4 日間）

## 参加者
- **テックリード / アーキテクト**（Design 主導）
- **AI コーディング・アシスタント**（Claude, Copilot, Cursor等）
- **シニア開発者 2 名**（実装・レビュー）
- **ジュニア開発者 1 名**（実装サポート）
- **QA / テスター**（テスト設計）
- **ドメイン専門家**（マーケティング PO の代理等）

---

## Week 1: Domain Design & Logical Design

### Day 1: Domain Analysis

#### 目的
ドメイン言語、主要なドメイン概念、ビジネスルールを可視化する

#### 1-1. セッション開始（プリワーク共有）

テックリードが既存資料を共有：

```markdown
## Unit-C: 推薦エンジンの要件（Elaborationから持ち越し）

### ビジネス要件
- 購買履歴 → パーソナライズ推薦 3-5 件
- コスメと食品の相乗効果を優先
- API 応答時間 < 200ms
- 初期精度: リコール 60% 以上

### 技術要件
- AWS Lambda or ECS（スケーラビリティ）
- DynamoDB またはベクトルDB（推薦ベース）
- CloudWatch 監視

### Data Flow（現時点での仮説）
顧客ID → 購買履歴取得 → ベクトル化
    ↓
相乗効果マトリクス
    ↓
TOP-K 推薦候補抽出
    ↓ JSON
推薦結果API返却
```

#### 1-2. Domain Modeling Session

AIに対して以下プロンプトを投げる:

```
あなたは Domain-Driven Design の専門家です。

【Context】
e-Commerce プラットフォームの推薦エンジン

【ビジネス要件】
- 顧客の購買履歴から、相乗効果の高い商品を推薦
- コスメ + 食品の組み合わせを優先
- リアルタイムAPI（<200ms）
- マーケティング担当者がマトリクス設定可能

【質問】
このドメインにおいて、以下を定義してください:
1. Core Entities: どのエンティティが主役か
2. Value Objects: どの概念が値オブジェクトか
3. Aggregates: エンティティ群をどうグループ化するか
4. Domain Events: どの変更イベントが起こるか
5. Repositories: どのデータを永続化するか
6. Ubiquitous Language: ドメイン用語を一覧

各概念に対して、Python/TypeScript の
クラス/型定義スケルトンも提示してください。
```

#### 1-3. AI による Domain Model 提案

```python
# Domain Model Proposal

## Entities

class Customer:
    """購買履歴を持つ顧客"""
    customer_id: str
    purchase_history: List[Purchase]
    preferences: CustomerPreference
    
class Product:
    """推薦対象の商品"""
    product_id: str
    category: ProductCategory  # enum: Cosmetics, Food, ...
    metadata: ProductMetadata
    synergy_score: float  # 相乗効果スコア（相手カテゴリとの相性）

class Purchase:
    """購買記録"""
    purchase_id: str
    customer_id: str
    product_id: str
    purchased_at: datetime
    quantity: int

## Value Objects

class ProductCategory:
    """商品カテゴリ値オブジェクト"""
    category: str  # "Cosmetics", "Food", "Apparel"
    
    @staticmethod
    def valid_categories():
        return ["Cosmetics", "Food", "Apparel", "Health"]

class SynergyMatrix:
    """相乗効果マトリクス（値オブジェクト）"""
    source_category: ProductCategory
    target_category: ProductCategory
    score: float  # 0-1.0
    
    def is_high_synergy(self, threshold=0.7):
        return self.score >= threshold

class RecommendationRequest:
    """推薦リクエスト（値オブジェクト）"""
    customer_id: str
    context: dict  # 追加コンテキスト
    limit: int = 5

## Aggregates

class CustomerRecommendationAggregate:
    """顧客推薦の集約"""
    root: Customer
    synergy_db: SynergyMatrix[]
    
    def recommend(self, req: RecommendationRequest) -> Recommendation:
        # 購買履歴から潜在嗜好を推定
        # 相乗効果マトリクスを適用
        # TOP-K 抽出
        ...

## Domain Events

class PurchaseRecorded(DomainEvent):
    """購買が記録された"""
    customer_id: str
    product_id: str
    timestamp: datetime

class RecommendationServed(DomainEvent):
    """推薦が提供された"""
    customer_id: str
    recommended_products: List[str]
    timestamp: datetime

## Repositories

class CustomerRepository:
    """顧客と購買履歴を永続化"""
    def find_by_id(customer_id: str) -> Customer: ...
    def save(customer: Customer) -> None: ...

class SynergyMatrixRepository:
    """相乗効果マトリクスを管理"""
    def load_current() -> SynergyMatrix[]: ...
    def update(matrix: SynergyMatrix[]) -> None: ...

## Ubiquitous Language

- Customer: 顧客
- Purchase: 購買記録
- Product: 商品
- Category: カテゴリ
- Synergy: 相乗効果
- Recommendation: 推薦
- Aggregate: 集約（複数エンティティの論理グループ）
```

#### 1-4. ファシリテータによる調整

参加者全員でレビュー：

```markdown
テックリード: 「SynergyMatrix を値オブジェクトとして扱うのは
           いいですね。変更されないデータだから」

PO: 「ただし、シーズンごとに『食品×コスメ』の相乗効果を
   上げ下げしたい。つまり『版管理』が必要では？」

AI回答を受けて：
「いい指摘です。SynergyMatrix の version を持たせ、
 load_current で最新版を引く仕様にします」

開発者A: 「Customer Aggregate で全購買履歴を
        メモリに持つのは、大規模顧客だとOOMになりませんか？」

テックリード: 「いい質問。レポジトリ層で
           『最近 6 ヶ月分』に限定する工夫をしましょう。
           これを Logical Design で詳細化します」
```

#### 1-5. Domain Model 確定

セッション終了時にファイル化：

```python
# domain_model.py - 確定版

# このファイルはドメイン言語の定義
# インポートされるのは Logical Design と Application層
```

---

### Day 2-3: Logical Design & Architecture Decision

#### 目的
Domain Model を、AWS サービスとクラウド設計パターンに対応付ける

#### 2-1. NFR (非機能要件) の詳細化

AIへのプロンプト:

```
Domain Model を以下の NFR に対応させるため、
Logical Architecture を設計してください。

【NFR】
- Latency: API < 200ms (p99)
- Throughput: 10,000 推薦/秒
- Availability: 99.5%
- Consistency: 相乗効果マトリクス変更は 30 秒以内に反映

【AWS サービス制約】
- Lambda (推薦ロジック実行)
- DynamoDB (購買履歴・推薦結果キャッシュ)
- Lambda@Edge (CDN層のフィルタリング)
- EventBridge (非同期イベント処理)

【質問】
1. データ配置戦略（RDB vs NoSQL vs ベクトルDB）
2. キャッシング戦略（何を、どこに、多久保つ）
3. スケーリング戦略（Lambda 並列度、DynamoDB キャパシティ）
4. 監視・ロギング戦略

各選択に対し、トレードオフ（コスト vs 性能）を述べよ。
```

#### 2-2. AI による Logical Architecture 提案

```markdown
## Logical Architecture: Recommendation Engine

### Data Placement

**Option: DynamoDB + ElastiCache + Lambda**

理由:
- DynamoDB: 購買履歴は Partition Key (Customer ID)、
           Sort Key (Purchase Timestamp) で効率検索可能
- ElastiCache (Redis): 相乗効果マトリクスをメモリ上で保持
                      版管理も容易（version key）
- Lambda: ステートレス、スケーリング簡単

Cost-Performance:
- DynamoDB RCU/WCU を On-Demand にすれば、
  バーストに強く、初期コスト低い
- ElastiCache r7g.large × 2 (レプリカ)：
  月額 ≒ 800 USD、相乗効果マトリクスは数 MB で収まる

### Caching Strategy

**3-tier キャッシング:**

1. Customer Recommendation Cache
   - TTL: 5 分
   - Key: "rec:{customer_id}:{context_hash}"
   - Store: ElastiCache
   - 理由: 同一顧客の連続リクエストを高速化

2. Synergy Matrix Cache
   - TTL: 30 秒（= 要件）
   - Key: "synergy:v{version}"
   - Store: ElastiCache（イベントドリブン更新）
   - 理由: 全リクエストが参照、頻繁に変更

3. Product Hot List
   - TTL: 1 時間
   - Key: "hotlist:{category}"
   - Store: ElastiCache
   - 理由: 在庫が多い商品を優先推薦

### Scaling Strategy

**Lambda:**
- Concurrency: 1000 (initial)
- Auto-scaling: CloudWatch メトリクス "Duration > 100ms" で +500

**DynamoDB:**
- パーティション設計: Customer ID (最大ホット度は ?)
- キャパシティ計算:
  - 読み取り: 10,000 TPS / 100 = 100 RCU (reserved)
  - 書き込み: 推薦記録 2,000 TPS / 1000 = 2 WCU
- スケーリング: Application Auto Scaling 有効

### Monitoring & Logging

**CloudWatch メトリクス:**
- Lambda: Duration, Error, Throttle
- DynamoDB: ConsumedReadCapacity, UserErrors
- ElastiCache: Evictions, Swap, CPU

**X-Ray:**
- エンドツーエンドレイテンシー追跡
- DynamoDB クエリのボトルネック特定

**失敗検知:**
- 推薦 API Error Rate > 1% → Alarm
- p99 Latency > 200ms → Alarm
- DynamoDB Throttle → Emergency Scaling
```

#### 2-3. Architecture Decision Records (ADR) 作成

```markdown
## ADR-001: NoSQL (DynamoDB) を推薦ロジックのデータベースとして採用

### Status
**Accepted** (Day 2 Mob Design session で確定)

### Context
推薦エンジンは:
- 顧客 ID による高速なランダムアクセスが必須
- 購買履歴は時系列で大量（顧客あたり数百～千件）
- スケール: 日 10 万顧客 × 平均 500 購買 = 5,000 万レコード

### Decision
**DynamoDB を採用する**

### Rationale
1. **スケーラビリティ**: ホットな顧客（VIP）による
   単一ノード への負荷集中を、
   自動パーティショニングで吸収
   
2. **Latency**: インデックス不要（Partition + Sort Key で十分）、
           p99 < 10ms 実現可能
   
3. **Operational**: AWS Lambda との統合が密接。
              VPC 不要で低遅延。

### Alternatives Considered
- PostgreSQL RDS:
  - 長所: 複雑クエリ可能
  - 短所: スケーリングに接続管理が必要、
        VPC 内通信でレイテンシ増加 (+30ms)
  
- Elasticsearch:
  - 長所: 柔軟クエリ、全文検索
  - 短所: 本用途では過剰機能、運用複雑

### Consequences
- (+) Lambda との統合シンプル
- (+) Auto-scaling で対応可能
- (-) 複雑な JOIN が不可。推薦ロジック側で対応必須
- (-) 強い一貫性保証なし。最終一貫性の設計が必須

### Related ADRs
- ADR-002: ElastiCache で相乗効果マトリクスを管理
```

---

### Day 3-4: System Context & Component Diagram

#### 目的
マイクロサービス間の境界、イベントフロー、デプロイ戦略を可視化

#### 3-1. C4 Model で設計

```
## Level 3: Component Diagram

[Recommendation Lambda]
  │
  ├─→ [DynamoDB - Purchase History]
  ├─→ [ElastiCache - Synergy Matrix]
  ├─→ [S3 - Model Artifacts]
  │
  └─→ [CloudWatch - Metrics/Logs]

[Marketing Console]
  │
  └─→ [Admin Lambda]
      │
      └─→ [DynamoDB - Synergy Config]
          │
          └─→ [EventBridge]
              │
              └─→ [Notification Lambda] → Email/Slack

[Batch Job (daily)]
  │
  └─→ [Analytics Lambda]
      │
      ├─→ [DynamoDB - Query]
      ├─→ [S3 - Output Parquet]
      │
      └─→ [QuickSight Dashboard]
```

#### 3-2. イベントフロー

```markdown
## Event Flow

### Scenario: 顧客が購入完了した場合

1. [PaymentService] → PurchaseRecorded イベント発行
2. [EventBridge] ルール: "source: payment" 
                       →ターゲット: RecommendationLambda
3. [RecommendationLambda]
   - 入力: PurchaseRecorded イベント
   - 処理:
     a) 顧客の購買履歴を DynamoDB から取得
     b) 相乗効果マトリクスを ElastiCache から取得
     c) ML アルゴリズムで推薦候補を計算
     d) 結果を DynamoDB キャッシュに保存
     e) RecommendationServed イベント発行
4. [EventBridge] ルール: "source: recommendation"
                       →ターゲット: EmailLambda
5. [EmailLambda]
   - 入力: RecommendationServed イベント
   - 処理: Salesforce API で顧客メールアドレス取得
          メール配信タスク作成

Total Latency Target: < 5 分（キューイング許容）
```

#### 3-3. デプロイ戦略

```markdown
## Deployment Strategy

### Phase 1: Development (Week 1-2)
- AWS Sandbox 環境（dev-account）
- ローカル DynamoDB × ElastiCache (Docker Compose)
- テスト環境: e2e テスト用ダミーデータ

### Phase 2: Staging (Week 3)
- AWS Staging 環境（staging-account）
- 本番と同じ AWS 設定（キャパシティ 1/10 スケール）
- E2E テスト、負荷テスト実施
- Canary Deploy テスト

### Phase 3: Production (Week 4)
- Canary Deploy:
  - 新 Lambda バージョン 5% トラフィック
  - 監視 1 時間
  - エラー率・遅延に閾値を設定
  - 良好 → 25% → 50% → 100%

- Rollback トリガー:
  - Error Rate > 5%
  - p99 Latency > 250ms
  - DynamoDB 異常
```

---

## Week 2: Code & Test & Integration

### Day 1: Code Generation & Review

#### 1-1. AI に実装を依頼

```
Logical Design に基づいて、
Python で Recommendation Lambda を実装してください。

【要件】
- ファイル構成:
  - handler.py: Lambda handler
  - recommendation_service.py: ビジネスロジック
  - repositories.py: DynamoDB, ElastiCache へのアクセス
  - models.py: データクラス
  - config.py: AWS リソース接続設定

- テスト:
  - unit_test.py: ユニットテスト（moto で DynamoDB mock）
  - integration_test.py: integration テスト（local stack）

- ベストプラクティス:
  - Type hints 完全対応
  - Docstring 完全記述
  - エラーハンドリング＆ログ出力
  - AWS X-Ray インストルメンテーション
```

#### 1-2. AI コード生成例（一部）

```python
# handler.py
import json
import logging
from recommendation_service import RecommendationService
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

patch_all()
logger = logging.getLogger(__name__)

service = RecommendationService()

@xray_recorder.capture('handler')
def lambda_handler(event, context):
    """
    Lambda handler for recommendation API
    
    Event structure:
    {
        "customer_id": "cust_123",
        "limit": 5,
        "context": {"source": "post_purchase"}
    }
    """
    try:
        customer_id = event.get('customer_id')
        limit = event.get('limit', 5)
        context_data = event.get('context', {})
        
        logger.info(f"Recommend request: customer={customer_id}, limit={limit}")
        
        recommendations = service.recommend(
            customer_id=customer_id,
            limit=limit,
            context=context_data
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'customer_id': customer_id,
                'recommendations': recommendations,
                'timestamp': datetime.utcnow().isoformat()
            })
        }
    
    except ValueError as e:
        logger.error(f"Validation error: {e}")
        return {
            'statusCode': 400,
            'body': json.dumps({'error': str(e)})
        }
    
    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }

# recommendation_service.py
class RecommendationService:
    
    def __init__(self):
        self.repo = PurchaseRepository()
        self.synergy_cache = SynergyCacheService()
        self.ml_engine = RecommendationMLEngine()
    
    def recommend(self, customer_id: str, limit: int, 
                  context: dict) -> List[dict]:
        """
        Generate recommendations for customer
        
        Args:
            customer_id: Target customer ID
            limit: Number of recommendations
            context: Optional context (source, session_id, etc)
        
        Returns:
            List of recommendation dicts
        
        Raises:
            ValueError: If customer_id invalid
        """
        # Validation
        if not customer_id or len(customer_id) > 50:
            raise ValueError(f"Invalid customer_id: {customer_id}")
        
        # Fetch purchase history (cached)
        purchases = self.repo.find_recent_purchases(
            customer_id, months=6, limit=50
        )
        
        # Fetch synergy matrix (from ElastiCache)
        synergy_matrix = self.synergy_cache.load()
        
        # Generate candidates
        candidates = self.ml_engine.generate_candidates(
            purchases, synergy_matrix, limit * 2  # 2x for filtering
        )
        
        # Filter + rank
        final_recs = self._filter_and_rank(
            candidates, purchases, limit
        )
        
        return final_recs
    
    def _filter_and_rank(self, candidates, purchases, limit):
        """Internal ranking logic"""
        # Remove already purchased
        filtered = [
            c for c in candidates
            if c['product_id'] not in {p.product_id for p in purchases}
        ]
        
        # Sort by score
        sorted_recs = sorted(
            filtered,
            key=lambda x: x['score'],
            reverse=True
        )
        
        return sorted_recs[:limit]
```

#### 1-3. Mob Programming で開発

同室で大画面を共有：

```
手順:
1. ドライバー（開発者A）: コードを書く
2. ナビゲーター（テックリード）: 方向性ガイド＆品質チェック
3. 他参加者: 質問・指摘

→ 交代は 30 分ごと

焦点:
- Domain Model との対応: 正しく実装されているか
- テスト容易性: ユニットテストは容易か
- エラーハンドリング: すべてのパスカバーしているか
```

#### 1-4. コードレビュー

```markdown
【レビュー項目チェックリスト】

Domain & Architecture:
- [ ] Domain Model クラスが正しく使われているか
- [ ] リポジトリ層が抽象化されているか
- [ ] ビジネスロジック層と AWS 依存が分離しているか

Performance:
- [ ] DynamoDB クエリ数が最小化されているか
- [ ] キャッシュヒット率が高いか
- [ ] 並列化できる部分を実装しているか

Testing:
- [ ] ユニットテストカバレッジ 80% 以上か
- [ ] エッジケース（購買なし、キャッシュミスなど）をテストしているか
- [ ] Mock/Stub が適切か

Observability:
- [ ] ログレベルが適切か（DEBUG/INFO/ERROR/CRITICAL）
- [ ] トレーサビリティ ID（correlation ID）を含めているか
- [ ] メトリクス出力があるか

Security:
- [ ] AWS IAM ロール権限が最小化されているか（最小権限原則）
- [ ] 顧客 ID のバリデーションがあるか
- [ ] シークレット管理が適切か（ハードコード値なし）
```

---

### Day 2-3: Testing

#### 2-1. Unit Tests（開発者が作成＆Mob review）

```python
# test_recommendation_service.py
import pytest
from unittest.mock import MagicMock, patch
from recommendation_service import RecommendationService

class TestRecommendationService:
    
    @pytest.fixture
    def service(self):
        """Service with mocked repositories"""
        service = RecommendationService()
        service.repo = MagicMock()
        service.synergy_cache = MagicMock()
        service.ml_engine = MagicMock()
        return service
    
    def test_recommend_returns_top_products(self, service):
        """Happy path: returns top N products"""
        # Setup
        purchases = [
            MagicMock(product_id='prod_1'),
            MagicMock(product_id='prod_2')
        ]
        service.repo.find_recent_purchases.return_value = purchases
        service.synergy_cache.load.return_value = {"matrix": "data"}
        service.ml_engine.generate_candidates.return_value = [
            {'product_id': 'prod_3', 'score': 0.9},
            {'product_id': 'prod_4', 'score': 0.85},
            {'product_id': 'prod_1', 'score': 0.8}  # Already purchased
        ]
        
        # Execute
        result = service.recommend('cust_123', limit=2)
        
        # Assert
        assert len(result) == 2
        assert result[0]['product_id'] == 'prod_3'
        assert result[1]['product_id'] == 'prod_4'
    
    def test_recommend_invalid_customer_id_raises(self, service):
        """Invalid customer ID raises ValueError"""
        with pytest.raises(ValueError):
            service.recommend('', limit=5)
        
        with pytest.raises(ValueError):
            service.recommend('x' * 100, limit=5)
    
    def test_recommend_respects_limit(self, service):
        """Returns exactly limit items (or less if insufficient)"""
        service.repo.find_recent_purchases.return_value = []
        service.synergy_cache.load.return_value = {}
        service.ml_engine.generate_candidates.return_value = [
            {'product_id': f'prod_{i}', 'score': 1.0 - i*0.1}
            for i in range(100)
        ]
        
        result = service.recommend('cust_123', limit=5)
        assert len(result) == 5

# Run tests
# $ pytest test_recommendation_service.py -v --cov
```

#### 2-2. Integration Tests（ローカルスタック）

```dockerfile
# docker-compose.yml (local development)
version: '3.8'

services:
  dynamodb:
    image: amazon/dynamodb-local
    ports:
      - "8000:8000"
    environment:
      AWS_ACCESS_KEY_ID: test
      AWS_SECRET_ACCESS_KEY: test
  
  localstack:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      SERVICES: lambda,dynamodb,s3
      DEBUG: 1
```

```python
# test_integration.py
import boto3
from moto import mock_dynamodb
import pytest

@mock_dynamodb
def test_integration_recommendation_end_to_end():
    """Full flow: DynamoDB query → recommendation logic → result"""
    
    # Setup: Create local DynamoDB table
    dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
    table = dynamodb.create_table(
        TableName='Purchases',
        KeySchema=[
            {'AttributeName': 'customer_id', 'KeyType': 'HASH'},
            {'AttributeName': 'purchase_id', 'KeyType': 'RANGE'}
        ],
        AttributeDefinitions=[
            {'AttributeName': 'customer_id', 'AttributeType': 'S'},
            {'AttributeName': 'purchase_id', 'AttributeType': 'S'}
        ],
        BillingMode='PAY_PER_REQUEST'
    )
    
    # Insert test data
    table.put_item(Item={
        'customer_id': 'cust_123',
        'purchase_id': 'pur_1',
        'product_id': 'prod_cosmetics_1',
        'timestamp': '2026-04-01'
    })
    
    # Execute recommendation service
    service = RecommendationService()
    recommendations = service.recommend('cust_123', limit=3)
    
    # Assert
    assert len(recommendations) <= 3
    assert all(rec['product_id'] != 'prod_cosmetics_1' 
               for rec in recommendations)  # No repeat purchase
```

---

### Day 4: Performance & Load Test

#### 4-1. 負荷テスト

```bash
# locustfile.py (負荷テスト)
from locust import HttpUser, task, between
import random

class RecommendationUser(HttpUser):
    wait_time = between(0.5, 2.0)
    
    @task
    def request_recommendation(self):
        customer_id = f"cust_{random.randint(1, 10000)}"
        self.client.get(
            f"/recommendations?customer_id={customer_id}&limit=5"
        )

# 実行:
# locust -f locustfile.py --host=https://dev.api.example.com -u 1000 -r 100 --duration 10m
```

結果収集:

```
Target: p99 Latency < 200ms, Success Rate > 99.5%

Results:
- p50 Latency: 42ms
- p95 Latency: 89ms
- p99 Latency: 156ms
- Error Rate: 0.2%

Bottleneck: DynamoDB read capacity（RCU が 80% 使用）
→ Action: DynamoDB キャパシティ +50 RCU
```

---

### Mob Construction チェックリスト

```markdown
- [ ] Domain Model 確定
- [ ] Logical Architecture 確定（ADR 記録）
- [ ] コード生成＆レビュー完了
- [ ] ユニットテスト 80% 以上カバレッジ
- [ ] Integration テスト Pass
- [ ] 負荷テスト Pass（SLA 達成）
- [ ] コード品質ゲート Pass（Sonarqube, etc）
- [ ] セキュリティスキャン Pass
- [ ] デプロイ手順書作成
```

---

## Week 3-4: Operations フェーズ（別ドキュメント）

次ステップ:
- CI/CD パイプライン設定
- Canary Deploy 実行
- 本番運用トレーニング
- 監視・アラート設定
