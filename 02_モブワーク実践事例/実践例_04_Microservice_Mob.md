# モブワーク実践ガイド 4: マイクロサービス Mob Construction

## シナリオ
支払い機能を独立マイクロサービス化し、既存オーダーシステムと疎結合する際の Mob Construction

## チャレンジ
- 複数チームの並列開発（Service A × Service B）
- 非同期通信設計（EventBridge / Kafka）
- サービス間のトレーサビリティ
- 共有ルール（共通エラーコード、ログ形式）の運用

---

## Week 1: Service Boundary Design（Domain Event 中心）

### Day 1: Service Decomposition

#### 1-1. Domain Event 駆動の設計

```markdown
## 既存 Monolith での Order → Payment フロー

[Monolith]
  OrderService.createOrder()
    ├─ Order エンティティ作成
    ├─ PaymentService.charge() [同期呼び出し] ← 問題: 支払い遅延で全体遅延
    └─ if Success → OrderCreatedEvent 発行

## マイクロサービス化後

[Order Service]
  OrderService.createOrder()
    ├─ Order エンティティ作成
    ├─ OrderCreatedEvent 発行
    └─ return (Customer は Order ID を取得)

[EventBridge/Kafka]
  └─→ OrderCreatedEvent ルーティング

[Payment Service]
  PaymentEventHandler.on(OrderCreatedEvent)
    ├─ Payment 記録作成
    ├─ Payment Provider API 呼び出し（非同期）
    └─ PaymentProcessedEvent or PaymentFailedEvent 発行

[Order Service] リスン
  OrderEventHandler.on(PaymentProcessedEvent)
    ├─ Order ステータス = PAID に更新
    └─ OrderPaidEvent 発行
```

#### 1-2. Event Storming ワークショップ

Mob で大型ホワイトボード（または Miro）を使用:

```
ファシリテータ: 「支払い処理に関わるイベントを全て挙げましょう」

参加者が付箋を貼る:

[Domain Events]
- OrderCreatedEvent
- PaymentInitiatedEvent
- PaymentProcessedEvent
- PaymentFailedEvent
- PaymentRetryEvent
- RefundInitiatedEvent
- OrderPaidEvent
- OrderCancelledEvent

[Aggregates (Service 境界)]
- Order Aggregate (Order Service)
- Payment Aggregate (Payment Service)
- Refund Aggregate (Payment Service)

[Commands]
- CreateOrder
- InitiatePayment
- RetryPayment
- IssueRefund

[Policy (受動的なイベントリスナー)]
- OrderCreatedEvent → InitiatePayment (Payment Service)
- PaymentFailedEvent → NotifyCustomer (Notification Service)
- PaymentProcessedEvent → UpdateOrderStatus (Order Service)
```

#### 1-3. Service Contracts 定義

各マイクロサービスのイベント契約を明示:

```json
// EventSchema: OrderCreatedEvent (Order Service が発行)
{
  "eventType": "order.created",
  "version": "1.0",
  "schema": {
    "orderId": {"type": "string", "format": "uuid"},
    "customerId": {"type": "string"},
    "totalAmount": {"type": "number", "minimum": 0},
    "currency": {"type": "string", "enum": ["USD", "JPY"]},
    "items": {
      "type": "array",
      "items": {
        "productId": {"type": "string"},
        "quantity": {"type": "integer"},
        "price": {"type": "number"}
      }
    },
    "shippingAddress": {"type": "object"},
    "timestamp": {"type": "string", "format": "datetime"}
  }
}

// EventSchema: PaymentProcessedEvent (Payment Service が発行)
{
  "eventType": "payment.processed",
  "version": "1.0",
  "schema": {
    "paymentId": {"type": "string", "format": "uuid"},
    "orderId": {"type": "string", "format": "uuid"},
    "amount": {"type": "number"},
    "currency": {"type": "string"},
    "status": {"type": "string", "enum": ["APPROVED", "FAILED"]},
    "transactionId": {"type": "string"},  // Payment Provider のトランザクション ID
    "timestamp": {"type": "string", "format": "datetime"}
  }
}
```

---

## Week 2: Service Implementation（Team A と Team B が並列）

### Day 1-2: Order Service（Team A）

#### 1-1. Domain Model

```python
# order_domain.py

from dataclasses import dataclass
from enum import Enum
from datetime import datetime
from typing import List

class OrderStatus(Enum):
    PENDING = "PENDING"
    PAYMENT_INITIATED = "PAYMENT_INITIATED"
    PAID = "PAID"
    SHIPPED = "SHIPPED"
    CANCELLED = "CANCELLED"

@dataclass
class OrderItem:
    product_id: str
    quantity: int
    price: float

@dataclass
class Order:
    order_id: str
    customer_id: str
    items: List[OrderItem]
    total_amount: float
    status: OrderStatus = OrderStatus.PENDING
    created_at: datetime = None
    
    def initiate_payment(self):
        """業務ルール: 支払い開始前に Order ステータス更新"""
        if self.status != OrderStatus.PENDING:
            raise ValueError(f"Cannot initiate payment for order in {self.status} status")
        self.status = OrderStatus.PAYMENT_INITIATED
    
    def mark_as_paid(self):
        """支払い完了イベント受信時に呼ばれる"""
        if self.status != OrderStatus.PAYMENT_INITIATED:
            raise ValueError(f"Cannot mark as paid for order in {self.status} status")
        self.status = OrderStatus.PAID
```

#### 1-2. Event Publishing

```python
# order_service.py

from event_publisher import EventPublisher

class OrderService:
    def __init__(self, repo, event_publisher: EventPublisher):
        self.repo = repo
        self.event_publisher = event_publisher
    
    def create_order(self, customer_id: str, items: List[dict]) -> str:
        """
        新規 Order 作成
        副作用: OrderCreatedEvent を EventBridge へ発行
        """
        order_id = generate_uuid()
        total_amount = sum(item['price'] * item['quantity'] for item in items)
        
        order = Order(
            order_id=order_id,
            customer_id=customer_id,
            items=[OrderItem(**item) for item in items],
            total_amount=total_amount,
            created_at=datetime.utcnow()
        )
        
        # Step 1: Aggregate を保存
        self.repo.save(order)
        
        # Step 2: Domain Event を発行
        event = {
            "eventType": "order.created",
            "version": "1.0",
            "orderId": order_id,
            "customerId": customer_id,
            "totalAmount": total_amount,
            "items": items,
            "timestamp": order.created_at.isoformat()
        }
        
        self.event_publisher.publish(event, topic="orders")
        
        return order_id
    
    def handle_payment_processed_event(self, event: dict):
        """
        Payment Service から PaymentProcessedEvent を受信
        → Order ステータスを PAID に更新
        """
        order_id = event['orderId']
        order = self.repo.find_by_id(order_id)
        
        if event['status'] == 'APPROVED':
            order.mark_as_paid()
            self.repo.save(order)
            
            # OrderPaidEvent を発行（Notification Service が受信など）
            self.event_publisher.publish({
                "eventType": "order.paid",
                "orderId": order_id,
                "timestamp": datetime.utcnow().isoformat()
            }, topic="orders")
        else:
            # Payment 失敗 → Retry 指示 or キャンセル指示
            logger.warning(f"Payment failed for order {order_id}")
```

### Day 1-2: Payment Service（Team B）

#### 2-1. Domain Model

```python
# payment_domain.py

class PaymentStatus(Enum):
    INITIATED = "INITIATED"
    PROCESSING = "PROCESSING"
    APPROVED = "APPROVED"
    FAILED = "FAILED"
    REFUNDED = "REFUNDED"

@dataclass
class Payment:
    payment_id: str
    order_id: str
    amount: float
    currency: str
    status: PaymentStatus = PaymentStatus.INITIATED
    created_at: datetime = None
    provider_transaction_id: str = None
    
    def process(self, provider_transaction_id: str):
        """支払い処理を実行"""
        self.status = PaymentStatus.PROCESSING
        self.provider_transaction_id = provider_transaction_id
    
    def approve(self):
        """支払い承認"""
        if self.status != PaymentStatus.PROCESSING:
            raise ValueError("Payment not in PROCESSING state")
        self.status = PaymentStatus.APPROVED
    
    def fail(self):
        """支払い失敗"""
        self.status = PaymentStatus.FAILED
```

#### 2-2. Event Handling

```python
# payment_service.py

class PaymentService:
    def __init__(self, repo, payment_provider, event_publisher):
        self.repo = repo
        self.payment_provider = payment_provider
        self.event_publisher = event_publisher
    
    def handle_order_created_event(self, event: dict):
        """
        Order Service から OrderCreatedEvent を受信
        → Payment 記録を作成 + 支払い処理開始
        """
        order_id = event['orderId']
        amount = event['totalAmount']
        currency = event['currency']
        
        # Step 1: Payment Aggregate を作成
        payment_id = generate_uuid()
        payment = Payment(
            payment_id=payment_id,
            order_id=order_id,
            amount=amount,
            currency=currency,
            created_at=datetime.utcnow()
        )
        
        self.repo.save(payment)
        
        # Step 2: 支払い Provider に依頼（非同期）
        try:
            provider_tx_id = self.payment_provider.authorize(
                order_id=order_id,
                amount=amount,
                currency=currency,
                metadata={"payment_id": payment_id}
            )
            
            payment.process(provider_tx_id)
            self.repo.save(payment)
            
            # Step 3: 支払い確定（実際には webhook で通知されることが多い）
            # ここでは簡略化
            payment.approve()
            self.repo.save(payment)
            
        except PaymentProviderException as e:
            logger.error(f"Payment processing failed: {e}")
            payment.fail()
            self.repo.save(payment)
        
        # Step 4: PaymentProcessedEvent を発行
        result_event = {
            "eventType": "payment.processed",
            "paymentId": payment_id,
            "orderId": order_id,
            "amount": amount,
            "status": payment.status.value,
            "transactionId": payment.provider_transaction_id,
            "timestamp": datetime.utcnow().isoformat()
        }
        
        self.event_publisher.publish(result_event, topic="payments")
```

---

## Week 2: Event Integration＆End-to-End Flow

### Day 3: Event Broker 設定（EventBridge or Kafka）

#### 3-1. AWS EventBridge 設定

```yaml
# eventbridge_config.yaml

EventRules:
  - Name: "OrderCreatedToPaymentService"
    EventBusName: "default"
    EventPattern:
      source: ["order-service"]
      detail-type: ["Order Created"]
    Targets:
      - Arn: "arn:aws:lambda:us-east-1:....:function:payment-handler"
        RoleArn: "arn:aws:iam::...:role/eventbridge-role"
        DeadLetterConfig:
          Arn: "arn:aws:sqs:us-east-1:...:payment-dlq"
  
  - Name: "PaymentProcessedToOrderService"
    EventBusName: "default"
    EventPattern:
      source: ["payment-service"]
      detail-type: ["Payment Processed"]
    Targets:
      - Arn: "arn:aws:lambda:us-east-1:....:function:order-handler"
        RoleArn: "arn:aws:iam::...:role/eventbridge-role"
        DeadLetterConfig:
          Arn: "arn:aws:sqs:us-east-1:...:order-dlq"
```

#### 3-2. End-to-End テスト

```python
# test_order_payment_flow.py

@pytest.mark.integration
class TestOrderPaymentFlow:
    
    @mock_eventbridge
    @mock_dynamodb
    def test_complete_order_creation_and_payment(self):
        """
        Scenario: Order 作成 → Payment 処理 → Order 更新
        """
        
        # Setup
        order_service = OrderService(order_repo, event_publisher)
        payment_service = PaymentService(
            payment_repo, 
            mock_payment_provider, 
            event_publisher
        )
        
        # Step 1: Order 作成
        order_id = order_service.create_order(
            customer_id="cust_123",
            items=[
                {"product_id": "prod_1", "quantity": 2, "price": 50.0}
            ]
        )
        
        # Verify: OrderCreatedEvent が published される
        published_events = event_publisher.get_published_events()
        assert len(published_events) == 1
        assert published_events[0]['eventType'] == 'order.created'
        
        # Step 2: Payment Service が event を handle
        order_created_event = published_events[0]
        payment_service.handle_order_created_event(order_created_event)
        
        # Verify: Payment が APPROVED 状態に
        payment = payment_repo.find_by_order_id(order_id)
        assert payment.status == PaymentStatus.APPROVED
        
        # Verify: PaymentProcessedEvent が published
        payment_events = [
            e for e in event_publisher.get_published_events()
            if e['eventType'] == 'payment.processed'
        ]
        assert len(payment_events) == 1
        
        # Step 3: Order Service が PaymentProcessedEvent を handle
        payment_processed_event = payment_events[0]
        order_service.handle_payment_processed_event(
            payment_processed_event
        )
        
        # Verify: Order が PAID 状態に
        order = order_repo.find_by_id(order_id)
        assert order.status == OrderStatus.PAID
```

---

## Week 3-4: Observability & Resilience

### Day 1-2: Distributed Tracing

```python
# tracing.py

from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

patch_all()

class TracingMiddleware:
    """すべての inter-service 呼び出しにトレースコンテキストを付与"""
    
    def __call__(self, event, context):
        # X-Ray に event 情報をログ
        xray_recorder.put_annotation("order_id", 
                                      event.get("orderId"))
        xray_recorder.put_annotation("service", 
                                      context.invoked_function_name)
```

### Day 2-3: Error Handling & Retry Policy

```python
# resilience.py

from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def call_payment_provider(order_id, amount):
    """
    Payment Provider API を呼び出し
    失敗時は指数バックオフで Retry
    """
    try:
        response = requests.post(
            f"https://provider.api/charge",
            json={"order_id": order_id, "amount": amount},
            timeout=5
        )
        response.raise_for_status()
        return response.json()
    except requests.RequestException as e:
        logger.warning(f"Payment provider call failed: {e}")
        raise

# Circuit Breaker パターン
from pybreaker import CircuitBreaker

payment_api_breaker = CircuitBreaker(
    fail_max=5,
    reset_timeout=60,
    exclude=[ValueError]  # ValueError は breaker を trip させない
)

@payment_api_breaker
def charge_with_breaker(order_id, amount):
    return call_payment_provider(order_id, amount)
```

---

## マイクロサービス Mob の成功要因

✓ Domain Event を中心に Service Boundary を引く
✓ Event Contract（JSON Schema）を公開・共有
✓ Async First の設計思想
✓ 各チームが独立開発可能な構造
✓ Distributed Tracing で全フロー可視化
✓ Resilience pattern（Retry, Circuit Breaker）を埋め込み
✓ End-to-End テストで統合検証
