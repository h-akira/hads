# ローカル開発環境

HADSは効率的なローカル開発環境を提供し、AWS環境をシミュレートしながら快適に開発できます。このページでは、ローカル開発環境の詳細な設定と使用方法を説明します。

## 🏗️ ローカル開発の仕組み

### 3つのサーバー構成

HADSのローカル開発環境は3つのサーバーで構成されています：

```
┌─────────────────────────────────┐
│    プロキシサーバー (8000)      │
│    統合エンドポイント           │
└─────────────┬───────────────────┘
              │
    ┌─────────┴─────────┐
    │                   │
    ▼                   ▼
┌─────────────┐  ┌─────────────┐
│ SAM Local   │  │ 静的ファイル │
│ (3000)      │  │ サーバー     │
│ Lambda実行  │  │ (8080)      │
└─────────────┘  └─────────────┘
```

1. **SAM Local** (ポート3000): Lambda関数をローカルで実行
2. **静的ファイルサーバー** (ポート8080): CSS、JS、画像を配信
3. **プロキシサーバー** (ポート8000): 統合されたエンドポイントを提供

## 🚀 開発サーバーの起動

### 一括起動（推奨）

```bash
# プロキシサーバーを起動（他のサーバーも自動起動）
hads-admin.py admin.json --local-server-run proxy
```

### 個別起動

```bash
# ターミナル1: SAM Local
hads-admin.py admin.json --local-server-run sam

# ターミナル2: 静的ファイルサーバー
hads-admin.py admin.json --local-server-run static

# ターミナル3: プロキシサーバー
hads-admin.py admin.json --local-server-run proxy
```

## ⚙️ admin.json の詳細設定

### 基本設定

```json
{
  "region": "ap-northeast-1",
  "profile": "default",
  "static": {
    "local": "static",
    "s3": "s3://your-bucket-name/static/"
  },
  "local_server": {
    "port": {
      "static": 8080,
      "proxy": 8000,
      "sam": 3000
    }
  }
}
```

### 高度な設定

```json
{
  "region": "ap-northeast-1",
  "profile": "development",
  "static": {
    "local": "static",
    "s3": "s3://your-bucket-name/static/"
  },
  "local_server": {
    "port": {
      "static": 8080,
      "proxy": 8000,
      "sam": 3000
    },
    "host": "0.0.0.0",
    "debug": true,
    "auto_reload": true,
    "cors": {
      "enabled": true,
      "origins": ["http://localhost:3000", "http://127.0.0.1:3000"]
    }
  },
  "environment": {
    "AWS_SAM_LOCAL": "true",
    "DEBUG": "true",
    "LOG_LEVEL": "INFO"
  }
}
```

## 🔧 効率的な開発ワークフロー

### ホットリロード設定

```bash
# ファイル変更を監視してSAMを自動再起動
hads-admin.py admin.json --local-server-run sam --watch

# 静的ファイルの変更を監視
npm run watch  # package.jsonで設定
```

### 複数環境での開発

```bash
# 開発環境
cp admin.json admin-dev.json
hads-admin.py admin-dev.json --local-server-run proxy

# ステージング環境
cp admin.json admin-staging.json
# admin-staging.jsonを編集
hads-admin.py admin-staging.json --local-server-run proxy
```

## 🧪 テスト機能

### コマンドラインテスト

```bash
# GETリクエストのテスト
hads-admin.py admin.json --test-get /
hads-admin.py admin.json --test-get /api/users
hads-admin.py admin.json --test-get /blog/my-post

# POSTリクエストのテスト
hads-admin.py admin.json --test-get-event event.json
```

### テストイベントファイル

```json
# event.json
{
  "path": "/api/users",
  "requestContext": {
    "httpMethod": "POST"
  },
  "body": "name=John&email=john@example.com"
}
```

### ユニットテスト

```python
# tests/test_views.py
import sys
import os
import json
sys.path.append(os.path.join(os.path.dirname(__file__), '../Lambda'))

from lambda_function import lambda_handler

def test_index_view():
    """トップページのテスト"""
    event = {
        "path": "/",
        "requestContext": {
            "httpMethod": "GET"
        }
    }
    
    response = lambda_handler(event, None)
    
    assert response["statusCode"] == 200
    assert "HADSアプリ" in response["body"]

def test_api_endpoint():
    """APIエンドポイントのテスト"""
    event = {
        "path": "/api/data",
        "requestContext": {
            "httpMethod": "GET"
        }
    }
    
    response = lambda_handler(event, None)
    
    assert response["statusCode"] == 200
    data = json.loads(response["body"])
    assert "status" in data

def test_protected_view():
    """認証が必要なページのテスト"""
    event = {
        "path": "/profile",
        "requestContext": {
            "httpMethod": "GET"
        }
    }
    
    response = lambda_handler(event, None)
    
    # 未認証の場合はリダイレクト
    assert response["statusCode"] == 302
    assert "Location" in response["headers"]
```

## 🔍 デバッグとログ

### ログレベルの設定

```python
# Lambda/project/settings.py
import logging

if DEBUG:
    LOG_LEVEL = logging.DEBUG
else:
    LOG_LEVEL = logging.INFO

logging.basicConfig(
    level=LOG_LEVEL,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
```

### デバッグビューの作成

```python
# Lambda/project/debug_views.py
from hads.shortcuts import render, json_response

def debug_info(master):
    """デバッグ情報表示"""
    if not master.settings.DEBUG:
        return render(master, "404.html", code=404)
    
    debug_data = {
        "request": {
            "method": master.request.method,
            "path": master.request.path,
            "body": master.request.body,
            "auth": master.request.auth,
            "username": master.request.username
        },
        "environment": {
            "local": master.local,
            "debug": master.settings.DEBUG,
            "base_dir": master.settings.BASE_DIR
        },
        "system": {
            "python_version": sys.version,
            "aws_sam_local": os.getenv("AWS_SAM_LOCAL"),
            "aws_region": os.getenv("AWS_DEFAULT_REGION")
        }
    }
    
    return json_response(master, debug_data)

def debug_headers(master):
    """リクエストヘッダーの表示"""
    if not master.settings.DEBUG:
        return render(master, "404.html", code=404)
    
    headers = master.event.get("headers", {})
    return json_response(master, {"headers": headers})
```

### エラーページの改善

```python
# Lambda/project/views.py
def custom_error_render(master, error_message):
    """カスタムエラーページ"""
    if master.settings.DEBUG:
        # 開発環境では詳細なエラー情報を表示
        import traceback
        error_html = f"""
        <h1>🐛 デバッグ情報</h1>
        <h2>エラーメッセージ</h2>
        <pre>{error_message}</pre>
        <h2>リクエスト情報</h2>
        <pre>Path: {master.request.path}</pre>
        <pre>Method: {master.request.method}</pre>
        <pre>Body: {master.request.body}</pre>
        <h2>スタックトレース</h2>
        <pre>{traceback.format_exc()}</pre>
        """
        return {
            "statusCode": 500,
            "headers": {"Content-Type": "text/html; charset=UTF-8"},
            "body": error_html
        }
    else:
        # 本番環境では一般的なエラーページ
        return render(master, "500.html", code=500)
```

## 🗄️ データベース開発

### DynamoDB Local

```bash
# DynamoDB Localを起動
docker run -p 8000:8000 amazon/dynamodb-local

# テーブル作成
aws dynamodb create-table \
  --table-name users \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 \
  --endpoint-url http://localhost:8000
```

### データベース接続の設定

```python
# Lambda/project/database.py
import boto3
import os

def get_dynamodb_resource():
    """DynamoDBリソースを取得"""
    if os.getenv("AWS_SAM_LOCAL") == "true":
        # ローカル開発環境
        return boto3.resource(
            'dynamodb',
            endpoint_url='http://localhost:8000',
            region_name='ap-northeast-1'
        )
    else:
        # AWS環境
        return boto3.resource('dynamodb')

def get_users_table():
    """ユーザーテーブルを取得"""
    dynamodb = get_dynamodb_resource()
    return dynamodb.Table('users')
```

## 🔧 開発ツール連携

### VS Code設定

```json
// .vscode/settings.json
{
    "python.defaultInterpreterPath": "./venv/bin/python",
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": true,
    "python.linting.flake8Enabled": true,
    "python.formatting.provider": "black",
    "files.associations": {
        "*.yaml": "yaml",
        "admin*.json": "jsonc"
    },
    "yaml.schemas": {
        "https://raw.githubusercontent.com/aws/serverless-application-model/main/samtranslator/schema/schema.json": "*template.yaml"
    }
}
```

### launch.json（デバッグ設定）

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Lambda Function",
            "type": "python",
            "request": "launch",
            "program": "${workspaceFolder}/Lambda/lambda_function.py",
            "console": "integratedTerminal",
            "env": {
                "AWS_SAM_LOCAL": "true"
            },
            "args": []
        }
    ]
}
```

### tasks.json（タスク設定）

```json
// .vscode/tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Start Local Server",
            "type": "shell",
            "command": "hads-admin.py",
            "args": ["admin.json", "--local-server-run", "proxy"],
            "group": "build",
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "new"
            },
            "problemMatcher": []
        },
        {
            "label": "Run Tests",
            "type": "shell",
            "command": "python",
            "args": ["-m", "pytest", "tests/"],
            "group": "test",
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "new"
            }
        }
    ]
}
```

## 📊 パフォーマンス最適化

### 開発時のキャッシュ設定

```python
# Lambda/project/settings.py
if DEBUG:
    # 開発時はキャッシュを無効化
    CACHE_ENABLED = False
    TEMPLATE_CACHE = False
else:
    # 本番環境ではキャッシュを有効化
    CACHE_ENABLED = True
    TEMPLATE_CACHE = True
```

### 静的ファイルの最適化

```javascript
// package.json
{
  "scripts": {
    "dev": "npm run watch-css & npm run watch-js",
    "watch-css": "sass static/scss:static/css --watch --style expanded",
    "watch-js": "webpack --mode development --watch",
    "build": "npm run build-css && npm run build-js",
    "build-css": "sass static/scss:static/css --style compressed",
    "build-js": "webpack --mode production"
  }
}
```

## 🐛 トラブルシューティング

### よくある問題と解決方法

#### 1. ポートが既に使用されている

```bash
# ポートの使用状況を確認
lsof -i :3000
lsof -i :8000
lsof -i :8080

# プロセスを終了
kill -9 <PID>

# または別のポートを使用
# admin.jsonでポート番号を変更
```

#### 2. 静的ファイルが見つからない

```bash
# 静的ファイルディレクトリの確認
ls -la static/

# 権限の確認
chmod -R 755 static/

# プロキシサーバーの再起動
hads-admin.py admin.json --local-server-run proxy
```

#### 3. Lambda関数のインポートエラー

```python
# Lambda/lambda_function.py
import sys
import os

# パスの確認とデバッグ
print(f"Current working directory: {os.getcwd()}")
print(f"Python path: {sys.path}")

# プロジェクトディレクトリを追加
sys.path.append(os.path.dirname(__file__))
```

#### 4. 認証が動作しない

```bash
# 環境変数の確認
echo $AWS_SAM_LOCAL
echo $AWS_DEFAULT_REGION

# admin.jsonの確認
cat admin.json | jq '.'

# ログの確認
tail -f ~/.aws/sam/logs/sam-app.log
```

## 🔄 継続的開発

### Git フック

```bash
#!/bin/sh
# .git/hooks/pre-commit
# コミット前にテストを実行

echo "Running tests..."
python -m pytest tests/

if [ $? -ne 0 ]; then
    echo "Tests failed. Commit aborted."
    exit 1
fi

echo "Running linting..."
flake8 Lambda/

if [ $? -ne 0 ]; then
    echo "Linting failed. Commit aborted."
    exit 1
fi

echo "All checks passed. Committing..."
```

### Makefile

```makefile
# Makefile
.PHONY: dev test build deploy clean

dev:
	hads-admin.py admin.json --local-server-run proxy

test:
	python -m pytest tests/ -v

build:
	sam build

deploy:
	sam deploy --no-confirm-changeset

clean:
	rm -rf .aws-sam/
	find . -name "__pycache__" -exec rm -rf {} +
	find . -name "*.pyc" -delete

install:
	pip install -r requirements.txt
	npm install

lint:
	flake8 Lambda/
	black Lambda/ --check

format:
	black Lambda/
	isort Lambda/
```

## 📋 ベストプラクティス

### 1. 開発環境の標準化

```bash
# .env ファイルで環境変数を管理
# .env
AWS_SAM_LOCAL=true
DEBUG=true
LOG_LEVEL=DEBUG
DATABASE_URL=http://localhost:8000
```

### 2. コード品質の維持

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 22.3.0
    hooks:
      - id: black
  - repo: https://github.com/pycqa/flake8
    rev: 4.0.1
    hooks:
      - id: flake8
  - repo: https://github.com/pycqa/isort
    rev: 5.10.1
    hooks:
      - id: isort
```

### 3. ドキュメント化

```python
# Lambda/project/README.md
# プロジェクト固有のREADME

## 開発環境セットアップ
1. `python -m venv venv`
2. `source venv/bin/activate`
3. `pip install -r requirements.txt`
4. `make dev`

## テスト実行
`make test`

## デプロイ
`make deploy`
```

## 次のステップ

ローカル開発環境を理解したら、以下のページで本番環境へのデプロイについて学習してください：

- [デプロイメント](./deployment.md) - 本番環境への安全なデプロイ
- [ベストプラクティス](./best-practices.md) - 効率的な開発手法

---

[← 前: 認証とCognito連携](./authentication.md) | [ドキュメント目次に戻る](./README.md) | [次: デプロイメント →](./deployment.md)
