# I: GitHub Copilot で Claude プラグインを利用する方法

GitHub Copilot では、Claude Code 向け marketplace や plugin 形式の一部をそのまま利用できる。
ここでは、主に GitHub Copilot CLI と VS Code で Claude 系 plugin を使う手順を整理する。

---

## 1. 先に結論

- GitHub Copilot は Claude 形式の plugin / marketplace を扱える場合がある
- まずは **変換せずにそのまま試す** のが基本
- 最も確実なのは **Copilot CLI で marketplace を追加し、plugin を install する方法**
- VS Code では **Chat: Plugins** または **Extensions の `@agentPlugins`** から操作する

---

## 2. 利用前の前提

### Copilot CLI を使う場合

以下が必要:

- GitHub Copilot CLI のインストール
- GitHub へのログイン
- 利用したい plugin が Claude 形式の標準的な構成であること

例:

```powershell
npm install -g @github/copilot-cli
copilot auth login
```

### VS Code を使う場合

以下を確認する:

- GitHub Copilot 拡張が有効
- Agent Plugins 機能が利用可能
- 組織ポリシーで plugin 機能が無効化されていない

---

## 3. Copilot CLI で使う手順

### Step 1. marketplace を追加する

Claude Code の demo marketplace を追加する例:

```powershell
copilot plugin marketplace add anthropics/claude-code
```

この時点では plugin 本体はまだ入らない。  
あくまで「plugin カタログを登録する」だけ。

### Step 2. 登録された marketplace 名を確認する

```powershell
copilot plugin marketplace list
```

表示例:

```text
Registered marketplaces:
  • claude-code-plugins (GitHub: anthropics/claude-code)
```

ここで重要なのは、**追加時の指定名** と **登録後の marketplace 名** は同じとは限らないこと。

- 追加時に使う: `anthropics/claude-code`
- browse / install で使う: `claude-code-plugins`

### Step 3. plugin 一覧を確認する

```powershell
copilot plugin marketplace browse claude-code-plugins
```

これはインストールではなく、**plugin 一覧の閲覧**。

### Step 4. plugin をインストールする

```powershell
copilot plugin install commit-commands@claude-code-plugins
```

一般形:

```powershell
copilot plugin install プラグイン名@マーケットプレイス名
```

### Step 5. インストール済み plugin を確認する

```powershell
copilot plugin list
```

---

## 4. marketplace を経由せず直接入れる方法

特定 plugin の場所が分かっているなら、リポジトリ内サブディレクトリを直接指定できる。

例:

```powershell
copilot plugin install anthropics/claude-code:plugins/frontend-design
```

ローカル clone 済みならローカルパスも使える。

```powershell
copilot plugin install C:\path\to\claude-code\plugins\frontend-design
```

まず marketplace 経由で試し、必要な時だけ直接指定にする方が分かりやすい。

---

## 5. VS Code で使う手順

VS Code では CLI のように terminal から plugin を入れるより、UI 経由の方が自然。

### 方法 A. Plugin 画面から探す

1. Extensions を開く
2. 検索欄で `@agentPlugins` を入力する
3. 目的の plugin を探して Install を押す

### 方法 B. Command Palette から開く

1. `Ctrl+Shift+P`
2. `Chat: Plugins` を実行
3. marketplace を参照して Install する

### 方法 C. marketplace を settings に追加する

`settings.json` に追加:

```json
{
  "chat.plugins.marketplaces": [
    "anthropics/claude-code"
  ]
}
```

これで Claude Code marketplace を VS Code から参照しやすくなる。

---

## 6. よくあるハマりどころ

### 1. marketplace 名を取り違える

以下は別物:

- `anthropics/claude-code` = marketplace のソース
- `claude-code-plugins` = 登録後に browse / install で使う marketplace 名

間違い例:

```powershell
copilot plugin marketplace browse anthropics-claude-code
```

正しい例:

```powershell
copilot plugin marketplace browse claude-code-plugins
```

### 2. plugin が見つからない

まず一覧を確認する:

```powershell
copilot plugin marketplace browse claude-code-plugins
```

### 3. plugin が動いても一部機能が噛み合わない

Claude plugin には次のような Claude 固有要素があることがある:

- `CLAUDE_PLUGIN_ROOT` のような環境変数
- `.claude/` 前提の設定ファイル
- Claude 専用の hook 挙動
- Claude 専用の install 手順

この場合は「plugin をそのまま認識できても、完全互換ではない」ことがある。

---

## 7. まず試すならこの順番

### 最短ルート

```powershell
copilot plugin marketplace add anthropics/claude-code
copilot plugin marketplace list
copilot plugin marketplace browse claude-code-plugins
```

気になる plugin が見つかったら:

```powershell
copilot plugin install プラグイン名@claude-code-plugins
```

---

## 8. 使い分けの目安

### marketplace 経由で入れるべきケース

- まず何があるか見たい
- 更新や再インストールを簡単にしたい
- plugin 名で管理したい

### 直接 install が向くケース

- 特定 plugin だけ試したい
- リポジトリ内の plugin ディレクトリが分かっている
- ローカル clone した plugin を検証したい

---

## 9. 参考コマンド一覧

```powershell
copilot plugin marketplace add anthropics/claude-code
copilot plugin marketplace list
copilot plugin marketplace browse claude-code-plugins
copilot plugin install commit-commands@claude-code-plugins
copilot plugin install anthropics/claude-code:plugins/frontend-design
copilot plugin list
copilot plugin update commit-commands
copilot plugin uninstall commit-commands
```

---

## 10. まとめ

GitHub Copilot で Claude plugin を使う時は、まず「変換」ではなく **そのまま marketplace 追加して試す** のが基本になる。  
特に Copilot CLI では、以下の流れを覚えておけば十分。

1. `copilot plugin marketplace add anthropics/claude-code`
2. `copilot plugin marketplace list`
3. `copilot plugin marketplace browse claude-code-plugins`
4. `copilot plugin install plugin-name@claude-code-plugins`

最初につまずきやすいのは、**ソース名** と **登録後 marketplace 名** の違いだけである。