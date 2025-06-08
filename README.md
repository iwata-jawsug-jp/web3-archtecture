# CloudFront VPC Origin対応 Terraform Web3層アーキテクチャ構築プロンプト

## 概要
terraformでWeb3階層のアプリケーション環境を構築する。CloudFront VPC Origin機能を活用して、セキュアなアーキテクチャを実現する。

## ⚠️ 重要な前提条件

### CloudFront VPC Origin機能について
- **発表日**: 2024年11月20日に発表された新機能
- **現在のステータス**: 
  - AWS Console/CLIで利用可能
  - **Terraformサポート**: <!--未対応（[GitHub Issue #40234](https://github.com/hashicorp/terraform-provider-aws/issues/40234)で対応中）-->
    - CloudFrontのVPC Origin機能はTerraformで利用可能
      - TerraformのAWSプロバイダ v5.82.0 以降で aws_cloudfront_vpc_origin リソースが追加され、VPCオリジンの設定ができるようになっています
      - VPCオリジンの作成: ALBを指定してVPCオリジンを作成し、CloudFrontディストリビューションのオリジンとして設定可能
      - セキュリティグループの設定: CloudFrontのマネージドプレフィックスリストを利用して、CloudFrontからのアクセスを許可する必要がある
      - Terraformのモジュール対応: terraform-aws-modules/cloudfront/aws の v4.0.0 以降でVPCオリジンの設定が可能
  - **CloudFormationサポート**: 近日対応予定
- **対応方針**: 初期段階では代替実装を使用し、Terraformサポート後に移行

## 構築条件

### 基本設定
- 各リソース名は`name_prefix`で作成する
- リージョン: ap-northeast-1（東京）
- VPCのサブネットは`public`、`protected`、`private`の3階層
- モジュール:
  - 各モジュールにREADME.mdを用意し、概要を説明する
  - 汎用的に使用できるようにパラメーター設定する
  - AWSリソース毎に作成する
    - 設定例：
    ```
    ├── modules               # Terraformモジュール
    │   ├── alb
    │   ├── cloudfront
    │   ├── database
    |   |   ├── aurora
    |   |   └── rds
    │   ├── ecs
    │   ├── ecr
    │   ├── acm
    │   └── initial-config
    ```

### ネットワーク構成
- **public subnet**: Internet Gateway経由でインターネットアクセス
- **protected subnet**: NAT Gateway経由でアウトバウンド通信のみ（internal ALB配置先）
- **private subnet**: インターネットアクセスなし（データベース配置先）

### CloudFront配信設定
- **Origin 1**: S3バケット（SPA用、パス: `/`）
- **Origin 2**: VPC Origin（API用、パス: `/api`、ターゲット: internal ALB）

### CloudFront VPC Origin要件
- **制約事項**: 
  - VPCは同一AWSアカウント内に存在する必要がある
  - VPCには Internet Gateway が必要（トラフィック経路ではなく、VPCのインターネット接続を示すため）
  - protectedサブネットは private subnet として設定（最低1つのIPv4アドレスが利用可能）
  - Lambda@Edge との互換性なし（CloudFront Functions は利用可能）
- **セキュリティ**: CloudFrontからのアクセスのみ許可（従来の prefix list や custom header 設定が不要）
- **コスト**: 追加料金なし

### internal ALB設定
- **配置**: protectedサブネット（private subnet）の各AZ
- **タイプ**: internal（internet-facing ではない）
- **セキュリティグループ**: CloudFront VPC Originからのアクセスのみ許可
- **SSL終端**: ALBでSSL/TLS終端実施（ACM証明書使用）
- **ターゲット**: 同一protectedサブネット内のECS Fargateタスク

### データベース設定
- **選択肢**: RDS for PostgreSQL または Aurora PostgreSQL
- **モジュール化**: 
  - databaseモジュールを作成
  - 選択可能な形でモジュール設計
  - RDSとAuroraはモジュール化し、databaseモジュールから呼び出す
- **パスワード管理**: Secrets Managerを使用（パスワードローテーションの更新間隔を365に変更）
- **データベースエンジン**: 15.10をデフォルトにする
- **データベースの初期化**: 
  - S3に配置したCSV（DML）およびSQL（DDL）で初期設定
  - Lambda関数で初期設定を実装

### ECS設定
- **配置**: protectedサブネット（private subnet）の各AZ
- **モジュール化**: clouster,task,serviceで分割
- **タスクパラメータ**: secret managerを使用する

### SSL証明書設定
- **CloudFront用**: ACM証明書（us-east-1リージョンで作成）
- **ALB用**: ACM証明書（ap-northeast-1リージョンで作成）
- **Route53**: 既存のホストゾーンから設定を取得

### backend設定
- tfstate保存用S3バケットは事前に手動作成または専用bootstrapで作成
- S3バケット名にAWSアカウントIDを追記する
- Terraform 1.11以降はS3バックエンド自体でロック管理が可能
- backend.hclは、以下のように設定
  ```hcl
  # backend.hcl例
  bucket = "your-tfstate-bucket"
  key = "base/terraform.tfstate"  # ステップごとに異なるkey
  region = "ap-northeast-1"
  encrypt = true
  use_lockfile = true
  ```
- backend.hclのbucketは、シェルスクリプトで一括修正する

## ステップ構成

### 1. base（基盤インフラ）
- VPC, subnet（public/protected/private）
- Internet Gateway, NAT Gateway
- Route Table

### 2. security（セキュリティグループ）
- ALB用セキュリティグループ
- ECS用セキュリティグループ  
- RDS用セキュリティグループ

### 3. database（データベース）
- RDS OR Aurora（選択式）
- Secrets Manager
- データベース初期化設定

### 4. backend（バックエンドアプリケーション）
- internal ALB
- ECS cluster
- ECS service
- ECS task definition

### 5. frontend（フロントエンド）
- S3（静的サイトホスティング）
- CloudFront distribution
- WAF
- ACM証明書（us-east-1）

### 6. shared（共有リソース）
- ECR repository
- Route53設定

### 7. initial-config（初期設定）
- データベース初期化スクリプト実行
- アプリケーション動作確認

## 実行順序

1. **infrastructure** (base + security + shared + Route53)
2. **certificates** (ACM証明書作成・検証)
3. **database** (RDS/Aurora + Secrets Manager)
4. **backend-foundation** (ALB + ECS cluster)
5. **container-setup** (ECRイメージpush + ECS task definition)
6. **backend-application** (ECS service)
7. **frontend** (S3 + CloudFront + WAF)
8. **initial-config** (DB初期化 + アプリケーション動作確認)

## 注意事項

### セキュリティ考慮事項
- VPC Origin使用時はCloudFrontが唯一のエントリーポイントとなる
- WAF設定により追加の保護レイヤーを提供
- 従来のPrefix ListやCustom Header設定は不要

### 運用考慮事項
- CloudFront VPC Originのデプロイには最大15分要する
- IPv6専用サブネットはサポートされない
- ネットワークACLは適切に設定する必要がある

### コスト最適化
- VPC Origin機能自体に追加料金なし
- NAT Gatewayのコスト削減効果
- データ転送コストの最適化

### 移行戦略
- 段階的な移行アプローチを推奨
- CloudFront Continuous Deploymentを活用したテスト
