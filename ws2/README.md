# Deploy your application to Cloud Run using Google Cloud's CI/CD pipeline
Cloud Run で Pull Request 毎の環境を払い出すデモ
1. 開発中の main ブランチに対して Pull Request が作成されると開発用の Cloud Run 環境を作成する。開発環境はタグ付きリビジョン固有 URL が払い出され、サービスエンドポイントに対するリクエストトラフィックはルーティングされないため、リビジョン固有 URL を知る人にしかアクセスが出来ない。
2. Pull Request が Merge されると、ステージング用の環境が作成され、本番用の環境にカナリアリリースが可能となる。
3. 開発用のブランチが削除されるとタグが削除される。

## Launch API
```
go run cmd/api/main.go
```

## Cloud Build pipelines
1. [cloudbuild_pr.yaml](cloudbuild_pr.yaml)（no-traffic で Cloud Run デプロイ & タグ発行）  
PR が作成されたら実行されるパイプライン

2. [cloudbuild.yaml](cloudbuild.yaml)  
main ブランチに merge (or push) されたら実行されるパイプライン（Cloud Deploy 経由で Cloud Run へデリバリーパイプラインを作成）

3. [cloudbuild_rm_run_tag.yaml](cloudbuild_rm_run_tag.yaml)
Branch が削除されたら実行（GitHub Actions から Cloud Build を呼び出し、タグを削除）


## Setup
[チュートリアル](tutorial.md)を参照

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.png)](https://ssh.cloud.google.com/cloudshell/open?cloudshell_git_repo=https://github.com/tpoppok/cloud-run-handson&cloudshell_working_dir=ws2&cloudshell_tutorial=tutorial.md&shellonly=true)

**This is not an officially supported Google product**. This directory contains some scripts that are used to teach Google Cloud beginners how to use Cloud Run in more efficient way.

see [tutorial.md](tutorial.md) for more details
