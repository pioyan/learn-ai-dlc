# H: AI-DLC 概念図・全体像ダイアグラム

テキストだけでは掴みにくい AI-DLC の構造を、Mermaid 図で視覚的に整理。

---

## 1. AI-DLC 全体フロー

```mermaid
flowchart TD
    A[📋 Intent 定義\nビジネス目標・KPI・スコープ] --> B[Mob Elaboration\n要件明確化 Workshop]
    B --> C[Unit 分解\n独立構築可能な機能単位]
    C --> D[Bolt 計画\n時間単位の最小実装]

    D --> E[Mob Construction\nAI + チームで実装]
    E --> F{Bolt 完了?}
    F -- 次の Bolt --> E
    F -- Unit 完了 --> G{Intent 達成?}
    G -- 次の Unit --> D
    G -- 全 Unit 完了 --> H[🚀 Operations\n本番稼働・監視・改善]
    H --> I{Intent 更新?}
    I -- 新 Intent --> A
    I -- 継続改善 --> H

    style A fill:#4A90D9,color:#fff
    style E fill:#7B68EE,color:#fff
    style H fill:#27AE60,color:#fff
```

---

## 2. Intent → Unit → Bolt の階層構造

```mermaid
graph TD
    I["🎯 Intent\n推薦エンジンで購買単価 +15%\n期間: 3ヶ月"]

    I --> UA["Unit-A\nメール通知サービス\n1〜2週間"]
    I --> UB["Unit-B\n行動ログ収集\n1〜3週間"]
    I --> UC["Unit-C\n推薦アルゴリズムコア\n2〜4週間"]

    UA --> BA1["Bolt-A1\nメールテンプレート設計 4h"]
    UA --> BA2["Bolt-A2\nSES 連携実装 8h"]
    UA --> BA3["Bolt-A3\nテスト 4h"]

    UB --> BB1["Bolt-B1\nイベントスキーマ設計 4h"]
    UB --> BB2["Bolt-B2\nKinesis ストリーム実装 8h"]

    UC --> BC1["Bolt-C1\nDynamoDB スキーマ 4h"]
    UC --> BC2["Bolt-C2\n協調フィルタリング 8h"]
    UC --> BC3["Bolt-C3\nキャッシュ層 4h"]
    UC --> BC4["Bolt-C4\nLambda ハンドラー 4h"]

    style I fill:#E74C3C,color:#fff
    style UA fill:#3498DB,color:#fff
    style UB fill:#3498DB,color:#fff
    style UC fill:#3498DB,color:#fff
```

---

## 3. Mob Construction のサイクル

```mermaid
flowchart LR
    A["📝 Bolt 選択\n次に作るものを決める"] --> B["🤖 AI に指示\nプロンプト設計・送信"]
    B --> C["💻 コード生成\nAI がドラフトを出力"]
    C --> D["👥 Mob レビュー\nチームで確認・議論"]
    D --> E{品質OK?}
    E -- 修正指示 --> B
    E -- OK --> F["✅ テスト実行\nユニット + 統合テスト"]
    F --> G{テスト通過?}
    G -- 失敗 --> B
    G -- 通過 --> H["📦 マージ & デプロイ\n次の Bolt へ"]
    H --> A
```

---

## 4. 従来手法 vs AI-DLC の役割マッピング

```mermaid
quadrantChart
    title 役割の重要度変化（横軸: AI-DLC での重要度, 縦軸: 従来手法での重要度）
    x-axis 低い --> 高い
    y-axis 低い --> 高い
    quadrant-1 重要度維持
    quadrant-2 従来は重要, AI-DLC では低下
    quadrant-3 どちらも低い
    quadrant-4 AI-DLC で新たに重要
    コーディング速度: [0.25, 0.85]
    タイピング技術: [0.1, 0.7]
    Intent 定義力: [0.9, 0.3]
    プロンプト設計: [0.85, 0.1]
    アーキテクチャ判断: [0.9, 0.85]
    コードレビュー: [0.8, 0.75]
    ドメイン知識: [0.95, 0.8]
    AI 出力の評価力: [0.9, 0.05]
```

---

## 5. マイクロサービス Event 駆動フロー（実践例_04 の図解）

```mermaid
sequenceDiagram
    participant C as 顧客
    participant OS as Order Service
    participant EB as EventBridge
    participant PS as Payment Service
    participant NS as Notification Service

    C->>OS: POST /orders (注文作成)
    OS->>OS: Order 作成 (PENDING)
    OS->>EB: OrderCreatedEvent を発行
    EB->>PS: OrderCreatedEvent をルーティング
    PS->>PS: Payment 記録作成
    PS->>PS: 支払い処理（外部 API 呼び出し）
    PS->>EB: PaymentProcessedEvent を発行
    EB->>OS: PaymentProcessedEvent をルーティング
    OS->>OS: Order ステータス → PAID
    EB->>NS: PaymentProcessedEvent をルーティング
    NS->>C: 注文確認メール送信
```

---

## 6. AI-DLC と外部フレームワークの関係

```mermaid
graph TB
    subgraph AI-DLC["AI-DLC コアフレームワーク"]
        direction TB
        INT["Intent"]
        UNIT["Unit"]
        BOLT["Bolt"]
        MOB_E["Mob Elaboration"]
        MOB_C["Mob Construction"]
        INT --> UNIT --> BOLT
        MOB_E --> INT
        BOLT --> MOB_C
    end

    subgraph DORA["DORA メトリクス（測定）"]
        LT["変更リードタイム"]
        DF["デプロイ頻度"]
        CFR["変更失敗率"]
        MTTR["MTTR"]
    end

    subgraph NIST["NIST AI RMF（ガバナンス）"]
        GOV["Govern"]
        MAP["Map"]
        MSR["Measure"]
        MGT["Manage"]
    end

    subgraph OWASP["OWASP（セキュリティ）"]
        SEC["LLM Top 10\nセキュリティ基準"]
    end

    MOB_C --> DORA
    AI-DLC --> NIST
    MOB_C --> OWASP

    style AI-DLC fill:#EBF5FB
    style DORA fill:#EAFAF1
    style NIST fill:#FEF9E7
    style OWASP fill:#FDEDEC
```

---

## 7. 学習フェーズのマップ

```mermaid
journey
    title AI-DLC 学習の旅
    section フェーズ1: 理解
      ロードマップを読む: 5: 学習者
      概念図を見る: 5: 学習者
      実践ガイドを通読: 4: 学習者
      用語集を参照: 3: 学習者
    section フェーズ2: 体験
      実践例01を読む: 4: 学習者
      AIで架空シナリオを試す: 3: 学習者
      プロンプト設計を学ぶ: 4: 学習者
      実践例02のコードを動かす: 3: 学習者
    section フェーズ3: 実践
      チームでMob Elaborationを実施: 3: 学習者, チーム
      振り返りとアンチパターン確認: 4: 学習者, チーム
      DORA メトリクスで測定: 4: チーム
      継続改善: 5: チーム
```

---

## 8. DDD の主要概念の関係図

```mermaid
graph LR
    subgraph BC1["Bounded Context: Order"]
        O_AGG["Order\nAggregate"]
        O_ENT["Order Entity\n(order_id, status)"]
        O_VO["OrderItem\nValue Object"]
        O_REPO["OrderRepository\nInterface"]
        O_EVT["OrderCreatedEvent\nDomain Event"]

        O_AGG --> O_ENT
        O_AGG --> O_VO
        O_REPO --> O_AGG
        O_ENT --> O_EVT
    end

    subgraph BC2["Bounded Context: Payment"]
        P_AGG["Payment\nAggregate"]
        P_ENT["Payment Entity\n(payment_id, status)"]
        P_REPO["PaymentRepository\nInterface"]
        P_EVT["PaymentProcessedEvent\nDomain Event"]

        P_AGG --> P_ENT
        P_REPO --> P_AGG
        P_ENT --> P_EVT
    end

    O_EVT -- "EventBridge でルーティング" --> P_AGG
    P_EVT -- "EventBridge でルーティング" --> O_ENT

    style BC1 fill:#D6EAF8
    style BC2 fill:#D5F5E3
```

---

## 図の見方のポイント

| 図番号 | 何を理解するために使う |
|--------|---------------------|
| 1. 全体フロー | 「AI-DLC はどんな順番で進むのか」 |
| 2. 階層構造 | 「Intent・Unit・Bolt の大きさの違い」 |
| 3. Mob Construction サイクル | 「1 Bolt を実装するときの流れ」 |
| 4. 役割変化 | 「何のスキルが重要になるか」 |
| 5. Event 駆動フロー | 「マイクロサービス間の非同期通信」 |
| 6. フレームワーク関係 | 「AI-DLC と DORA/NIST/OWASP の位置関係」 |
| 7. 学習マップ | 「自分がどのフェーズにいるか」 |
| 8. DDD 概念 | 「Entity・Aggregate・Domain Event の関係」 |
