# AI-DLC 理解と実践ガイド

この文書は、AI-Driven Development Lifecycle（AI-DLC）を短期間で理解し、現場に導入するための実践ガイドです。既存のAI-DLC原典翻訳に加え、公開情報（DORA、NIST、OWASP、Thoughtworks）を参照して補強しています。

- 対象読者: 開発リーダー、テックリード、PO、アーキテクト、開発者
- 想定: 複数チームで中〜大規模システムを継続開発している組織
- 目的: 「AIを使う」から「AIを前提に開発方式を再設計する」へ移行する

---

## 1. まず3分で掴む AI-DLC

AI-DLCは、従来の「人間が計画してAIは補助する」流れを反転し、次の分業を前提にします。

- AI: 意図の分解、設計案提示、コード/テスト生成、運用分析
- 人間: 重要意思決定、妥当性検証、リスク判断、ガバナンス

キーワードは次の3つです。

- Intent: 何を達成したいか（高レベル目的）
- Unit: Intentを価値単位へ分解した作業塊
- Bolt: 時間〜日単位で回す最小反復

従来の週単位スプリント中心設計ではなく、短周期で「AI提案→人間検証→即実行」を高頻度に回す点が本質です。

---

## 2. なぜAI-DLCが必要か

AIコーディング支援が普及すると、局所作業（実装・修正）は速くなります。一方で、全体最適（品質、安定性、価値創出速度）が改善しないケースが増えます。

典型例:

- 生成コードは増えるが、リワークも増える
- 変更は速いが、設計の整合が崩れる
- 速度は上がるが、セキュリティ/コンプライアンスの確認が追いつかない

AI-DLCはこの問題に対し、手法側で次を組み込むアプローチです。

- 設計技法（例: DDD）を手法のコアに入れる
- フェーズ間の成果物連結（トレーサビリティ）を強制する
- 人間のレビュー点を「最小限だが必須」に絞る

---

## 3. AI-DLCの全体像（実務版）

### 3.1 フェーズ

- Inception:
  - Intentの明確化
  - ユーザーストーリー/NFR/リスク定義
  - UnitとBolt案の作成
- Construction:
  - Domain Design（業務モデル）
  - Logical Design（NFRとアーキ設計）
  - コード/テスト生成と検証
- Operations:
  - デプロイ、監視、異常検知、運用改善

### 3.2 重要な運用原則

- AIの提案は「たたき台」であり、決定は人間が行う
- 成果物を次工程の文脈として必ず残す
- 大きな一括投入より、小さなUnit/Boltで進める
- 速度だけでなく安定性指標を同時監視する

---

## 4. 他手法との違い（短く）

- Scrumとの差:
  - Sprint（週単位）より短いBolt（時間〜日単位）
  - AI主導の分解と提案が前提
- 従来SDLCとの差:
  - 人手中心の段階ゲートから、AI提案+人間検証の連続フローへ
- 単純なAI導入との差:
  - ツール追加ではなく、役割・成果物・儀式を再設計する

---

## 5. 外部知見で補強する実践ポイント

### 5.1 指標設計: DORA系で速度と安定性を同時に見る

DORA研究は、開発能力と組織成果の関係を継続的に示しており、AI時代でも「フローと安定性」の同時改善が重要です。Thoughtworks Radar（2026）でも、AI時代ほどDORA指標の重要性が増すと整理されています。

最小セット（推奨）:

- Change lead time
- Deployment frequency
- Change failure rate
- MTTR（平均復旧時間）
- Rework rate（AI導入時の劣化早期検知に有効）

運用のコツ:

- AI生成行数やプロンプト数を主KPIにしない
- 週次で「速度×安定性」の両面レビューを行う
- リワーク増加を、設計/レビュー不足の早期シグナルとして扱う

### 5.2 ガバナンス: NIST AI RMFで統制を設計する

NIST AI RMFは、AIリスク管理の共通フレームです。AI-DLCでは次の対応が実装しやすいです。

- Govern:
  - AI利用方針、責任分界、承認ルールを定義
- Map:
  - どの工程で何のリスクが出るかを特定
- Measure:
  - 幻覚、脆弱性混入、説明不能判断などを測定
- Manage:
  - 重大リスクに対する是正手順を定義

実務では、AI-DLCの各成果物（要件、設計、テスト、運用記録）をAI RMFの証跡として扱うと監査しやすくなります。

### 5.3 セキュリティ: OWASP GenAI/LLM Top 10を最低ラインにする

AIエージェント導入時は、一般的なアプリ脆弱性に加えて、LLM特有の脅威（プロンプト注入、機密漏えい、ツール悪用など）を扱う必要があります。

実装の最低ライン:

- 入出力フィルタとポリシー判定
- ツール呼び出し権限の最小化
- 高権限操作の人間承認
- プロンプト/応答/ツール実行の監査ログ
- モデル更新時の回帰テスト

### 5.4 導入技法: Context Engineeringを最初に整える

Thoughtworks Radarで強調される通り、AI活用の成否は「プロンプト文言」より「文脈設計」に依存します。

推奨実装:

- 全文脈を一度に渡さず、段階的に読み込む
- 組織標準（設計規約、セキュリティ方針、レビュー観点）を共有指示として管理
- 長い作業では中間成果を要約し、文脈の劣化を防ぐ

AI-DLCで言えば、Intent→Unit→Design→Code→Opsの成果物連結そのものが、実務的なContext Engineeringになります。

---

## 6. 90日導入ロードマップ

### Day 0-30: 基盤整備

- パイロット1案件を選定（複雑すぎず、価値が測れる領域）
- AI利用ポリシーと承認ルールを文書化
- 成果物テンプレートを定義（Intent, Unit, ADR, Test, Risk）
- 計測基盤を設定（DORA指標 + Rework率）

完了条件:

- チーム全員が同じプロセスで1Boltを完了できる

### Day 31-60: 実行最適化

- Mob ElaborationとMob Constructionを定着
- 設計レビュー観点を標準化（DDD境界、NFR、セキュリティ）
- テスト自動化と回帰テストを強化
- 失敗事例をナレッジ化して再発防止

完了条件:

- 2〜3回のBoltで、品質劣化なしにリードタイム短縮が見える

### Day 61-90: スケール

- 2チーム目へ展開
- 共通指示（agent instructions）とテンプレートをサービス雛形に埋め込み
- 監査ログと運用手順を整備
- 四半期単位のROI評価を開始

完了条件:

- チーム横断で同じ品質基準と計測軸で運用できる

---

## 7. 失敗しやすいポイント

- 速度指標だけを追い、障害率やリワークを見ない
- AI提案を未検証で採用する
- 成果物を残さず、再現できない開発になる
- セキュリティを後工程に回す
- 「個人のうまいプロンプト」に依存してチーム標準化しない

---

## 8. すぐ使えるチェックリスト

- Intentが事業価値に結びついている
- Unitが疎結合かつ独立デプロイ可能
- Boltのスコープが1日以内で検証可能
- DDD観点の設計レビューを通過
- 機能/性能/セキュリティテストが自動化されている
- 高リスク操作に人間承認がある
- DORA+Rework指標を週次で確認している

---

## 9. 参考情報（公開情報）

- AI-DLC原文
  - https://prod.d13rzhkk8cj2z0.amplifyapp.com/
- DORA Research Program
  - https://dora.dev/research/
- DORA AI（ROI / Capabilities Model）
  - https://dora.dev/ai/
- GitHub Copilot研究（生産性・満足度・速度）
  - https://github.blog/news-insights/research/research-quantifying-github-copilots-impact-on-developer-productivity-and-happiness/
- NIST AI Risk Management Framework
  - https://www.nist.gov/itl/ai-risk-management-framework
- OWASP GenAI / LLM Top 10
  - https://owasp.org/www-project-top-10-for-large-language-model-applications/
  - https://genai.owasp.org/llm-top-10/
- Thoughtworks Technology Radar（2026, Techniques）
  - https://www.thoughtworks.com/en-us/radar/techniques/ai-assisted-software-delivery

---

## 10. 最後に

AI-DLCの導入で最も重要なのは、AIの出力量を増やすことではなく、

- 意思決定の質
- 変更の安定性
- 学習ループの速さ

を同時に高めることです。

「AIに何を書かせるか」よりも「どの判断点で人間が責任を持つか」を先に設計すると、導入成功率は大きく上がります。
