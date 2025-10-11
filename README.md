# Testing AWS infrastructure using LocalStack and Terraform Test

このパターンは、AWS 上でインフラをプロビジョニングすることなく、Terraform で IaC をローカルにテストするための解決策を提供します。Terraform 1.6 で導入された [Terraform Test framework](https://developer.hashicorp.com/terraform/language/tests)を用い、LocalStack との統合方法を示します。これにより、コスト最適化、速度と効率、一貫性と再現性、分離と安全性、そして簡素化された開発ワークフローを実現します。

LocalStack に対してテストを実行することで、実際の AWS サービスを使用する必要がなくなり、AWS におけるリソースの作成・変更・削除に伴うコストを回避できます。ローカルでのテストは、AWS にリソースをデプロイする場合と比べて大幅に高速です。

この迅速なフィードバックループにより、開発とデバッグが加速します。LocalStack はローカルで動作するため、インターネット接続なしでも Terraform スクリプトの開発とテストが可能です。LocalStack はテストのための一貫した環境を提供し、この一貫性により、外部の AWS の変更やネットワークの問題に左右されず、テスト結果が同じになることを保証します。

CI/CD パイプラインとの統合により、Terraform のスクリプトやモジュールを自動的にテストできます。これにより、デプロイ前にインフラコードが十分に検証されます。LocalStack を用いたテストは、本番の AWS リソースや環境に誤って影響を与えないことを保証します。この分離により、さまざまな構成を安全に試行・テストできます。開発者は即時のフィードバックを得ながらローカルで Terraform スクリプトをデバッグでき、開発プロセスが効率化されます。

本番環境により近づけるために、異なる AWS リージョン、アカウント、サービス構成をシミュレートできます。

## Prerequisites

- Docker Installed and configured to enable default Docker socket (/var/run/docker.sock).

- [Docker Installation Guide for Linux](https://docs.docker.com/engine/install/).

- [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/).

- [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/).

- Docker Compose [installed](https://docs.docker.com/compose/install/).

- AWS Command Line Interface (AWS CLI), [installed](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html).

- Terraform CLI, [installed](https://developer.hashicorp.com/terraform/cli) (Terraform documentation).

- Terraform AWS Provider, [configured](https://hashicorp.github.io/terraform-provider-aws/) (Terraform documentation).

## Target architecture

The code in this repository helps you set up the following target architecture.

![Architecture](docs/Architecture.png)

The diagram illustrates a CI/CD pipeline for a LocalStack Docker Container setup. Here's a breakdown of the components and their interactions:

**Source Code Repository**

1. ユーザーがソースコードリポジトリにコード変更をコミットします。

**CI/CD パイプライン**

2. コード変更がビルドプロセスをトリガーします。

3. ビルドプロセスは、コード変更が機能することを確認するためにテストもトリガーします。

**LocalStack Docker Container**

LocalStack Docker コンテナは、以下の AWS サービスをローカルでホストします：

4. ファイル保存用の Amazon S3 バケット。

5. 監視とログ記録用の Amazon CloudWatch。

6. サーバーレスコード実行用の AWS Lambda 関数。

7. マルチステップワークフロー調整用の AWS Step Function。

8. NoSQL データ保存用の Amazon DynamoDB。

**Workflow**

- ユーザーがソースコードリポジトリ (1) にコード変更をコミットします。

- CI/CD パイプラインが変更を検知し、静的な Terraform コード解析、LocalStack の Docker コンテナのビルド (2)、およびテストの実行 (3) のためにビルドプロセスをトリガーします。テストステージでは、AWS クラウドにリソースをデプロイすることなく、LocalStack を相手にインフラのテストを実行します（手順 3 ～ 8）。

LocalStack の Docker コンテナ内で、テストは次を行います:

- S3 バケットにオブジェクトをアップロードします（手順 4）。

- Amazon S3 のイベント通知を通じて AWS Lambda 関数を呼び出し（手順 4）、ログは Amazon CloudWatch に保存されます（手順 5）。

- その Lambda が状態遷移の実行を開始します（手順 6）。

- 実行は S3 オブジェクト名を DynamoDB テーブルに書き込みます（手順 7）。

- その後、アップロードしたオブジェクト名が DynamoDB テーブルのエントリと一致することを検証します（手順 8）。

提供されているテストには、指定した名前で S3 バケットがデプロイされていること、ならびに AWS Lambda 関数が正常にデプロイされていることを検証する例も含まれます。

LocalStack の Docker コンテナは、さまざまな AWS サービスをエミュレートするローカル開発環境を提供し、開発者は実際の AWS クラウドで費用を発生させることなくアプリケーションをテストおよび反復できます。

## Terraform Test

### Run Local Stack Container

クローンしたリポジトリで、bash シェルで以下のコマンドを入力して、LocalStack Docker をデタッチモードで実行開始します。

```shell
docker compose up -d
```

LocalStack コンテナが起動して実行されるまで待機します。

### Terraform Initialization

クローンしたリポジトリから以下のコマンドを入力して Terraform を初期化します。

```shell
terraform init
```

### Run Terraform Test

Terraform Test を実行するために以下のコマンドを入力します。

```shell
terraform test
```

すべてのテストが正常に通過したことを確認します。

出力は以下のようになります：

```shell
tests/localstack.tftest.hcl... in progress
  run "check_s3_bucket_name"... pass
  run "check_lambda_function"... pass
  run "check_name_of_filename_written_to_dynamodb"... pass
tests/localstack.tftest.hcl... tearing down
tests/localstack.tftest.hcl... pass

Success! 3 passed, 0 failed.
```

### Resource Cleanup

LocalStack コンテナを破棄するために以下のコマンドを入力します。

```shell
docker compose down
```

## Debugging with AWS CLI

### Run Local Stack Container

クローンしたリポジトリで、bash シェルで以下のコマンドを入力して、LocalStack Docker をデタッチモードで実行開始します。

```shell
docker-compose up -d
```

LocalStack コンテナが起動して実行されるまで待機します。

### Authentication

AWS クラウドをエミュレートするローカル実行コンテナ内で AWS CLI コマンドを実行できるように、以下の環境変数をエクスポートします。

```shell
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_SESSION_TOKEN=test
export AWS_REGION=eu-central-1
```

### Create Resources Locally

ローカルで実行中のコンテナ内でリソースを作成します。

```shell
terraform init
terraform plan
terraform apply -auto-approve
```

最後に、デプロイされたリソースに対して AWS CLI コマンドを実行できます。例えば、ステートマシンが作成されたことを確認することができます。

```shell
aws --endpoint-url http://localhost:4566 stepfunctions list-state-machines
```

### Destroy the resources

```shell
terraform destroy -auto-approve
```

LocalStack コンテナを破棄するために以下のコマンドを入力します。

```shell
docker compose down
```

## GitHub Actions

[GitHub Actions](.github/workflows/localstack-terraform-test.yml)を使用して CI/CD パイプラインで LocalStack と Terraform Test を統合する方法の例を提供しています。

## Authors

Pattern created by Ivan Girardi (AWS) and Ioannis Kalyvas (AWS).

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
