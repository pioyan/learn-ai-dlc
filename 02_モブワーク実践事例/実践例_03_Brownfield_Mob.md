# モブワーク実践ガイド 3: ブラウンフィールド Mob Construction

## シナリオ
既存 e-Commerce システムに「顧客360度ビュー」機能を追加する際の Mob Construction プロセス  
（既存システムの分析・モダナイゼーション→新機能実装）

## 所要時間
Week 1: コード逆解析＆モデリング（3～4 日）
Week 2: 新機能実装＆統合（4～5 日）

---

## Week 1: 既存システムの理解と逆解析

### Day 1-2: Legacy Code Analysis

#### 1-1. 状況把握

既存システムの実態:

```markdown
## 現状分析

### システム構成（レガシー）
- Main DB: Oracle 11g（2010年代に構築）
- アプリ層: Java Spring 3.x（古い）
- API: SOAP + REST（混在）
- テスト: 手動テスト中心
- 監視: なし

### 問題点
- コード整理がされていない（Fat Models）
- ビジネスロジックが UI 層に散在
- トランザクション管理が曖昧
- 変更の影響範囲が不明
→ 新機能追加が危険＋時間がかかる

### ゴール
「顧客360度ビュー」を追加しながら、
該当部分を DDD + AWS モダナイゼーションする
```

#### 1-2. コード逆解析の AI 支援

AIに複数の既存コードスニペットを渡す:

```python
# example_legacy_code.py （実際のコード）

# 顧客マスターテーブル取得ロジック
def get_customer_data(customer_id):
    """
    This is the main method to fetch customer info
    DEPRECATED: Use get_customer_profile instead
    """
    conn = oracle_connection  # グローバル変数
    cursor = conn.cursor()
    
    # 複数クエリで顧客情報取得（N+1 問題あり）
    query1 = f"""
        SELECT * FROM CUSTOMER_MASTER 
        WHERE CUSTOMER_ID = {customer_id}  -- SQL インジェクションリスク
    """
    cursor.execute(query1)
    master = cursor.fetchone()
    
    # 住所情報を別クエリ（N+1）
    query2 = f"SELECT * FROM CUSTOMER_ADDRESS WHERE CUSTOMER_ID = {master[0]}"
    cursor.execute(query2)
    address = cursor.fetchall()
    
    # 支払い情報を別クエリ
    query3 = f"SELECT * FROM PAYMENT_HISTORY WHERE CUSTOMER_ID = {master[0]}"
    cursor.execute(query3)
    payments = cursor.fetchall()
    
    # Bean クラスに手作業で詰める
    customer_bean = CustomerBean()
    customer_bean.id = master[0]
    customer_bean.name = master[1]
    # ... 多数のフィールド代入（エラーの温床）
    
    return customer_bean
```

AIへのプロンプト:

```
上記レガシーコードを分析し、以下を実施してください:

1. Static Model（静的モデル）:
   - エンティティ: 何が主役か
   - 属性: どのデータが紐づくか
   - 関係: テーブル間の関係

2. Dynamic Model（動的モデル）:
   - 「顧客情報取得」というユースケースの
     シーケンス図を作成
   - どのステップが冗長か、改善可能か

3. 技術的問題点:
   - N+1 問題
   - SQL インジェクション
   - グローバル接続管理
   - データバインディングの手作業

4. DDD モデル案:
   - 上記問題を解決する Domain Model を提案
   - Repository パターンで既存 Oracle と接続

5. AWS モダナイゼーション案:
   - どう段階移行するか（Big Bang vs 段階的）
```

#### 1-3. AI による逆解析結果

```markdown
## Reverse Engineering Result

### Static Model

```
Customer (Aggregate Root)
├── customer_id: String (PK)
├── name: String
├── email: String
├── phone: String
├── registration_date: Date
│
Address (Entity) × N
├── address_id: String (PK)
├── customer_id: String (FK)
├── address_type: Enum (HOME, WORK, OTHER)
├── street: String
├── city: String
├── postal_code: String
│
PaymentMethod (Value Object)
├── method_type: Enum (CREDIT_CARD, BANK_TRANSFER)
├── last_four_digits: String
├── is_default: Boolean
│
Transaction (Entity) × N
├── transaction_id: String (PK)
├── customer_id: String (FK)
├── amount: Decimal
├── transaction_date: DateTime
├── payment_method: PaymentMethod
```

### Dynamic Model: Get Customer 360 View

```
[UI] → [CustomerController.getProfile(customerId)]
  │
  ├─→ [CustomerService]
  │   ├─→ [CustomerRepository.findById()]
  │   │   └─→ [Oracle: CUSTOMER_MASTER]
  │   │
  │   ├─→ [AddressRepository.findByCustomerId()]
  │   │   └─→ [Oracle: CUSTOMER_ADDRESS]
  │   │
  │   ├─→ [PaymentRepository.findByCustomerId()]
  │   │   └─→ [Oracle: PAYMENT_HISTORY]
  │   │
  │   └─→ [CustomerProfile DTO] ← Aggregate
  │
  └─→ [JSON Response]

Issues:
- 3 つのクエリが順序実行（遅い）
- 接続管理がグローバル変数
- 各ステップでの例外処理なし
```

### Problems & Solutions

| Problem | Cause | Solution |
|---------|-------|----------|
| N+1 Query | Repository 各々が独立クエリ | LEFT JOIN でまとめる |
| SQL Injection | 文字列連結 | Prepared Statement |
| Global DB Connection | グローバル状態 | DI で接続管理 |
| Manual Mapping | Bean フィールド手作業 | ORM (Hibernate/JPA) |
| No Error Handling | try-catch なし | Exception Translation |

### DDD Model: Customer Aggregate (New)

```java
@Data
@Entity
public class Customer {
    @Id
    private String customerId;
    
    private String name;
    private String email;
    
    @OneToMany(cascade = CascadeType.ALL, mappedBy = "customer")
    private Set<CustomerAddress> addresses;
    
    @Transient
    private PaymentMethods paymentMethods;  // Value Object
    
    @OneToMany(cascade = CascadeType.ALL, mappedBy = "customer")
    private List<Transaction> transactions;
    
    // Domain Methods
    public CustomerProfile get360View() {
        return CustomerProfile.of(this);
    }
    
    public void addAddress(CustomerAddress addr) {
        addresses.add(addr);
        addr.setCustomer(this);
    }
}
```

### AWS Modernization Path

**Phase 1** (Week 1-2): Repository Pattern 導入
- Repository 層を実装
- DAO 層を廃止
- レガシー Oracle はそのまま（Wrapper）

**Phase 2** (Week 3-4): AWS RDS on PostgreSQL へ段階移行
- Oracle ↔ PostgreSQL レプリケーション
- 読み取りを PostgreSQL へ
- 書き込みは Oracle のまま

**Phase 3** (Week 5-6): 完全移行
- 全トラフィック PostgreSQL へ
- Oracle サポート終了
```

#### 1-4. チーム全員による設計確認

```markdown
【Mob Design Session】

テックリード: 「N+1 を LEFT JOIN でまとめるとして、
           Address と Transaction は可変長だから
           制限が必要では？」

シニア開発者: 「最近 6 ヶ月の取引だけを取得しましょう」

PO: 「360 度ビューにはどの情報が必須ですか？」

テックリード: 「CSR が見る画面なので、
           - 基本情報（名前、住所）
           - 過去 6 ヶ月の取引
           - 連絡先
           - デフォルト支払い方法
           が必須」

AI: 「では Query を以下のように最適化できます」

```

---

## Week 2: 新機能実装と既存システムとの統合

### Day 1: Customer 360 API 実装

#### 1-1. AI によるコード生成

```java
// CustomerRepository.java （DDD Repository パターン）
public interface CustomerRepository {
    Optional<Customer> findById(String customerId);
    
    /**
     * Find customer with all related data for 360-degree view
     * Optimized single query with LEFT JOINs
     */
    Optional<Customer360View> find360ViewById(String customerId);
}

// CustomerRepository_Oracle.java （Oracle 実装）
@Repository
public class CustomerRepository_Oracle implements CustomerRepository {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    @Override
    public Optional<Customer360View> find360ViewById(String customerId) {
        String query = """
            SELECT 
                c.CUSTOMER_ID, c.NAME, c.EMAIL, c.PHONE,
                a.ADDRESS_ID, a.ADDRESS_TYPE, a.STREET, a.CITY,
                pm.PAYMENT_METHOD, pm.LAST_FOUR, pm.IS_DEFAULT,
                t.TRANSACTION_ID, t.AMOUNT, t.TRANS_DATE
            FROM CUSTOMER_MASTER c
            LEFT JOIN CUSTOMER_ADDRESS a 
                ON c.CUSTOMER_ID = a.CUSTOMER_ID
            LEFT JOIN PAYMENT_METHOD pm 
                ON c.CUSTOMER_ID = pm.CUSTOMER_ID
            LEFT JOIN (
                SELECT * FROM PAYMENT_HISTORY 
                WHERE TRANSACTION_DATE >= SYSDATE - 180
            ) t ON c.CUSTOMER_ID = t.CUSTOMER_ID
            WHERE c.CUSTOMER_ID = ?
        """;
        
        List<Customer360View> results = jdbc.query(
            query,
            new Customer360ViewRowMapper(),
            customerId
        );
        
        return results.isEmpty() ? Optional.empty() : 
               Optional.of(aggregateRows(results));
    }
    
    private Customer360View aggregateRows(List<Customer360View> rows) {
        // 複数行を集約（Address × N + Transaction × N）
        return rows.stream()
            .collect(Customer360ViewCollector.getInstance());
    }
}

// Customer360ViewService.java （ビジネスロジック）
@Service
public class Customer360ViewService {
    
    @Autowired
    private CustomerRepository repo;
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public Customer360ViewDTO get360View(String customerId) 
        throws CustomerNotFoundException {
        
        Customer360View view = repo.find360ViewById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        
        // Domain Event 発行（監査ログ）
        eventPublisher.publishEvent(
            new Customer360ViewAccessed(customerId)
        );
        
        return Customer360ViewDTO.from(view);
    }
}

// Customer360Controller.java （API エンドポイント）
@RestController
@RequestMapping("/api/v1/customers")
public class Customer360Controller {
    
    @Autowired
    private Customer360ViewService service;
    
    @GetMapping("/{customerId}/profile")
    public ResponseEntity<Customer360ViewDTO> getProfile(
        @PathVariable String customerId
    ) {
        Customer360ViewDTO profile = service.get360View(customerId);
        return ResponseEntity.ok(profile);
    }
}
```

#### 1-2. 統合テスト

```java
@SpringBootTest
@ActiveProfiles("test-oracle")
public class Customer360IntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private TestDataBuilder testDataBuilder;
    
    @Test
    public void testGet360ViewWithAllData() {
        // Setup: テストデータ作成（Oracle の実テーブル）
        String customerId = testDataBuilder
            .createCustomer("John Doe")
            .withAddress("HOME", "123 Main St")
            .withAddress("WORK", "456 Office Blvd")
            .withPaymentMethod("CREDIT_CARD", "****1234")
            .withTransactions(10)  // 過去 6 ヶ月
            .getId();
        
        // Execute
        ResponseEntity<Customer360ViewDTO> response = restTemplate
            .getForEntity(
                "/api/v1/customers/{id}/profile",
                Customer360ViewDTO.class,
                customerId
            );
        
        // Assert
        assertEquals(HttpStatus.OK, response.getStatusCode());
        Customer360ViewDTO profile = response.getBody();
        
        assertNotNull(profile);
        assertEquals("John Doe", profile.getName());
        assertEquals(2, profile.getAddresses().size());
        assertTrue(profile.getTransactions().size() <= 10);
    }
    
    @Test
    public void testNonExistentCustomerReturns404() {
        ResponseEntity<ErrorDTO> response = restTemplate
            .getForEntity(
                "/api/v1/customers/nonexistent/profile",
                ErrorDTO.class
            );
        
        assertEquals(HttpStatus.NOT_FOUND, response.getStatusCode());
    }
}
```

---

### Day 2-3: Backward Compatibility＆段階的切り替え

#### 2-1. Old API との共存

```java
// 既存の古い API を保持（サポート期間は 3 ヶ月）
@RestController
@RequestMapping("/api/legacy/customers")
@Deprecated(since = "v2.0", forRemoval = true)
public class CustomerLegacyController {
    
    @Autowired
    private Customer360ViewService newService;
    
    @Deprecated
    @GetMapping("/{customerId}")
    public ResponseEntity<CustomerBean> getCustomer_OLD(
        @PathVariable String customerId
    ) {
        logger.warn("Using deprecated API: /api/legacy/customers");
        
        // 新 Service で取得
        Customer360ViewDTO newData = newService.get360View(customerId);
        
        // 古い形式に変換して返す
        CustomerBean legacyBean = CustomerBean.from(newData);
        
        return ResponseEntity.ok(legacyBean);
    }
}
```

#### 2-2. Canary Deploy 計画

```markdown
## Canary Deploy Strategy

### Phase 1: Feature Flag（Week 2）
- 新 API 実装完了
- Feature Flag で無効化（デフォルト OFF）
- 内部テストのみ

### Phase 2: Canary（Week 3）
- 5% の CSR ユーザーに対してのみ有効化
- 7 日間監視
- メトリクス: 応答時間、エラー率、ユーザー満足度

結果: 良好 → 25% へ

### Phase 3: Gradual Rollout（Week 4-6）
- 25% → 50% → 100%
- 各段階で 1 週間監視

### Rollback トリガー
- Error Rate > 1%
- p99 Latency > 500ms
- Database Connection Pool Exhaustion
```

---

### Day 3-4: Performance＆Security 検証

#### 3-1. SQL パフォーマンス最適化

```sql
-- Before: N+1 クエリ
-- After: 1 クエリ（最適化）

-- 実行計画確認
EXPLAIN PLAN FOR
SELECT c.CUSTOMER_ID, c.NAME,
       a.ADDRESS_ID, a.ADDRESS_TYPE,
       pm.PAYMENT_METHOD,
       t.TRANSACTION_ID, t.AMOUNT
FROM CUSTOMER_MASTER c
LEFT JOIN CUSTOMER_ADDRESS a ON c.CUSTOMER_ID = a.CUSTOMER_ID
LEFT JOIN PAYMENT_METHOD pm ON c.CUSTOMER_ID = pm.CUSTOMER_ID
LEFT JOIN PAYMENT_HISTORY t ON c.CUSTOMER_ID = t.CUSTOMER_ID
  AND t.TRANSACTION_DATE >= TRUNC(SYSDATE) - 180
WHERE c.CUSTOMER_ID = '12345';

-- インデックス追加
CREATE INDEX idx_customer_address_cid ON CUSTOMER_ADDRESS(CUSTOMER_ID);
CREATE INDEX idx_payment_method_cid ON PAYMENT_METHOD(CUSTOMER_ID);
CREATE INDEX idx_payment_history_date ON PAYMENT_HISTORY(TRANSACTION_DATE, CUSTOMER_ID);
```

#### 3-2. セキュリティレビュー

```markdown
## Security Checklist

- [x] SQL Injection 対策
    - [x] Prepared Statement 使用
    - [x] パラメータバインディング

- [x] Authentication/Authorization
    - [x] CSR には VIEW 権限のみ
    - [x] 顧客自身は自分のデータのみ
    - [x] Admin 権限チェック

- [x] データ保護
    - [x] PII (個人識別情報) のログ出力なし
    - [x] 通信 SSL/TLS 強制
    - [x] データベース接続パスワード管理

- [x] 監査ログ
    - [x] 誰が、何を、いつ見たか記録
    - [x] GDPR 要件対応
```

---

## ブラウンフィールド Mob の特徴

- 既存コード理解に時間をかける（短縮不可）
- 逆解析ツール（AI）を活用して効率化
- Domain Model は「理想形」から設計し、段階導入
- Old API と新 API の共存期間をしっかり計画
- Canary Deploy で低リスク移行
- パフォーマンス改善を可視化して信頼獲得
