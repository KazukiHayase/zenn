---
title: "安く早く開発するための個人開発アーキテクチャ"
emoji: "🎉"
type: "tech"
topics:
  - "gcp"
  - "nextjs"
  - "terraform"
  - "個人開発"
  - "hasura"
published: true
published_at: "2022-12-27 21:14"
---

# はじめに

最近趣味で個人開発をしながらアーキテクチャの検討を行なっていたのですが、自分なりにいい感じの結論に辿り着いたので今回はそのアーキテクチャの紹介しようと思います！

インフラ、バックエンド、フロントエンドの各セクションに自分が使用しているテンプレートのリポジトリのリンクを載せてあるので興味のある方は参考にしてください。

また今回紹介するアーキテクチャはあくまで一例なので、間違いや不備などがあればご指摘いただければ幸いです。

# 前提条件

個人開発で使用するアーキテクチャを考える上で、自分の中でいくつか前提条件があります。

## ランニングコストを抑える

いくつか前提条件がある中で、個人的に一番重要な要素になります。

バズるサービスを作りたいという気持ちはありますが、そのためにいくらでもコストをかけられるかと言われるとそうではありません。むしろ個人開発となると、抑えられるコストはできる限り抑えたいというのが本音です。

また定額課金も可能であれば避けたいです。
1サービスあたり月に数千円が固定でかかると仮定した場合に、1サービスであればまだ許容できますが、それがサービス数分となると結構な金額になってしまいます。

そのため、ある程度の規模までは無料枠で運用でき、無料枠を超えてもアクセスに応じた従量課金という形が理想になります。

## 開発スピードが早い

開発期間が長すぎるとモチベも続かないですし、リリースしてこけた時のダメージが大きいと思うので、リリースまでのリードタイムは短くしたいです。ただ個人開発で実際に手を動かせるのは自分だけなので、実装をサボれるところはできる限りサボれるような構成が望ましいです。

## オートスケールする

個人開発をする上で少なからずバズって欲しいという気持ちがあるので、アクセスが増えたとしても自動でスケールしてくれるようにはしておきたいです。

現時点ではマネタイズに関してそこまで考えてはいませんが、広告などで何かしらアクセスに応じた収益を得られるようにはしたいと考えています。そうすることによってサービスだけがスケールしていき、赤字を垂れ流しながら運用をするといった事態はある程度避けられるのではないかなと思っています。

# 全体像

全体像は下記になります。それぞれ詳しく解説していきます。
![](https://storage.googleapis.com/zenn-user-upload/7794714196cd-20221227.png)

# インフラ

https://github.com/KazukiHayase/personal-project-infra-template

クラウドはGCPを使用して、リソースは全てTerraformで管理しています。Terraform Cloudが無料利用枠だと最大5ユーザーまでしか使用できないのですが、個人で使う分には特に気にする必要がないのでTerraform Cloudを使用しています。

https://www.hashicorp.com/products/terraform/pricing

ディレクトリ構造は下記のようになっています。

```
.
├── modules
│   ├── cloud_run.tf
│   ├── firebase.tf
│   ├── secret_manager.tf
│   ├── service_account.tf
│   └── variables.tf
├── main.tf
├── variables.tf
├── Makefile
└── versions.tf
```

同じリポジトリに対して違う環境変数を持ったWorkspacesをTerraform Cloud上に作成することで本番以外の環境を作成する方針にしています。`prod/`, `stg/`のようにディレクトリを分けてもよかったのですが、環境ごとに変わるリソースが特にないのと管理が面倒なのでこの方針にしました。

今回紹介する構成は認証を考慮してないのですが、もし使用するのであれば同じGCPのFirebase Authenticationか、もしくは使ったことのあるAuth0あたりを使おうかなと思っています。

またこの構成は使い回す想定なので、環境構築の手順はできるだけ簡略化するようにも心がけていて

1. GCPプロジェクトを作成
2. `make init`を実行
3. Terraform CloudからApply

で環境が作成できるようになっています！

`make init`はApplyに必要な諸々の事前準備を行うために用意しています。テンプレのMakefileをそのまま載せて置くので気になる方はご覧ください。

:::details Makefile

```makefile
GCP_PROJECT_ID := ""
REGISTRY_NAME := docker
HASURA_VERSION := v2.16.0

.PHONY: init
init:
	make enable_googleapis
	make create_service_account
	make create_artifact_registry
	make create_secret_manager
	make push_hasura_image

# 必要なAPIの有効化
.PHONY: enable_googleapis
enable_googleapis:
	gcloud services enable artifactregistry.googleapis.com
	gcloud services enable appengine.googleapis.com
	gcloud services enable iam.googleapis.com
	gcloud services enable run.googleapis.com
	gcloud services enable cloudresourcemanager.googleapis.com
	gcloud services enable firebase.googleapis.com
	gcloud services enable cloudbuild.googleapis.com
	gcloud services enable runtimeconfig.googleapis.com
	gcloud services enable cloudfunctions.googleapis.com
	gcloud services enable serviceusage.googleapis.com
	gcloud services enable secretmanager.googleapis.com

# Terraform用のサービスアカウントを作成
.PHONY: create_service_account
create_service_account:
	gcloud iam service-accounts create terraform --display-name="terraform"
	gcloud projects add-iam-policy-binding $(GCP_PROJECT_ID) \
		--member serviceAccount:"terraform@$(GCP_PROJECT_ID).iam.gserviceaccount.com" \
		--role "roles/owner" \
		--no-user-output-enable

# Hasuraのイメージを格納するartifact registryを作成
.PHONY: create_artifact_registry
create_artifact_registry:
	gcloud artifacts repositories create $(REGISTRY_NAME) --location=asia-northeast1 --repository-format=docker

# 手動で登録する用のシークレットマネージャーの作成
# NOTE: NEONでデータベースを作成してURLを手動でシークレットマネージャーに登録する
.PHONY: create_secret_manager
create_secret_manager:
	echo -n "" | gcloud secrets create HASURA_GRAPHQL_DATABASE_URL \
    --replication-policy="automatic" \
    --data-file=-

# Hasuraのイメージをartifact registryにpush
.PHONY: push_hasura_image
push_hasura_image:
	docker pull hasura/graphql-engine:$(HASURA_VERSION)
	docker tag hasura/graphql-engine:$(HASURA_VERSION) asia-northeast1-docker.pkg.dev/$(GCP_PROJECT_ID)/$(REGISTRY_NAME)/hasura:latest
	docker push asia-northeast1-docker.pkg.dev/$(GCP_PROJECT_ID)/$(REGISTRY_NAME)/hasura:latest
```

:::

# バックエンド

https://github.com/KazukiHayase/personal-project-hasura-template

## Cloud Run

バックエンドはHasuraをCloud Runに載せる形式を採用しました。
Cloud Runの採用理由は下記になります。

- 毎月の無料枠と従量課金の料金体系
- アプリケーション構築の手軽さ
- リクエストに応じでオートスケールする

特に料金体系に関しては**インスタンスの実行時間**に対する従量課金になっています。そのため設定で最小インスタンス数を0にすれば、アクセスがない場合はインスタンス数が0までスケールインし、その分のランニングコストを抑えることができます。
（後述しますが最小インスタンス数の設定はアプリケーションのパフォーマンスとトレードオフになります）

https://cloud.google.com/run/pricing?hl=ja#pricing_calculator

またHsuraにはHasura Cloudというフルマネージドなサービスもあるのですが、そちらは料金が$99/月/プロジェクトと金額も大きく、プロジェクト毎にかかってくるので不採用としました。

## Hasura

HasuraとはDBのスキーマからGraphQL APIを自動生成してくれるGraphQLサーバーです。APIの自動生成以外にもマイグレーションの管理や、テーブルのカラム単位での認可制御を行うことができます。

基本的なCRUDであれば自動生成してくれるAPIで賄えるので、バックエンドの実装工数を丸々なくすことができます！そのため実装が必要なのはフロントエンドのみになるので、開発スピードにおいてHasuraはかなり貢献してくれます。

一部複雑なドメインロジックや外部APIとの連携などは、Hasuraで対応できない場合があります。その場合はRemote Schemasという機能を使うことになります。Remote Schemasは、自前で用意したGraphQLサーバーをHasuraに取り込むことができる機能で、Hasuraだけでは賄えない機能をカバーすることができます。

ただ思っている以上にHasuraでできることは多いので、個人開発くらいの規模であれば全然Hasuraだけで対応できると思っています。ここに関しては発表スライドもあるので興味のある方はそちらもご覧ください。

@[speakerdeck](3c35731a92354c908237e8c5d83d1c33)

## Neon

NeonとはフルマネージドかつサーバーレスなPostgreSQLです。ブランチ機能もあり少し前に話題になっていた[PlanetScale](https://planetscale.com/)のPostgreSQL版というイメージが近いかなと思います。
Neonは執筆時点（2022/12/26）ではテクニカルプレビュー期間で無料で提供されており、2023年の1Qに有料プランの開始を計画しています。

Neonを採用した理由としては無料枠があるPostgreSQLという点です。Hasuraを使っている関係でDBはPostgreSQLを用意する必要があります。大体のRDBはアクセスがなくても最低でも毎月数千円かかってきますし、頼みの綱だったHerokuも無料枠を終了してしまいました。その中でフルマネージドかつ無料枠があるNeonを見つけ、プレビュー期間ではありますが個人開発で使う分には許容できると考え採用しました。

現時点で無料枠では下記が提供されています。
（GA後も無料枠は提供されるとのことですが、今後変更される可能性が大いにあるので参考程度に留めてください）

- 1プロジェクト
- 10ブランチ
    - 1ブランチごとに3GBのデータ容量
- 3エンドポイント
    - 1エンドポイントごとに1 vCPUと最大4GBのRAM

https://neon.tech/

# フロントエンド

https://github.com/KazukiHayase/personal-project-frontend-template

## Next.js

こちらはSSRが必要な場合の定番の選定かなと思います。
Next.js自体に強いこだわりがあるわけではなく、SSRができるかつ普段から使っていて慣れているという理由で採用しました。なのでプロダクトによってはRemixなど他のフレームワークも使ってみたいなと思っています。

## Firebase Hosting

デプロイ先はコスト面を考慮してFirebase Hostingを採用しました。

Next.jsのデプロイ先はVercelが有力候補としてありますが、Hobbyプランでは商用利用が禁止なのでProプラン以上を選択する必要があります。
VercelのProプランでは$20/月の定額課金で1TBまでのデータ転送料が含まれており、1TBを超過した分は$0.4/GBの従量課金になります。

Firebase Hostingでは10GBのストレージと0.36GB/日のデータ転送の無料枠があり、超過分はそれぞれ$0.026/GB、$0.15/GBの従量課金になります。

|  | Firebase Hosting | Vercel（Proプラン） |
| --- | --- | --- |
| 無料枠 | ストレージ：10GB<br>データ転送：0.36GB/日 | - |
| 定額課金 | - | $20/月<br>（1TBのデータ転送を含む） |
| ストレージ | $0.026/GB | - |
| データ転送 | $0.15/GB | $0.4/GB（1TB超過分） |

両者を比較するとデータ転送量に応じてコスト最適なデプロイ先が下記のようになります。
（Firebase HostingのストレージとSSRのためのCloud Functionの料金は無視しています）

| ~150GB/月 | 150~1500GB/月 | 1500~GB/月 |
| --- | --- | --- |
| Firebase Hosting | Vercel | Firebase Hosting |

上記を踏まえた上で、Vercelの場合は初期段階でも$20/月が固定でかかってしまうので、コスト面を優先して初期段階は完全に無料で始められるFirebase Hostingを採用しました。
コストの観点だけ見れば、最適なデプロイ先が変わるタイミングでデプロイ先を変えるのも手かなと思います。
フロントエンドのデプロイ先は最後まで悩んだ部分で、ISRなどのパフォーマンスの観点からVercelに変更することもありそうな気がしています。

https://firebase.google.com/pricing?hl=ja
https://vercel.com/pricing

# その他検討事項

## コールドスタートについて

上記の構成ではCloud Runの最小インスタンス数を0にしている関係で、コールドスタートが発生するのでパフォーマンス面に懸念があります。ここに関して最小インスタンス数を1にすることで回避はできます。ただその場合インスタンスが常にアイドル状態になるので、アクセスの有無に関わらず常に課金されてしまいます。

[リクエストの処理中にのみ CPU を割り当てる](https://cloud.google.com/run/docs/configuring/cpu-allocation?hl=ja)形式で、CPUが1でメモリが1GiBのインスタンスをアイドル状態で1秒間実行した場合の料金が$0.000005/秒になります。
そのため1ヶ月実行した場合は約$13/月となります。

メモリの設定などで金額も変わってきますがそれでも$10/月前後はかかってきます。
なのでこれを許容できる場合は最小インスタンス数を1にして、パフォーマスを向上させるのが良いと思います。

## DBについて

DBはNeonの他にCockroachDBも検討しました。ちょうど最近HasuraがCockroachDBに対応したこともありDBの第一候補として調査してました。ただまだリリースしたてということもありHasuraでマイグレーションが管理できなかったり、メタデータを保存するようにCockroachDBとは別でPostgreSQLを用意しないといけなかったりと制約が多かったので断念しました。

ただHasuraもCockroachDBも積極的に開発は行なっているようなので、今後上記のような制約がなくなるようであればCockroachDBへの移行も検討しています。

他にも何かいい感じのPostgreSQLのサービスがあれば教えていただけると幸いです！

https://www.cockroachlabs.com/
https://hasura.io/docs/latest/databases/postgres/cockroachdb/index/

# まとめ

個人的にかなりいい感じのアーキテクチャができたのではないかなと思っています！特に固定でかかる費用がなく、無料で始められる点がかなり気に入っています。

Hasuraに関しては業務でも1年以上使用しているのですが、かなりの実装工数が削減されるので個人開発においては最強の味方だと思います！興味のある方はぜひ使ってみてください！

DBやフロントエンドのデプロイ先などまだ検討の余地は全然あると思っているので、何か他にもいい方法があればご意見いただけると嬉しいです！