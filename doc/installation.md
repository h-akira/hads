# インストールと初期設定

このページでは、HADSフレームワークをインストールして、開発環境を構築する手順を説明します。

## 前提条件

HADSを使用するために、以下のツールがインストールされている必要があります：

### 必須ツール

- **Python 3.9以上** - HADSフレームワーク本体
- **AWS CLI** - AWSリソースの管理
- **AWS SAM CLI** - サーバレスアプリケーションの開発・デプロイ
- **Git** - ソースコード管理

### 推奨ツール

- **Docker** - SAM Localでのローカル実行
- **VS Code** - Python開発に適したエディタ

## インストール手順

### 1. AWS CLI のインストール

```bash
# macOSの場合
brew install awscli

# Windowsの場合
# AWS公式サイトからインストーラーをダウンロード

# Linuxの場合
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### 2. AWS SAM CLI のインストール

```bash
# macOSの場合
brew install aws-sam-cli

# Windowsの場合
# AWS公式サイトからMSIインストーラーをダウンロード

# Linuxの場合
pip install aws-sam-cli
```

### 3. HADSフレームワークのインストール

```bash
pip install hads
```

> **注意**: 現在HADSはまだPyPIに公開されていません。リポジトリからのインストール方法は以下の通りです：

```bash
git clone https://github.com/h-akira/hads.git
cd hads
pip install -e .
```

## AWS認証設定

### AWS認証情報の設定

```bash
aws configure
```

プロンプトに従って以下を入力：
- **AWS Access Key ID**: AWSアクセスキー
- **AWS Secret Access Key**: AWSシークレットキー
- **Default region name**: `ap-northeast-1` (東京リージョン推奨)
- **Default output format**: `json`

### プロファイルの設定（複数環境の場合）

```bash
# 開発環境用プロファイル
aws configure --profile dev

# 本番環境用プロファイル
aws configure --profile prod
```

## 最初のプロジェクト作成

### 1. プロジェクトの初期化

```bash
hads-admin.py --init
```

対話的にプロジェクト設定を入力：

```
Enter project name (directory name): my-first-app
Enter suffix (to make resources unique, default is same as project name): my-first-app
Enter python version (default is 3.12): 3.12
Enter region (default is ap-northeast-1): ap-northeast-1
```

### 2. 生成されたファイル構造

```
my-first-app/
├── admin.json          # HADS設定ファイル
├── samconfig.toml      # SAM設定ファイル
├── template.yaml       # CloudFormationテンプレート
└── Lambda/
    ├── lambda_function.py     # メインハンドラー
    ├── project/
    │   ├── settings.py        # Django風設定ファイル
    │   └── urls.py           # URLルーティング
    └── templates/            # Jinja2テンプレート
```

### 3. admin.json の設定

生成された `admin.json` を環境に合わせて編集：

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

## 開発環境の確認

### 1. ローカルでの動作確認

```bash
cd my-first-app

# SAM Local でAPIサーバーを起動
hads-admin.py admin.json --local-server-run sam
```

### 2. ブラウザでアクセス

ブラウザで `http://localhost:3000` にアクセスして、正常に動作することを確認します。

### 3. テスト用エンドポイントの確認

```bash
# GETリクエストのテスト
hads-admin.py admin.json --test-get /
```

## 静的ファイルの設定

### 1. 静的ファイルディレクトリの作成

```bash
mkdir static
mkdir static/css
mkdir static/js
mkdir static/images
```

### 2. S3バケットの作成（本番環境用）

```bash
aws s3 mb s3://your-unique-bucket-name --region ap-northeast-1
```

### 3. 静的ファイルの同期

```bash
# S3に静的ファイルをアップロード
hads-admin.py admin.json --static-sync2s3
```

## エディタの設定

### VS Code の推奨拡張機能

- **Python** - Python開発サポート
- **AWS Toolkit** - AWS統合
- **YAML** - template.yamlの編集

### .vscode/settings.json

```json
{
    "python.defaultInterpreterPath": "./venv/bin/python",
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": true,
    "yaml.schemas": {
        "https://raw.githubusercontent.com/aws/serverless-application-model/main/samtranslator/schema/schema.json": "*template.yaml"
    }
}
```

## トラブルシューティング

### よくある問題と解決方法

#### 1. AWS認証エラー

```bash
# 認証情報を確認
aws sts get-caller-identity

# プロファイルを指定して確認
aws sts get-caller-identity --profile your-profile
```

#### 2. SAM CLIが見つからない

```bash
# SAM CLIのインストール確認
sam --version

# パスの確認
which sam
```

#### 3. Pythonバージョンの不整合

```bash
# 仮想環境の作成
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows

# HADSのインストール
pip install -e .
```

## 次のステップ

インストールが完了したら、以下のページに進んでください：

- [クイックスタート](./quickstart.md) - 簡単なアプリケーションの作成
- [プロジェクト構造](./project-structure.md) - プロジェクトの詳細な構造

---

[← ドキュメント目次に戻る](./README.md) | [次: クイックスタート →](./quickstart.md)
