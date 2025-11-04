# Falcon MCP Docker修正・デプロイメント手順書

## 概要

このドキュメントでは、`quay.io/crowdstrike/falcon-mcp:latest` Dockerイメージを使用している環境で、Content-Lengthヘッダー問題を解決するためのソースコード修正から、テスト、再コンテナ化、GitLabへの登録までの一連の手順を説明します。

## 前提条件

- Docker及びDocker Composeがインストールされている
- GitLabアクセス権限がある
- 基本的なLinux/Dockerコマンドの知識がある
- CrowdStrike Falcon APIの認証情報がある

## 手順概要

1. [既存コンテナからソースコード抽出](#1-既存コンテナからソースコード抽出)
2. [ソースコード修正](#2-ソースコード修正)
3. [ローカルテスト](#3-ローカルテスト)
4. [Dockerイメージ再構築](#4-dockerイメージ再構築)
5. [統合テスト](#5-統合テスト)
6. [GitLabへの登録](#6-gitlabへの登録)
7. [本番環境デプロイ](#7-本番環境デプロイ)

---

## 1. 既存コンテナからソースコード抽出

### 1.1 作業ディレクトリの準備

```bash
# 作業ディレクトリを作成
mkdir -p ~/falcon-mcp-custom
cd ~/falcon-mcp-custom

# Gitリポジトリとして初期化
git init
```

### 1.2 既存イメージからソースコード抽出

```bash
# 既存のイメージをpull
docker pull quay.io/crowdstrike/falcon-mcp:latest

# 一時コンテナを作成してソースコードを抽出
docker create --name falcon-mcp-temp quay.io/crowdstrike/falcon-mcp:latest
docker cp falcon-mcp-temp:/app ./
docker rm falcon-mcp-temp

# 抽出したファイルを確認
ls -la app/
```

### 1.3 Dockerfileの作成

```bash
# 元のイメージの情報を確認
docker inspect quay.io/crowdstrike/falcon-mcp:latest

# Dockerfileを作成
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

# 作業ディレクトリを設定
WORKDIR /app

# システムパッケージの更新とインストール
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Pythonの依存関係をコピーしてインストール
COPY app/pyproject.toml app/uv.lock* ./
RUN pip install --no-cache-dir uv && \
    uv pip install --system -r pyproject.toml

# アプリケーションコードをコピー
COPY app/ ./

# エントリーポイントを設定
ENTRYPOINT ["python", "-m", "falcon_mcp.server"]
EOF
```

---

## 2. ソースコード修正

### 2.1 修正対象ファイルの特定

```bash
# client.pyファイルの場所を確認
find app/ -name "client.py" -type f
```

### 2.2 Content-Lengthヘッダー修正の適用

`app/falcon_mcp/client.py` ファイルを以下のように修正します：

```python
# 修正前の import部分に json を追加
import json

# ContentLengthAPIHarnessクラスを追加
class ContentLengthAPIHarness(APIHarnessV2):
    """Custom APIHarnessV2 that ensures Content-Length header is set."""
    
    def command(self, operation: str, **kwargs) -> Dict[str, Any]:
        """Execute a Falcon API command with Content-Length header enforcement."""
        # Ensure Content-Length header is set for requests with body
        if 'body' in kwargs and kwargs['body'] is not None:
            # Prepare headers
            if 'headers' not in kwargs:
                kwargs['headers'] = {}
            
            # Calculate content length based on body type
            body_data = kwargs['body']
            if isinstance(body_data, dict):
                # Convert dict to JSON string
                body_json = json.dumps(body_data, separators=(',', ':'))
                content_length = len(body_json.encode('utf-8'))
                # Update the body to be the JSON string
                kwargs['body'] = body_json
            elif isinstance(body_data, str):
                content_length = len(body_data.encode('utf-8'))
            elif isinstance(body_data, bytes):
                content_length = len(body_data)
            else:
                # Convert to string and calculate length
                body_str = str(body_data)
                content_length = len(body_str.encode('utf-8'))
                kwargs['body'] = body_str
            
            # Set Content-Length header
            kwargs['headers']['Content-Length'] = str(content_length)
            
            # Also set Content-Type if not already set
            if 'Content-Type' not in kwargs['headers']:
                kwargs['headers']['Content-Type'] = 'application/json'
            
            logger.debug("Set Content-Length: %d for operation: %s", content_length, operation)
        
        return super().command(operation, **kwargs)

# FalconClientクラス内で APIHarnessV2 を ContentLengthAPIHarness に変更
# self.client = ContentLengthAPIHarness(...)
```

### 2.3 修正内容の確認

```bash
# 修正内容をGitで管理
git add .
git commit -m "Initial commit: Extract source from official image"
git commit -am "Fix: Add Content-Length header support for proxy compatibility"
```

---

## 3. ローカルテスト

### 3.1 テスト環境の準備

```bash
# テスト用の環境変数ファイルを作成
cat > .env.test << 'EOF'
FALCON_CLIENT_ID=your_client_id_here
FALCON_CLIENT_SECRET=your_client_secret_here
FALCON_BASE_URL=https://api.crowdstrike.com
EOF

# テスト用のdocker-compose.ymlを作成
cat > docker-compose.test.yml << 'EOF'
version: '3.8'
services:
  falcon-mcp-test:
    build: .
    env_file: .env.test
    ports:
      - "3000:3000"
    volumes:
      - ./logs:/app/logs
    command: ["python", "-m", "falcon_mcp.server", "--debug"]
EOF
```

### 3.2 ローカルビルドとテスト

```bash
# Dockerイメージをビルド
docker-compose -f docker-compose.test.yml build

# テスト実行
docker-compose -f docker-compose.test.yml up -d

# ログを確認
docker-compose -f docker-compose.test.yml logs -f

# ヘルスチェック
curl -X POST http://localhost:3000/health || echo "Health check endpoint may not be available"
```

### 3.3 Intel モジュールの機能テスト

```bash
# MCPクライアントを使用したテスト（例：mcp-use）
# または、直接APIテストを実行

# コンテナ内でのテスト実行
docker-compose -f docker-compose.test.yml exec falcon-mcp-test python -c "
from falcon_mcp.client import FalconClient
from falcon_mcp.modules.intel import IntelModule

# クライアント初期化テスト
try:
    client = FalconClient()
    print('✓ Client initialization successful')
    
    # 認証テスト
    if client.authenticate():
        print('✓ Authentication successful')
        
        # Intel モジュールテスト
        intel = IntelModule(client)
        print('✓ Intel module initialization successful')
    else:
        print('✗ Authentication failed')
except Exception as e:
    print(f'✗ Test failed: {e}')
"
```

---

## 4. Dockerイメージ再構築

### 4.1 本番用Dockerfileの最適化

```bash
# マルチステージビルドを使用した最適化版Dockerfile
cat > Dockerfile.prod << 'EOF'
# Build stage
FROM python:3.11-slim as builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy dependency files
COPY app/pyproject.toml app/uv.lock* ./

# Install dependencies
RUN pip install --no-cache-dir uv && \
    uv pip install --system -r pyproject.toml

# Production stage
FROM python:3.11-slim

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# Copy application code
COPY app/ ./

# Create non-root user
RUN useradd --create-home --shell /bin/bash falcon && \
    chown -R falcon:falcon /app

USER falcon

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import falcon_mcp.client; print('OK')" || exit 1

ENTRYPOINT ["python", "-m", "falcon_mcp.server"]
EOF
```

### 4.2 本番イメージのビルド

```bash
# 本番用イメージをビルド
docker build -f Dockerfile.prod -t falcon-mcp-custom:latest .

# イメージサイズを確認
docker images | grep falcon-mcp

# セキュリティスキャン（可能であれば）
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image falcon-mcp-custom:latest
```

---

## 5. 統合テスト

### 5.1 本番環境類似テスト

```bash
# 本番環境類似の設定でテスト
cat > docker-compose.integration.yml << 'EOF'
version: '3.8'
services:
  falcon-mcp:
    image: falcon-mcp-custom:latest
    env_file: .env.test
    restart: unless-stopped
    networks:
      - falcon-network
    healthcheck:
      test: ["CMD", "python", "-c", "import falcon_mcp.client; print('OK')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  falcon-network:
    driver: bridge
EOF

# 統合テスト実行
docker-compose -f docker-compose.integration.yml up -d

# ヘルスチェック確認
docker-compose -f docker-compose.integration.yml ps
```

### 5.2 プロキシ環境でのテスト

```bash
# プロキシ設定を含むテスト環境
cat > docker-compose.proxy-test.yml << 'EOF'
version: '3.8'
services:
  falcon-mcp:
    image: falcon-mcp-custom:latest
    env_file: .env.test
    environment:
      - HTTP_PROXY=http://your-proxy:port
      - HTTPS_PROXY=http://your-proxy:port
      - NO_PROXY=localhost,127.0.0.1
    networks:
      - falcon-network

networks:
  falcon-network:
    driver: bridge
EOF

# プロキシ環境でのテスト
docker-compose -f docker-compose.proxy-test.yml up -d
docker-compose -f docker-compose.proxy-test.yml logs -f
```

---

## 6. GitLabへの登録

### 6.1 GitLabリポジトリの準備

```bash
# GitLabリポジトリを作成（WebUIまたはAPI経由）
# 例：https://your-gitlab.com/your-group/falcon-mcp-custom

# リモートリポジトリを追加
git remote add origin https://your-gitlab.com/your-group/falcon-mcp-custom.git
```

### 6.2 ソースコードのプッシュ

```bash
# ブランチ戦略の設定
git checkout -b main
git push -u origin main

# 開発ブランチの作成
git checkout -b develop
git push -u origin develop

# 修正内容をfeatureブランチで管理
git checkout -b feature/content-length-fix
git push -u origin feature/content-length-fix
```

### 6.3 GitLab CI/CDパイプラインの設定

```bash
# .gitlab-ci.ymlを作成
cat > .gitlab-ci.yml << 'EOF'
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

before_script:
  - docker info

test:
  stage: test
  script:
    - docker build -t falcon-mcp-test .
    - docker run --rm falcon-mcp-test python -m pytest tests/ || echo "No tests found"
  only:
    - merge_requests
    - develop
    - main

build:
  stage: build
  script:
    - docker build -f Dockerfile.prod -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

deploy_staging:
  stage: deploy
  script:
    - echo "Deploy to staging environment"
    # ここにステージング環境へのデプロイスクリプトを記述
  only:
    - develop
  when: manual

deploy_production:
  stage: deploy
  script:
    - echo "Deploy to production environment"
    # ここに本番環境へのデプロイスクリプトを記述
  only:
    - main
  when: manual
EOF
```

### 6.4 ドキュメントの追加

```bash
# READMEの更新
cat > README.md << 'EOF'
# Falcon MCP Custom

CrowdStrike Falcon MCP Server with Content-Length header fix for proxy environments.

## Changes from Official Version

- Added Content-Length header support for proxy compatibility
- Custom ContentLengthAPIHarness class implementation

## Usage

### Docker Compose

```yaml
version: '3.8'
services:
  falcon-mcp:
    image: your-registry/falcon-mcp-custom:latest
    environment:
      - FALCON_CLIENT_ID=your_client_id
      - FALCON_CLIENT_SECRET=your_client_secret
      - FALCON_BASE_URL=https://api.crowdstrike.com
```

### Environment Variables

- `FALCON_CLIENT_ID`: CrowdStrike API Client ID
- `FALCON_CLIENT_SECRET`: CrowdStrike API Client Secret  
- `FALCON_BASE_URL`: Falcon API Base URL (default: https://api.crowdstrike.com)

## Testing

```bash
docker-compose -f docker-compose.test.yml up -d
```

## Deployment

See [DEPLOYMENT.md](DEPLOYMENT.md) for detailed deployment instructions.
EOF

# 変更履歴の記録
cat > CHANGELOG.md << 'EOF'
# Changelog

## [Custom-1.0.0] - $(date +%Y-%m-%d)

### Added
- Content-Length header support for proxy environments
- ContentLengthAPIHarness class for automatic header management

### Fixed
- HTTP 411 Length Required error in proxy environments
- Missing Content-Length header in API requests with body

### Changed
- FalconClient now uses ContentLengthAPIHarness instead of APIHarnessV2
EOF
```

---

## 7. 本番環境デプロイ

### 7.1 本番用Docker Composeファイル

```bash
# 本番環境用のdocker-compose.yml
cat > docker-compose.prod.yml << 'EOF'
version: '3.8'
services:
  falcon-mcp:
    image: your-gitlab-registry/falcon-mcp-custom:latest
    restart: unless-stopped
    environment:
      - FALCON_CLIENT_ID=${FALCON_CLIENT_ID}
      - FALCON_CLIENT_SECRET=${FALCON_CLIENT_SECRET}
      - FALCON_BASE_URL=${FALCON_BASE_URL}
      - HTTP_PROXY=${HTTP_PROXY}
      - HTTPS_PROXY=${HTTPS_PROXY}
      - NO_PROXY=${NO_PROXY}
    volumes:
      - ./logs:/app/logs
      - ./config:/app/config:ro
    networks:
      - falcon-network
    healthcheck:
      test: ["CMD", "python", "-c", "import falcon_mcp.client; print('OK')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  falcon-network:
    driver: bridge
EOF
```

### 7.2 デプロイメントスクリプト

```bash
# deploy.shスクリプトを作成
cat > deploy.sh << 'EOF'
#!/bin/bash

set -e

# 設定
REGISTRY="your-gitlab-registry"
IMAGE_NAME="falcon-mcp-custom"
TAG="${1:-latest}"

echo "Deploying Falcon MCP Custom version: $TAG"

# 最新イメージをpull
docker pull $REGISTRY/$IMAGE_NAME:$TAG

# 既存コンテナを停止
docker-compose -f docker-compose.prod.yml down

# 新しいバージョンでデプロイ
export IMAGE_TAG=$TAG
docker-compose -f docker-compose.prod.yml up -d

# ヘルスチェック
echo "Waiting for service to be healthy..."
timeout 60 bash -c 'until docker-compose -f docker-compose.prod.yml ps | grep -q "healthy"; do sleep 5; done'

echo "Deployment completed successfully!"
EOF

chmod +x deploy.sh
```

### 7.3 ロールバック手順

```bash
# rollback.shスクリプトを作成
cat > rollback.sh << 'EOF'
#!/bin/bash

set -e

PREVIOUS_TAG="${1:-previous}"

echo "Rolling back to version: $PREVIOUS_TAG"

# 前のバージョンにロールバック
export IMAGE_TAG=$PREVIOUS_TAG
docker-compose -f docker-compose.prod.yml down
docker-compose -f docker-compose.prod.yml up -d

echo "Rollback completed!"
EOF

chmod +x rollback.sh
```

---

## テストタイミングとチェックポイント

### 開発段階でのテスト

1. **コード修正後**: 構文チェック、基本的な動作確認
2. **ローカルビルド後**: Dockerイメージの正常ビルド確認
3. **機能テスト**: Intel モジュールの各機能テスト

### 統合テスト段階

1. **プロキシ環境テスト**: Content-Lengthヘッダーの動作確認
2. **パフォーマンステスト**: レスポンス時間の確認
3. **セキュリティテスト**: 脆弱性スキャン

### デプロイ前テスト

1. **ステージング環境**: 本番環境と同等の設定でのテスト
2. **負荷テスト**: 想定される負荷での動作確認
3. **監視テスト**: ログ出力、メトリクス収集の確認

### 本番デプロイ後

1. **ヘルスチェック**: サービスの正常起動確認
2. **機能確認**: 主要機能の動作テスト
3. **監視確認**: エラーログ、パフォーマンスメトリクスの確認

---

## トラブルシューティング

### よくある問題と解決方法

1. **ビルドエラー**: 依存関係の問題
   ```bash
   docker build --no-cache -t falcon-mcp-custom .
   ```

2. **認証エラー**: 環境変数の設定確認
   ```bash
   docker-compose exec falcon-mcp env | grep FALCON
   ```

3. **プロキシエラー**: プロキシ設定の確認
   ```bash
   docker-compose logs falcon-mcp | grep -i proxy
   ```

4. **Content-Lengthヘッダー確認**: デバッグログの有効化
   ```bash
   # 環境変数にDEBUG=trueを追加
   ```

---

## まとめ

この手順書に従うことで、既存のFalcon MCPサーバのContent-Lengthヘッダー問題を解決し、安全にカスタマイズ版をデプロイできます。各段階でのテストを確実に実行し、問題が発生した場合は適切にロールバックできる体制を整えることが重要です。
