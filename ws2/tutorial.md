# Cloud Run ハンズオン(2/2) **Google Cloud の CI/CD パイプライン**

本ハンズオンは、以下のリポジトリを元に作成されています。
[cloud-run-tag-dev-example](https://github.com/tyorikan/cloud-run-tag-dev-example/tree/main)

## 目次
- [解説] ハンズオンの内容と目的
- アーキテクチャの概要
- Google Cloud プロジェクトの選択
- 環境の準備
  - シェル環境変数の設定
  - API の有効化
  - IAM の準備
- GitHub と Cloud Build の連携
  - Workload Idenitty 連携の準備 (GitHub Actions)
- Artifact Registry リポジトリの作成
- Cloud Build の準備
  - リポジトリの設定
  - トリガーの作成
- Cloud Run サービスの準備
- 開発の流れを体験
  - Main ブランチへの Pull Request と連動した開発用環境の起動
  - Main ブランチへの Merge と連動したデリバリーパイプラインの作成と本番環境へのロールアウト
  - 開発用ブランチの削除と連動した開発用環境の削除

## ハンズオンの内容と目的
アプリケーションを開発する際には、ソースリポジトリのメインブランチから機能毎にブランチを派生させ、そのブランチで開発をすることが多いでしょう。開発した機能を本番アプリケーションへ取り込む際にはメインブランチへのプルリクエストを作成することで、メンバーによる動作確認やコードレビューを経てメインブランチに統合される形で機能追加は実現します。
このフローにおいて、動作確認のために限られたメンバーだけがアクセスできる開発環境を用意することが多いでしょう。
また、新機能の取り込みが確定しても、本番運用中のアプリケーションの全ユーザーに新機能をいきなり開放することはリスクを伴うため、多くの場合はアクティブトラフィックを少しずつ現行バージョンから新バージョンへ段階的にロールアウトする、カナリアリリースを採用するチームが多いでしょう。

Cloud Build や Cloud Deploy を使うことで、Google Cloud のサーバーレス実行環境である Cloud Run での開発にあたり、上に述べたような環境の自動的な払い出しやカナリア的なアプローチによる新機能のロールアウトを実現することが出来ます。

このハンズオンでは、この流れを実際の開発業務に見立てて体験します。
ここでは、以下の Google Cloud コンポーネントを使用します。
- [Cloud Build](https://cloud.google.com/build/pricing?hl=ja)
- [Cloud Deploy](https://cloud.google.com/deploy/pricing?hl=ja)
- [Artifact Registry](https://cloud.google.com/artifact-registry/pricing?hl=ja)
- [Cloud Run](https://cloud.google.com/run/pricing?hl=ja)
- [Secret Manager](https://cloud.google.com/source-repositories/pricing?hl=ja)
- [Cloud Storage](https://cloud.google.com/storage/pricing?hl=ja)
* [料金計算ツール](https://cloud.google.com/products/calculator?hl=ja)を使うと、予想使用量に基づいて費用の見積もりを生成できます。

### ハンズオンの流れ
このハンズオンでは、GitHub リポジトリに対するアクションに連動して、次の3つの [Cloud Build トリガー](https://cloud.google.com/build/docs/triggers?hl=ja)が自動的に実行されます。
1. 新機能開発のために Main ブランチから開発用ブランチを派生させて開発し、Main ブランチにプルリクエストを作成したことに連動して、Cloud Run のタグ付き開発環境を起動するトリガー
2. プルリクエストを Main ブランチに取り込む(マージ)ことに連動して、Cloud Deploy でカナリアリリースのパイプラインを作成するトリガー
3. マージ後に開発用ブランチを削除したことに連動(GitHub Actions)して、開発環境を削除するトリガー


## Google Cloud プロジェクトの選択
このチュートリアルでは、様々な[サービス アカウント](https://cloud.google.com/iam/docs/service-account-overview?hl=ja)を作成するため、チュートリアル実施者がオーナー権限（または編集者権限 + IAM 管理者権限）を持つプロジェクトが必要となります。

**なるべく新しいプロジェクトを作成してください。**

<walkthrough-project-setup>
</walkthrough-project-setup>

### 選択されたプロジェクト名を確認する
```bash
gcloud config get project
```
正しく設定されていれば、以下のように表示されます。
```
Your active configuration is: [cloudshell-17515]
spider124-111
```
もし正しく設定されていない (unset 表示されている) 場合、以下のコマンドを実行して下さい。 **[YOUR_PROJECT_ID]** はご自身のプロジェクト ID に置き換えてください。
```bash
gcloud config set project [YOUR_PROJECT_ID]
```

## 環境の準備
### シェル環境変数の設定
プロジェクトで繰り返し使用する値をシェルの環境変数に設定します。 **GITHUB_ACCOUNT** は自身の GitHub アカウント名に置き換えてください。
**長時間の離席などにより Cloud Shell のセッションが切断された場合、必ずこの手順を再度実行して環境変数を設定し直して下さい。**
```bash
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects list --filter="$(gcloud config get-value project)" --format="value(PROJECT_NUMBER)")
export GITHUB_ACCOUNT=[YOUR_GITHUB_ACCOUNT]
```

## API の有効化
ハンズオンで必要になるサービスの API を有効化します。
<walkthrough-enable-apis apis="artifactregistry.googleapis.com,run.googleapis.com,cloudbuild.googleapis.com,clouddeploy.googleapis.com,compute.googleapis.com,iam.googleapis.com,iamcredentials.googleapis.com,cloudresourcemanager.googleapis.com,sts.googleapis.com,secretmanager.googleapis.com"></walkthrough-enable-apis>

### (Option) コマンドラインで有効化する場合
コマンドラインを使う場合、以下のコマンドを実行します。
```bash
gcloud services enable artifactregistry.googleapis.com run.googleapis.com cloudbuild.googleapis.com clouddeploy.googleapis.com compute.googleapis.com iam.googleapis.com iamcredentials.googleapis.com cloudresourcemanager.googleapis.com sts.googleapis.com secretmanager.googleapis.com
```

## IAM の準備
本ハンズオンで使用するサービス アカウントを作成します。
1. Cloud Build 用のサービスアカウントの作成
```bash
gcloud iam service-accounts create cloud-build-runner 
```
2. Cloud Run 用のサービスアカウントの作成
```bash
gcloud iam service-accounts create demo-backend-api
```

### Role の付与
前の手順で作成したサービス アカウントに、必要となる権限を割り当てます。これらの作業はコンソールからも実行可能です。
1. Cloud Deploy で利用するデフォルト サービス アカウントに **Cloud Deploy ランナー** ・ **Cloud Deploy リリース担当者** ・ **Cloud Run デベロッパー** ・ **サービス アカウント ユーザー** の権限を割り当てます。
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com --role=roles/clouddeploy.jobRunner
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com --role=roles/clouddeploy.releaser
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com --role=roles/run.developer
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com --role=roles/iam.serviceAccountUser
```
2. Cloud Build で利用するサービス アカウントに **Cloud Build サービス アカウント** ・ **Cloud Deploy オペレーター** ・  **Cloud Run 管理者** ・ **サービス アカウント ユーザー** の権限を割り当てます。
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/cloudbuild.builds.builder
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/clouddeploy.operator
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/run.admin
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/iam.serviceAccountUser
```

3. Cloud Build の[サービス エージェント](https://cloud.google.com/iam/docs/service-account-types?hl=ja#service-agents)に **Secret Manager 管理者** の権限を割り当てます。この権限は GitHub Actions が Cloud Build のトリガーを起動する際に使用します。
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:service-$PROJECT_NUMBER@gcp-sa-cloudbuild.iam.gserviceaccount.com --role=roles/secretmanager.admin
```

## GitHub の準備
1. GitHub 側で新しく空のリポジトリを作成し、リモートリポジトリとして設定します。
2. [clouddeploy.yaml](deploy/clouddeploy.yaml) 内のプロジェクト ID を修正してローカルリポジトリに Commit します。
```bash
sed -i -e "s#projects/cloud-run-deploy-demo#projects/${PROJECT_ID}#g" deploy/clouddeploy.yaml
```
3. ws2 以下のコンテンツ一式をリモートリポジトリの Main ブランチに Push します。
4. **[GitHub 側の設定]** Secret を設定します。
**Settings** -> **Secrets and variables** -> **Actions** を選択し、右ペインから **Variables** タブを選択します。
**Repository Variable** に下表の値を追加します。下表の"GCP_PROJECT_NUMBER" 及び "GCP_SA_ID" にはプロジェクト番号やプロジェクト名が自動的に入力されています。もし空欄になっている場合は手作業で追記して下さい。


| Name | Value | Note |
-------|--------|------ 
| CLOUD_BUILD_REGION | asia-northeast1 ||
| CLOUD_BUILD_TRIGGER_NAME | demo-backend-api-remove-cloud-run-tag ||
| GCP_PROJECT_NUMBER | <walkthrough-project-number/> |数字|
| GCP_SA_ID | cloud-build-runner@<walkthrough-project-id>.iam.gserviceaccount.com|"@"の後にプロジェクト ID が含まれているか|
| WORKLOAD_IDENTITY_POOL | github-actions-pool ||
| WORKLOAD_IDENTITY_PROVIDER | github-actions-provider ||

* 入力例

![](https://raw.githubusercontent.com/tpoppok/cloud-run-handson/main/ws2/images/gh-variables.png)

## Workload Idenitty 連携の準備
GitHub Actions で Cloud Build を呼び出すための GitHub Actions の設定を行います。
1. (**[Cloud Console](https://console.cloud.google.com) での操作**) **IAM** -> **Workload Identity 連携** へ移動し、プロバイダを追加します。下表の通り入力します。


設定項目 | 値
--------|------
ID プール名|github-actions-pool
プロバイダ | OIDC
プロバイダ名|github-actions-provider
発行元|https://token.actions.githubusercontent.com
オーディエンス|デフォルト
属性のマッピング ((Google*)=(OIDC*))|google.subject=assertion.sub, attribute.repository_owner=assertion.repository_owner

* 属性マッピングの設定例

![](https://raw.githubusercontent.com/tpoppok/cloud-run-handson/main/ws2/images/gh-attirbutemapping.png)

2. GitHub Actions から Cloud Build を呼び出すため、Cloud Build で利用するサービス アカウントに対し、Workload Identity ユーザーの権限を追加します。
```bash
gcloud iam service-accounts add-iam-policy-binding cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com \
    --role=roles/iam.workloadIdentityUser \
    --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-actions-pool/attribute.repository_owner/${GITHUB_ACCOUNT}"
```


## Artifact Registry リポジトリの作成
ビルドしたコンテナをホストする [Artifact Registry](https://cloud.google.com/artifact-registry?hl=ja) のリポジトリを作成します。
```bash
gcloud artifacts repositories create cloud-run-source-deploy \
    --repository-format=docker \
    --location=asia-northeast1
```

## Cloud Build の準備
GitHub 側での各操作に連動し、Cloud Build が予め定義したトリガーに従ってコンテナイメージのビルドや Cloud Deploy のパイプライン作成などを行います。

### リポジトリの接続 (Cloud Console からの作業)
#### ホスト接続の作成
1. `Cloud Build` -> `リポジトリ` を選択し、右ペインで `第 2 世代` をクリックします。
2. `ホスト接続を作成` をクリックします。
3. 左側のプロバイダ一覧の中から `GitHub` を選択します。
4. `リージョン` には **asia-northeast1**、名前は任意のものを入力し、接続を作成します。ポップアップが表示された場合は "continue" をクリックします。
5. `既存の GitHub インストールの使用` が表示されるので、 "インストール" から個人の GitHub アカウントを選択し、 `確認` をクリックします。

#### リポジトリの接続
1. 第 2 世代のリポジトリ一覧から、先の手順で作成した接続を見つけます。
2. 一番右のボタンをクリックし、 `リポジトリをリンク` をクリックします。
3. 以前の手順で作成したプライベートリポジトリにチェックを付け、"OK" をクリックします。
4. `リンク` をクリックします。 `リポジトリ名` はそのままで構いません。

![](https://raw.githubusercontent.com/tpoppok/cloud-run-handson/main/ws2/images/linkrepo.png)

#### 環境変数の設定
1. 作成したホスト接続のリソース名を取得し、環境変数に代入します。
```bash
export GITHUB_HOST=$(gcloud builds connections list --region=asia-northeast1|awk 'NR==1 {print $2}')
```

2. 作成したリポジトリのリソース名を取得し、環境変数に代入します。
```bash
export GITHUB_REPO=$(gcloud builds repositories list --region=asia-northeast1 --connection=$GITHUB_HOST|awk 'NR==1 {print $2}')
```

### Cloud Build トリガーの作成
GitHub リポジトリの各種状態遷移によって起動する Cloud Build トリガーを作成します。

1. demo-backend-api-pull-request
```bash
cat <<EOF > ./pr-trigger.yaml
description: Build and deploy to Cloud Run service demo-backend-api on pull request
filename: cloudbuild_pr.yaml
includeBuildLogs: INCLUDE_BUILD_LOGS_WITH_STATUS
name: demo-backend-api-pull-request
repositoryEventConfig:
  pullRequest:
    branch: .*
    commentControl: COMMENTS_ENABLED_FOR_EXTERNAL_CONTRIBUTORS_ONLY
  repository: projects/${PROJECT_ID}/locations/asia-northeast1/connections/${GITHUB_HOST}/repositories/${GITHUB_REPO}
  repositoryType: GITHUB
serviceAccount: projects/${PROJECT_ID}/serviceAccounts/cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com
EOF

gcloud builds triggers import --source=./pr-trigger.yaml --region asia-northeast1
```
2. demo-backend-api-push-main
```bash
cat <<EOF > ./main-trigger.yaml
description: Build and deploy to Cloud Run service demo-backend-api on push to main
filename: cloudbuild.yaml
includeBuildLogs: INCLUDE_BUILD_LOGS_WITH_STATUS
name: demo-backend-api-push-main
repositoryEventConfig:
  push:
    branch: ^main$
  repository: projects/${PROJECT_ID}/locations/asia-northeast1/connections/${GITHUB_HOST}/repositories/${GITHUB_REPO}
  repositoryType: GITHUB
serviceAccount: projects/${PROJECT_ID}/serviceAccounts/cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com
EOF

gcloud builds triggers import --source=./main-trigger.yaml --region asia-northeast1
```
3. demo-backend-api-remove-cloud-run-tag	
```bash
cat <<EOF > ./rm-tag-trigger.yaml
description: Remove Cloud Run Tags
gitFileSource:
  path: cloudbuild_rm_run_tag.yaml
  repository: projects/${PROJECT_ID}/locations/asia-northeast1/connections/${GITHUB_HOST}/repositories/${GITHUB_REPO}
  revision: refs/heads/main
name: demo-backend-api-remove-cloud-run-tag
serviceAccount: projects/${PROJECT_ID}/serviceAccounts/cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com
sourceToBuild:
  ref: refs/heads/main
  repository: projects/${PROJECT_ID}/locations/asia-northeast1/connections/${GITHUB_HOST}/repositories/${GITHUB_REPO}
EOF

gcloud builds triggers import --source=./rm-tag-trigger.yaml --region asia-northeast1
```

## Cloud Run サービスの準備
今回のハンズオンでは、既に Cloud Run 上のサービスは開発・運用中であり、そのサービスの追加開発を行うケースを想定しています。そのため、この手順で既存の Cloud Run サービスを立ち上げます。

サンプルコンテナを利用して以下2つのサービスを作成します。
1. demo-backend-api-dev	
2. demo-backend-api-prod
```bash
gcloud config set run/region asia-northeast1
gcloud config set run/platform managed

gcloud run deploy demo-backend-api-dev --image=us-docker.pkg.dev/cloudrun/container/hello --allow-unauthenticated --service-account=demo-backend-api@${PROJECT_ID}.iam.gserviceaccount.com
gcloud run deploy demo-backend-api-prod --image=us-docker.pkg.dev/cloudrun/container/hello --allow-unauthenticated --service-account=demo-backend-api@${PROJECT_ID}.iam.gserviceaccount.com
```

## 試してみる
1. GitHub で Pull Request (to main branch) を作成しましょう
2. main ブランチにマージしましょう
3. ブランチを削除しましょう
4. Cloud Deploy で dev to prod にプロモートしてみましょう
