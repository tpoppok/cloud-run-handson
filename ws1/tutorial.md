# Cloud Run ハンズオン (1/2)

## 環境準備

### Google Cloud プロジェクトの選択

ハンズオンを行う Google Cloud プロジェクトをまだ作成されていない場合は、[こちらのリンク](https://console.cloud.google.com/projectcreate) から新しいプロジェクトを作成してください。

**なるべく新しいプロジェクトが望ましいです。**

### 必要なロール

ハンズオンを進めるためには以下 **1** or **2** の何れかの IAM ロールが必要です。

1. [オーナー](https://cloud.google.com/iam/docs/understanding-roles#basic)
2. [編集者](https://cloud.google.com/iam/docs/understanding-roles#basic)、[Project IAM 管理者](https://cloud.google.com/iam/docs/understanding-roles#resourcemanager.projectIamAdmin)、[Cloud Datastore オーナー](https://cloud.google.com/iam/docs/understanding-roles#datastore.owner)、[Cloud Run 管理者](https://cloud.google.com/iam/docs/understanding-roles#run.admin)

それでは最初に、ハンズオンを進めるための環境準備を行います。

#### GCP のプロジェクト ID を環境変数に設定

環境変数 `PROJECT_ID` に GCP プロジェクト ID を設定します。[GOOGLE_CLOUD_PROJECT_ID] 部分にご使用になられる Google Cloud プロジェクトの ID を入力してください。
例: `export PROJECT_ID=myproject`

```bash
export PROJECT_ID=[GOOGLE_CLOUD_PROJECT_ID]
```

#### CLI（gcloud コマンド）から利用する GCP のデフォルトプロジェクトを設定

操作対象のプロジェクトを設定します。

```bash
gcloud config set project $PROJECT_ID
```

デフォルトのリージョンを設定します。

```bash
gcloud config set compute/region asia-northeast1
```

以下のコマンドで、現在の設定を確認できます。
```bash
gcloud config list
```

### ProTips
gcloud コマンドには、config 設定をまとめて切り替える方法があります。
アカウントやプロジェクト、デフォルトのリージョン、ゾーンの切り替えがまとめて切り替えられるので、おすすめの機能です。
```bash
gcloud config configurations list
```

## **参考: Cloud Shell の接続が途切れてしまったときは?**

一定時間非アクティブ状態になる、またはブラウザが固まってしまったなどで `Cloud Shell` が切れてしまう、またはブラウザのリロードが必要になる場合があります。その場合は以下の対応を行い、チュートリアルを再開してください。

### **1. チュートリアル資材があるディレクトリに移動する**

```bash
cd ~/cloudshell_open/gig-training-materials/gig08-01/
```

### **2. チュートリアルを開く**

```bash
teachme tutorial.md
```

### **3. gcloud のデフォルト設定**

```bash
source vars.sh
```

途中まで進めていたチュートリアルのページまで `Next` ボタンを押し、進めてください。

## [解説] ハンズオンの内容

### **概要**
このラボでは、Cloud Run の持つ高いスケーラビリティや、リクエストトラフィックの自由自在な制御を体験します。

次の図は、ラボの開始状態を示しています。 アーキテクチャは完全にサーバーレスです。 Cloud Firestore NoSQL データベースと相互作用するコンテナ化された Web サービスを Cloud Run にデプロイします。

![](image/overview-img.png?raw=true)

このアーキテクチャは、2つの Cloud Run サービスで構成されています。

Metrics writer

- メトリックを Cloud Firestore データベースに書き込むシンプルな「helloworld」スタイルのサービス。
- 各メトリックライターインスタンスは、1秒ごとにハートビートレコードを Cloud Firestore データベースに書き込みます。

> ハートビートレコードは、インスタンスがアクティブであるかどうか（要求を処理しているかどうか）、最後の1秒間に受信した要求の数、およびその他のメタデータを示します

Visualizer web app

- Cloud Run でホストされ、メトリックライターインスタンスによって永続化されたメトリックを読み取り、いい感じのグラフを表示するウェブアプリ。

### **目的**
このラボでは、次のタスクを実行します。

- コンテナ化されたサービスを Cloud Run にデプロイします。
- スケーリングの動作を確認するために Cloud Run に対して負荷を生成します。
- ネットワークトラフィックを操作するためのトラフィック分割ルールを構成します。

## 1. ローカル実行と Cloud Run へのデプロイ
このタスクでは、環境を設定し、最初のアーキテクチャをデプロイします。

- ビルド済みのコンテナイメージを使用して Cloud Run サービスを展開します。
- イメージが使用するプログラミング言語、Webフレームワーク、または依存関係は関係ありません。
- イメージは、標準化されたユニバーサルなフォーマットでパッケージ化されています。
- イメージは、変更することなく、さまざまなコンテナ実行環境に展開できます。

### 環境のセットアップ
1. `Cloud Shell` を開きます。

>Note: README の青い`OPEN IN GOOGLE CLOUD SHELL` ボタンから開始された場合は、すでにリポジトリはクローンされていますので、4 にスキップしてください。

2. このラボのスクリプトを含む git リポジトリをクローンします。 gcloud の承認を求められた場合は、承認してください。
```bash
git clone https://github.com/tpoppok/cloud-run-handson.git
```

3. リポジトリディレクトリに移動します。
```bash
cd ~/gig-training-materials/gig08-01
```

4. 必要な API を有効にします。
```bash
gcloud services enable run.googleapis.com \
  firestore.googleapis.com \
  compute.googleapis.com
```

5. スクリプトを実行して、プロジェクト ID とデフォルトリージョンのシェル変数を設定します。
```bash
source vars.sh
```

6. デフォルトで Cloud Run のマネージド環境を利用するよう、 `gcloud` コマンドで設定します。
```bash
gcloud config set run/platform managed
```

7. GCP のプロジェクト番号を環境変数に設定します。

```bash
export PROJECT_NUM=$(gcloud projects describe $PROJECT_ID --format json | jq -r '.projectNumber')
```

8. Cloud Run が使用するサービスアカウントに、プロジェクトの編集者 IAM ロールを設定します。

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --role roles/editor \
  --member "serviceAccount:$PROJECT_NUM-compute@developer.gserviceaccount.com"
```

9. Firestore データベースを作成します。
```bash
gcloud firestore databases create --location $REGION
```

### metrics-writer コンテナをローカルで実行する
ここでは、metrics-writer コンテナをローカルで実行します。公開されている Google Artifact Registry からコンテナイメージを取得します。コンテナイメージは実行可能であり、完全に自己完結型です。すべてがイメージにパッケージ化されているため、依存関係やランタイム環境をインストールする必要はありません。

<!-- Source = https://source.cloud.google.com/cnaw-workspace/cloudrun-visualizer/+/master:README.md -->
1. metrics-writer コンテナイメージをローカルの Cloud Shell インスタンスにダウンロードします。
```bash
docker pull asia-northeast1-docker.pkg.dev/gig6-1/gig6-1/metrics-writer:latest
```

2. イメージを実行します。プロジェクト ID に環境変数を設定し、ローカルポートをコンテナポートにマップします。
```bash
docker run \
  -e GOOGLE_CLOUD_PROJECT=${PROJECT_ID} \
  -p 8080:8080 \
  asia-northeast1-docker.pkg.dev/gig6-1/gig6-1/metrics-writer:latest
```

以下のような出力が表示されます。

**Output**
```terminal
> hello-world-metrics@0.0.1 start /usr/src/app
> functions-framework --target=helloMetrics --source ./src/

Serving function...
Function: helloMetrics
Signature type: http
URL: http://localhost:8080/
```

3. 新しい Cloud Shell タブを開きます。

4. 新しい Cloud Shell タブで、ローカルコンテナを呼び出します。
```bash
curl localhost:8080
```

以下のような出力が表示され、正常な応答を示しています。

**Output**
```terminal
Hello from blue
```

5. 最初の Cloud Shell タブに戻ります。メトリックスケジュールが開始していることが示されています。これらのログは無視してかまいません。

**Output**
```terminal
URL: http://localhost:8080/
initialising instance: 7229f512-6676-4211-90ca-80545c26aeb1
starting metrics schedule...
Metrics: id=7229f512, activeRequests=0,  requestsSinceLast=1
Metrics: id=7229f512, activeRequests=0,  requestsSinceLast=0
```

>Note: エラーが発生した場合は、しばらく待ってから、上記の手順（手順2から手順5）のコマンドを再実行してください。

6. control-c でローカル実行されているコンテナを停止します。

### 初期アーキテクチャのデプロイ

1. `metrics-writer` アプリを Cloud Run にデプロイします。Google Artifact Registry からのビルド済みコンテナイメージを使用します。
```bash
gcloud run deploy metrics-writer \
  --concurrency 1 \
  --allow-unauthenticated \
  --image asia-northeast1-docker.pkg.dev/gig6-1/gig6-1/metrics-writer:latest
```
以下のような出力が表示されます。

**Output**
```terminal
Deploying container to Cloud Run service [metrics-writer] in project [gig7-1] region [asia-northeast1]
OK Deploying new service... Done.
  OK Creating Revision... Revision deployment finished. Checking container health.
  OK Routing traffic...
  OK Setting IAM Policy...
Done.
Service [metrics-writer] revision [metrics-writer-00001-ras] has been deployed and is serving 100 percent of traffic.
Service URL: https://metrics-writer-rmclwajz3a-an.a.run.app
```

2. metrics-writer サービスの URL の値を使用してシェル変数を設定します。
```bash
export WRITER_URL=$(gcloud run services describe metrics-writer --format='value(status.url)')
```

3. metrics-writer サービスに接続できることを確認します。
```bash
curl $WRITER_URL
```

以下のような出力が表示されます。

**Output**
```terminal
Hello from blue
```

4. `visualizer` アプリを Cloud Run にデプロイします。ここでも、Google Artifact Registry から事前に作成されたコンテナイメージを使用します。
```bash
gcloud run deploy visualizer \
  --allow-unauthenticated \
  --max-instances 5 \
  --image asia-northeast1-docker.pkg.dev/gig6-1/gig6-1/visualizer:latest
```

5. visualizer サービスは Web アプリです。ローカルマシンで、Web ブラウザを開いてサービス URL にアクセスし、deploy コマンドの出力から URL 値をコピーします。

以下のような空のグラフが表示されます:

![](image/visualizer_graph.png?raw=true)

6. Cloud Run サービスを一覧表示します。metrics-writer と visualizer の2つのサービスが表示されます。
```bash
gcloud run services list
```

以下のような出力が表示されます。

**Output**
```terminal
✔
SERVICE: metrics-writer
REGION: asia-northeast1
URL: https://metrics-writer-rmclwajz3a-an.a.run.app
LAST DEPLOYED BY: admin@tomook.altostrat.com
LAST DEPLOYED AT: 2022-06-09T02:12:43.845980Z

✔
SERVICE: visualizer
REGION: asia-northeast1
URL: https://visualizer-rmclwajz3a-an.a.run.app
LAST DEPLOYED BY: admin@tomook.altostrat.com
LAST DEPLOYED AT: 2022-06-09T04:38:29.682058Z
```

7. [Cloud Run セクション](https://console.cloud.google.com/run) にアクセスして、サービスの内容を確認します。

## 2. スケールアウト対応

>_**クラウドネイティブの原則**: クラウドネイティブアプリはステートレスでディスポーザブルであり、高速の自動スケーリング用に設計されています。_

このモジュールでは、metrics-writer の Cloud Run サービスに対してトラフィックを生成して、自動スケーリングの動作を確認します。次に、サービスの構成を変更して、スケーリング動作への影響を確認します。

![](image/scale-out_img.png?raw=true)

### Cloud Run コンテナインスタンスの自動スケーリング

Cloud Runでは、アクティブな各 [リビジョン](https://cloud.google.com/run/docs/resource-model#revisions) は、着信要求を処理するために必要なコンテナインスタンスの数に自動的にスケーリングされます。詳細については、 [インスタンスの自動スケーリング](https://cloud.google.com/run/docs/about-instance-autoscaling) のドキュメントを参照してください。

作成されるインスタンスの数は、次の影響を受けます。
- 既存のインスタンスのCPU使用率（インスタンスを 60％ の CPU 使用率で提供し続けることを目標としています）
- [インスタンスあたりの同時リクエストの最大数（サービス）](https://cloud.google.com/run/docs/about-concurrency)
- [コンテナ インスタンス（サービス）の最大数](https://cloud.google.com/run/docs/configuring/max-instances)
- [最小インスタンス数（サービス）](https://cloud.google.com/run/docs/configuring/min-instances)

### リクエスト トラフィックを生成する

1. Cloud Shell を開きます。以前のシェルがしばらく非アクティブだった場合は、再接続が必要になる場合があります。その場合は、再接続後、リポディレクトリに移動し、環境変数を再設定します。

```bash
cd ~/cloudshell_open/cloud-run-handson/ws1/ && source vars.sh
```

2. Cloud Run サービスを一覧表示します。
```bash
gcloud run services list
```

3. まだ開いていない場合は、visualizer サービスの URL への Web ブラウザーページを1つ開きます。

4. [hey](https://github.com/rakyll/hey) コマンドラインユーティリティを使用して、サービスに 30 ワーカーで 30 秒間リクエストトラフィックを生成します。 `hey` ユーティリティはすでに Cloud Shell にインストールされています。
```bash
hey -z 30s -c 30 $WRITER_URL
```
オプションは、「30秒間(30s)に同時並列数 30 のリクエストを実行する」ということを意味しています。

5. visualizer Web アプリを表示するブラウザーページに切り替えます。ページにグラフがプロットされています。 Cloud Run は、トラフィック量を処理するためにアクティブなインスタンスの数を急速に拡大しました。

![](image/visualizer_graph_2.png?raw=true)

6. 30 秒が経過するまでグラフを監視します。 Cloud Run は、インスタンスがゼロになるまで急速にスケールダウンします。アクティブなインスタンスのピーク数を覚えておいてください。

7. cloud shell に戻ります。 `hey` ユーティリティは、負荷テストの要約を出力します。要約メトリックと応答時間のヒストグラムを見てください。

![](image/hey_summary.png?raw=true)

8. クラウドコンソールの [Cloud Run セクション](https://console.cloud.google.com/run) にアクセスします。`metrics-writer` サービスをクリックし、`指標` タブを選択します。

![](image/cloudrun_metrics_image.png?raw=true)

Cloud Run は、リクエスト数、リクエストレイテンシ、コンテナインスタンス数など、すぐに使用できる便利な[モニタリング指標]を提供していることがわかります。

9. 期間を「1時間」に変更し、「コンテナ インスタンス 数」グラフを確認します。 ピークの「アクティブな」インスタンス値は、ビジュアライザーグラフに表示された値とほぼ一致する必要があります。 グラフが更新されるまで約 3 分待つ必要があります。

>Note: クラウドコンソールの[指標]タブには、Cloud Run サービスに関する最も正確な情報が表示されます。 この情報は、Cloud Monitoring から取得されます。 ただし、コンソールのメトリックは更新に約 3 分かかります。 このラボでは、visualizer グラフを使用してリアルタイムのスケーリングを示します。 visualizer はデモ専用です。

### サービスのコンカレンシーをアップデート

Cloud Run は、特定のコンテナインスタンスで同時に処理できるリクエストの最大数を指定する [concurrency](https://cloud.google.com/run/docs/about-concurrency) 設定を提供します。

コードで並列リクエストを処理できない場合は、 `concurrency=1` を設定してください。図のように、各コンテナインスタンスは一度に 1 つのリクエストのみを処理します。

コンテナが複数のリクエストを同時に処理できる場合は、より高いコンカレンシーを設定します。指定されたコンカレンシー値は _maximum_ であり、インスタンスの CPU がすでに高度に使用されている場合、Cloud Run は特定のコンテナインスタンスに対してそれほど多くの要求を費やさない可能性があります。図では、サービスは最大 80 の同時要求（デフォルト）を処理するように構成されています。したがって、Cloud Run は、3 つのリクエストすべてを単一のコンテナインスタンスに送信します。

![](image/concurrency_image.png?raw=true)

`concurrency=1`の初期設定で metrics-writer サービスをデプロイしました。これは、各コンテナインスタンスが一度に 1 つのリクエストのみを処理することを意味します。この値を使用して、Cloud Run の高速自動スケーリングを示しました。ただし、このような単純なサービスでは、おそらくはるかに高い同時実行性を処理できます。ここでは、コンカレンシー設定を増やして、スケーリング動作への影響を調査します。

1. metrics-writer サービスのコンカレンシー設定を更新します。これにより、サービスの新しいリビジョンが作成されます。準備が整うと、すべてのリクエストがこの新しいリビジョンにルーティングされます。

```bash
gcloud run services update metrics-writer \
  --concurrency 5
```

2. コマンドを再実行して、負荷を生成します。
```bash
hey -z 30s -c 30 $WRITER_URL
```

3. visualizer Web アプリを表示するブラウザーページに戻ります。ページに別のグラフがプロットされています。

![](image/visualizer_graph_3.png?raw=true)

4. `hey` 出力の要約を確認します。

![](image/hey_summary_2.png?raw=true)

### サービスの最大インスタンス構成を更新する

ここでは、[コンテナ インスタンスの最大数](https://cloud.google.com/run/docs/about-instance-autoscaling#max-instances) 設定を使用して、リクエストに応じたサービスのスケーリングを制限します。 この設定は、コストを管理したり、データベースなどのバックエンドサービスへの接続数を制限したりする方法として使用します。

1. metrics-writer サービスの最大インスタンス設定を更新します。
```bash
gcloud run services update metrics-writer \
  --max-instances 5
```

2. コマンドを再実行して、負荷を生成します。
```bash
hey -z 30s -c 30 $WRITER_URL
```

3. visualizer Web アプリを表示するブラウザーページに戻ります。ページに別のグラフがプロットされています。

4. `hey` 出力の要約を確認します。以前の出力と比較してどうですか？

## 3. 軽快なトラフィック

>_**クラウドネイティブの原則**: クラウドネイティブアプリには、プログラム可能なネットワークデータプレーンがあります。_

このセクションでは、Cloud Run トラフィックの分割と ingress ルールを構成します。 このネットワーク動作は、シンプルな API 呼び出しを使用してプログラムします。

![](image/nimble-traffic_image.png?raw=true)

### タグ付きバージョンをデプロイする

名前付きタグを新しいリビジョンに割り当てることができます。これにより、サービストラフィックなしで、特定の URL でリビジョンにアクセスできます。次に、そのタグを使用して、トラフィックをタグ付きリビジョンに徐々に移行し、タグ付きリビジョンをロールバックできます。この機能の一般的な使用例は、トラフィックを処理する前に、新しいサービスリビジョンのテストと検証に使用することです。

1. Cloud Shell を開きます。以前のシェルがしばらく非アクティブだった場合は、再接続が必要になる場合があります。その場合は、再接続後、repo ディレクトリに移動し、環境変数を再設定します。
```bash
cd ~/cloudshell_open/cloud-run-handson/ws1/ && source vars.sh
```

2. metrics-writer サービスの新しいリビジョンをデプロイし、コンカレンシーと最大インスタンス値を既知の値に戻します。
```bash
gcloud run services update metrics-writer \
  --concurrency 5 \
  --max-instances 7
```

3. metrics-writer サービスの新しいリビジョンをデプロイします。 'green' というタグを指定します。 `--no-traffic` フラグを設定します。これは、トラフィックが新しいリビジョンにルーティングされないことを意味します。表示されるグラフの色を制御する LABEL 環境変数を設定します（環境変数はタグとはまったく関係がないことに注意してください）。
```bash
gcloud run deploy metrics-writer \
  --tag green \
  --no-traffic \
  --set-env-vars LABEL=green \
  --image asia-northeast1-docker.pkg.dev/gig6-1/gig6-1/metrics-writer:latest
```

以下のような出力が表示されます。リビジョンはトラフィックの 0％ を処理しており、タグ名の前に専用 URL が付いていることに注意してください。

**Output**
```terminal
Deploying container to Cloud Run service [metrics-writer] in project [gig7-1] region [asia-northeast1]
OK Deploying... Done.
  OK Creating Revision...
  OK Routing traffic...
Done.
Service [metrics-writer] revision [metrics-writer-00005-don] has been deployed and is serving 0 percent of traffic.
The revision can be reached directly at https://green---metrics-writer-rmclwajz3a-an.a.run.app
```

4. 前のコマンドは、新しい 'green' リビジョン専用のタグ付き URL を出力します。 [TAGGED_URL] をコマンド出力の値に置き換えて、シェル変数を設定します。
```bash
export GREEN_URL=[TAGGED_URL]
```

5. サービスリビジョンを一覧表示します。 2 つのアクティブなリビジョンがあることがわかります。
```bash
gcloud run revisions list --service metrics-writer
```

6. サービスへのリクエストを実行します。コマンドを数回実行すると、サービスが常に 'blue' を返すことがわかります。'green' サービスは、プライマリサービス URL からのトラフィックを処理していません。
```bash
curl $WRITER_URL
```

7. 新しいタグ付き URL に対してリクエストを実行します。サービスが 'green' を返すことがわかります。メインリビジョンは引き続きすべてのライブトラフィックに対応していますが、テスト可能な専用 URL が付いたタグ付きバージョンがあります。
```bash
curl $GREEN_URL
```

**Output**
```terminal
Hello from green
```

### トラフィックスプリッティングの構成

Cloud Run を使用すると、トラフィックを受信するリビジョンまたはタグを指定したり、リビジョンによって受信されるトラフィックの割合を指定したりできます。この機能を使用すると、以前のリビジョンにロールバックし、リビジョンを段階的にロールアウトして、トラフィックを複数のリビジョンに分割できます。

1. トラフィック分割を構成し、トラフィックの 10％　を 'green' とタグ付けされたリビジョンに送信します。
```bash
gcloud run services update-traffic \
  metrics-writer --to-tags green=10
```

以下のような出力が表示されます。出力には、現在のトラフィック構成が記述されています。

**Output**
```terminal
OK Updating traffic... Done.
  OK Routing traffic...
Done.
URL: https://metrics-writer-rmclwajz3a-an.a.run.app
Traffic:
  90% metrics-writer-00004-wof
  10% metrics-writer-00005-don
        green: https://green---metrics-writer-rmclwajz3a-an.a.run.app
```

2. metrics-writer サービスに対してリクエストロードを生成します。メインサービスの URL が表示されます。
```bash
hey -z 30s -c 30 $WRITER_URL
```

3. visualizer Web アプリを表示するブラウザーページに戻ります。ページにグラフがプロットされています。グラフには、緑と青の2本の線があります。 'green' サービスはトラフィックの約 10％ を受信して​​います。

![](image/visualizer_graph_4.png?raw=true)

4. 別のトラフィック分割を構成し、トラフィックの 50％ を 'green' とタグ付けされたリビジョンに送信します。
```bash
gcloud run services update-traffic \
  metrics-writer --to-tags green=50
```

5. metrics-writer サービスに対してリクエストロードを生成します。
```bash
hey -z 30s -c 30 $WRITER_URL
```

6. visualizer Web アプリを表示するブラウザーページに戻ります。今回は、トラフィックが 'green' と 'blue' のリビジョン間で均等に分割されます。



## おめでとうございます!

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

これでこのラボは完了です。
- Google Cloud のサーバーレスコンテナプラットフォームである Cloud Run を使用して、いくつかの主要なクラウドネイティブの原則を実装しました。
- コンテナ化された Web サービスを展開し、高速自動スケーリングをトリガーし、ネットワークトラフィックを操作し、組み込みのセキュリティ制御を適用しました。
- Cloud Run を使用して、クラウドネイティブアプリケーションのモダナイゼーションを加速できることを学びました。

デモで使った資材が不要な方は、次の手順でクリーンアップを行って下さい。

## **クリーンアップ（プロジェクトを削除）**

ハンズオン用に利用したプロジェクトを削除し、コストがかからないようにします。

### **1. Google Cloud のデフォルトプロジェクト設定の削除**

```bash
gcloud config unset project
```

### **2. プロジェクトの削除**

```bash
gcloud projects delete $PROJECT_ID
```

### **3. ハンズオン資材の削除**

```bash
cd $HOME && rm -rf ./cloudshell_open/cloud-run-handson
```