# モブワーク実践ガイド 5: セキュリティ重視 Mob Construction

## シナリオ
金融機関向けの「顧客資産管理API」を、ISO 27001 + PCI DSS要件を満たしながら実装する Mob Construction

## チャレンジ
- Zero Trust Architecture の実装
- 監査ログの法務要件対応
- 暗号化・鍵管理の自動化
- 本番環境での検証プロセス

---

## Week 1: Security by Design

### Day 1: Threat Model 作成（Mob）

#### 1-1. STRIDE 分析

AIに以下をリクエスト:

```
金融API「顧客資産管理」について、STRIDE Threat Model を作成してください。

【System Context】
- アクター: 顧客、CSR職員、内部監査官
- 主要リソース: 顧客名、口座残高、取引履歴
- 環境: AWS クラウド、HTTPS 通信
- 規制要件: PCI DSS, ISO 27001

【出力】
各 STRIDE 要素について:
1. 脅威: 何が起こり得るか
2. 影響: ビジネス被害
3. 対策: 具体的な実装方法
```

#### 1-2. AI による Threat Model 出力

```markdown
## STRIDE Threat Model

### S - Spoofing Identity

脅威: APIクライアントが他の顧客になりすまし、資産情報を閲覧

影響:
- 顧客資産情報の漏えい
- 法規制違反（PCI DSS 3.2.1）

対策:
1. OAuth 2.0 + JWT ベースの認証
   - API Key ではなく Bearer Token
   - Token 署名は RS256 (RSA)
   
2. mTLS (Mutual TLS) をサービス間通信で強制
   - クライアント証明書検証
   - サーバー証明書検証

実装例:
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) 
        throws Exception {
        return http
            .oauth2ResourceServer(rs -> rs.jwt(jwt -> 
                jwt.jwtAuthenticationConverter(
                    new JwtAuthenticationConverter()
                )
            ))
            .authorizeHttpRequests(authz -> authz
                .requestMatchers(PUBLIC_URLS).permitAll()
                .anyRequest().authenticated()
            )
            .build();
    }
}
```

### T - Tampering with Data

脅威: ネットワーク中の API レスポンスが改ざんされる

影響:
- 顧客に誤った資産情報を提供
- 資産消失の訴訟

対策:
1. TLS 1.3 を強制（All traffic encrypted）
2. Request/Response signing（JWS）
3. Integrity check: HMAC-SHA256

実装:
```python
def add_integrity_header(response_body: str):
    """レスポンスボディに HMAC 署名を追加"""
    hmac_key = os.environ['RESPONSE_SIGN_KEY']
    signature = hmac.new(
        hmac_key.encode(),
        response_body.encode(),
        hashlib.sha256
    ).hexdigest()
    
    return {
        "body": response_body,
        "X-Integrity-Check": signature
    }
```

### R - Repudiation

脅威: ユーザーが「自分は高額送金を指示していない」と否定

影響:
- 纏争増加
- 監査指摘

対策:
1. 全トランザクションの監査ログ記録
   - User ID, Timestamp, Action, Parameters, IP
   - 改ざん防止: append-only ログ
   
2. 電子署名（デジタル署名）要件検討
   - 金融の高額取引は署名要求

実装:
```python
class AuditLog:
    def __init__(self, user_id, action, params):
        self.id = uuid.uuid4()
        self.user_id = user_id
        self.action = action  # "TRANSFER", "VIEW", "DELETE"
        self.params = encrypt(params)  # 機密情報は暗号化
        self.timestamp = datetime.utcnow()
        self.ip_address = get_client_ip()
        self.signature = self.calculate_signature()
    
    def calculate_signature(self):
        """ログの改ざん検知用"""
        data = f"{self.id}:{self.user_id}:{self.action}:{self.timestamp}"
        return hmac.new(
            AUDIT_KEY.encode(),
            data.encode(),
            hashlib.sha256
        ).hexdigest()
```

### I - Information Disclosure

脅威: エラーメッセージから システム情報が漏える

影響:
- 攻撃者が脆弱性を推測できる
- セキュリティスキャン効率化

対策:
1. エラーメッセージの汎用化
   - ユーザーには「エラーが発生しました」
   - ログには詳細を記録
   
2. ログレベルの適切な分離
   - DEBUG: 詳細（本番では無効）
   - ERROR: 汎用メッセージ

```python
def handle_database_error(e: Exception):
    # ユーザーに返す
    return {
        "error": "Internal error occurred",
        "request_id": request_id
    }

# ログには詳細
logger.error(
    f"DB Query failed",
    extra={
        "exception": str(e),
        "query": query,
        "request_id": request_id
    }
)
```

### D - Denial of Service

脅威: 大量リクエストで API が応答不可になる

影響:
- 顧客がサービス利用不可
- SLA 違反

対策:
1. Rate Limiting
   - Per User: 100 req/min
   - Per IP: 1000 req/min
   
2. DDoS 対策: AWS Shield / WAF
3. Circuit Breaker パターン
4. 自動スケーリング

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/assets")
@limiter.limit("100/minute")
def get_assets(request: Request):
    # API 処理
    pass
```

### E - Elevation of Privilege

脅威: 通常ユーザーが Admin 機能にアクセス

影響:
- 全顧客データが盗まれる
- システムが改ざんされる

対策:
1. RBAC (Role-Based Access Control)
   - CUSTOMER: 自分のデータ VIEW のみ
   - CSR: 割り当てられた顧客の VIEW/UPDATE
   - ADMIN: すべての操作
   
2. 最小権限原則 (Principle of Least Privilege)

```python
def check_permission(user, resource_type, action):
    """リソースごとの権限チェック"""
    required_role = PERMISSION_MATRIX[resource_type][action]
    
    if required_role not in user.roles:
        raise ForbiddenError(
            f"User {user.id} lacks {required_role} for {action}"
        )
```
```

#### 1-3. Threat Model チーム確認

```markdown
【Mob レビュー】

テックリード: 「mTLS は本当に必要か？
            Lambda 同士の通信に証明書管理は複雑では」

セキュリティ専門家: 「Lambda 内では IAM ロール で制御可能。
                 ただし外部 API 連携や
                 マルチクラウド環境では必須」

PO: 「初版（3ヶ月パイロット）では mTLS を
    見送り、OAuth + JWT に集中しましょう」

ファシリテータ: 「了解。Threat Model に
            『mTLS は将来実装』と明記します」
```

---

## Week 1-2: Secure Implementation

### Day 2-3: Code Implementation with Security Patterns

#### 2-1. Authentication Middleware

```python
# auth_middleware.py

from fastapi import Depends, HTTPException, Request
from jose import JWTError, jwt
import os

ALGORITHM = "RS256"
PUBLIC_KEY = open(os.environ['JWT_PUBLIC_KEY_PATH']).read()

async def verify_token(token: str) -> dict:
    """JWT トークン検証（署名＋有効期限）"""
    try:
        payload = jwt.decode(
            token,
            PUBLIC_KEY,
            algorithms=[ALGORITHM]
        )
        
        # Token claims 検証
        user_id = payload.get("sub")
        scopes = payload.get("scopes", [])
        exp = payload.get("exp")
        
        if not user_id:
            raise HTTPException(status_code=401, detail="Invalid token")
        
        return {
            "user_id": user_id,
            "scopes": scopes,
            "token_exp": exp
        }
    
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

async def get_current_user(
    request: Request,
    token_data: dict = Depends(verify_token)
) -> dict:
    """リクエストから認証ユーザー取得"""
    
    # IP ホワイトリスト確認（オプション）
    client_ip = request.client.host
    if not is_ip_whitelisted(token_data["user_id"], client_ip):
        logger.warning(f"IP mismatch for user {token_data['user_id']}")
        # 重大リスク判定
    
    return token_data
```

#### 2-2. 暗号化・鍵管理

```python
# encryption.py

import os
from cryptography.fernet import Fernet
from aws_kms import KmsClient

class SecureVault:
    """顧客資産情報の暗号化管理"""
    
    def __init__(self):
        self.kms = KmsClient()
        # AWS KMS で暗号化キーを管理
        self.master_key_id = os.environ['KMS_KEY_ID']
    
    def encrypt_asset_data(self, asset_info: dict) -> str:
        """
        PII (Personally Identifiable Information) を暗号化
        - 顧客名
        - 口座番号
        - 残高
        """
        plaintext = json.dumps(asset_info)
        
        # KMS で Data Key を生成
        data_key_response = self.kms.generate_data_key(
            KeyId=self.master_key_id,
            KeySpec='AES_256'
        )
        
        # Data Key で Fernet 暗号化
        cipher = Fernet(base64.b64encode(
            data_key_response['Plaintext'][:32]
        ))
        ciphertext = cipher.encrypt(plaintext.encode())
        
        # 暗号化されたデータ（ciphertext）を保存
        # KMS 暗号化済みの Data Key も別途保存
        return {
            "encrypted_data": base64.b64encode(ciphertext).decode(),
            "encrypted_key": base64.b64encode(
                data_key_response['CiphertextBlob']
            ).decode()
        }
    
    def decrypt_asset_data(self, encrypted_obj: dict) -> dict:
        """復号（キー管理は自動化）"""
        
        # KMS で Data Key を復号
        decrypt_response = self.kms.decrypt(
            CiphertextBlob=base64.b64decode(
                encrypted_obj['encrypted_key']
            )
        )
        
        # Fernet で データを復号
        cipher = Fernet(base64.b64encode(
            decrypt_response['Plaintext'][:32]
        ))
        plaintext = cipher.decrypt(
            base64.b64decode(encrypted_obj['encrypted_data'])
        )
        
        return json.loads(plaintext.decode())
```

#### 2-3. 監査ログ（改ざん防止）

```python
# audit_log.py

from datetime import datetime
import hashlib
import hmac

class ImmutableAuditLog:
    """Append-only 監査ログ（改ざん検知）"""
    
    def __init__(self, dynamodb_table):
        self.table = dynamodb_table
        self.log_hmac_key = os.environ['AUDIT_HMAC_KEY']
    
    def record(self, user_id: str, action: str, resource: str, 
               result: str, details: dict):
        """監査イベントを記録"""
        
        log_id = str(uuid.uuid4())
        timestamp = datetime.utcnow()
        
        # ハッシュチェーン（前のログとの連鎖）
        prev_hash = self._get_previous_log_hash()
        
        current_data = f"{log_id}:{user_id}:{action}:{resource}:{timestamp}:{prev_hash}"
        current_hash = hmac.new(
            self.log_hmac_key.encode(),
            current_data.encode(),
            hashlib.sha256
        ).hexdigest()
        
        # DynamoDB に append（更新不可）
        self.table.put_item(Item={
            'log_id': log_id,
            'timestamp': int(timestamp.timestamp()),
            'user_id': user_id,
            'action': action,
            'resource': resource,
            'result': result,
            'details': json.dumps(details),
            'prev_hash': prev_hash,
            'current_hash': current_hash,
            'ttl': int(timestamp.timestamp()) + 7*365*24*3600  # 7 年保持
        })
        
        logger.info(f"Audit log recorded: {log_id}")
    
    def _get_previous_log_hash(self) -> str:
        """最新ログのハッシュ取得"""
        response = self.table.query(
            KeyConditionExpression='...',  # 最新ログ取得
            ScanIndexForward=False,
            Limit=1
        )
        
        if response['Items']:
            return response['Items'][0]['current_hash']
        return 'GENESIS'
```

---

## Week 2: Security Validation & Compliance Checklist

### Day 1: 脆弱性スキャン

```bash
#!/bin/bash
# security_scan.sh

# SAST (Static Application Security Testing)
echo "Running SAST..."
sonarqube-scanner \
  -Dsonar.projectKey=financial-api \
  -Dsonar.sources=. \
  -Dsonar.host.url=https://sonarqube.company.com

# Dependency チェック
echo "Checking dependencies..."
pip install safety
safety check --json > dependency_report.json

# Secret スキャン
echo "Scanning for secrets..."
git-secrets --scan

# Docker image scan
echo "Scanning Docker image..."
trivy image financial-api:latest
```

### Day 2: Compliance Checklist

```markdown
## PCI DSS / ISO 27001 Compliance Checklist

### Authentication & Access Control
- [x] 全 API エンドポイントで認証必須
- [x] OAuth 2.0 + JWT 実装
- [x] パスワード要件: 最小 12 文字
- [x] アクセス制御: RBAC 実装
- [x] 定期的な権限レビュー

### Encryption
- [x] 転送中: TLS 1.3 (All endpoints HTTPS-only)
- [x] 保存時: AES-256 (KMS key rotation: 年 1 回)
- [x] バックアップ暗号化
- [x] キー管理: AWS KMS

### Logging & Monitoring
- [x] 全トランザクションログ記録
- [x] ログ保持: 最小 7 年
- [x] ログ改ざん検知（ハッシュチェーン）
- [x] Real-time alerting (失敗率 > 1%, SLA 違反)

### Infrastructure
- [x] WAF 有効化（DDoS, SQL Injection 対策）
- [x] VPC 分離（データベースは Private Subnet）
- [x] Secrets Manager で認証情報管理
- [x] Network Policy: 最小権限原則

### Incident Response
- [x] インシデント対応計画書作成
- [x] 連絡先リスト整備
- [x] 定期的なドリル実施（年 2 回）
- [x] 法的通知手順確立

### Testing
- [x] ペネトレーションテスト（年 1 回、外部業者）
- [x] セキュリティテストカバレッジ 100%
- [x] カオスエンジニアリング（年 2 回）
```

### Day 3: Penetration Test

```markdown
## ペネトレーションテストシナリオ

### Red Team Attack 1: Authentication Bypass

攻撃手法: JWT トークン改ざん

防御確認:
- [x] トークン署名検証: RS256 (秘密鍵がないと改ざん不可)
- [x] Expiration 確認: Token 有効期限チェック
- [x] Scope 検証: Token に含まれる権限確認

結果: 防御成功

### Red Team Attack 2: Data Exfiltration

攻撃手法: SQL Injection 経由でデータベースから顧客情報を抽出

防御確認:
- [x] Parameterized Query: SQL Injection 不可能
- [x] Database Access Logging: すべてのクエリが監査ログに
- [x] Row-level Security: 顧客は自分のデータのみアクセス可能

結果: 防御成功

### Red Team Attack 3: DDoS

攻撃手法: 大量リクエストで API ダウン

防御確認:
- [x] Rate Limiting: Per User 100/min で throttle
- [x] AWS Shield: Layer 3/4 DDoS 対策
- [x] Auto Scaling: Lambda 並列度自動増加

結果: 防御成功 (応答時間増加が限定的)
```

---

## セキュリティ Mob の成功要因

- Threat Model を最初に作成（STRIDE で網羅的に）
- 暗号化・鍵管理を AI + 自動化で実装
- 監査ログは改ざん検知可能に設計
- コード検査（SAST）と脆弱性スキャンを CI に組み込み
- ペネトレーションテストで現実的な攻撃を想定
- Compliance チェックリストで法務/監査対応を自動追跡
