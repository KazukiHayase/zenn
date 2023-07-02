---
title: "レビュー依頼されているすべてのPRブラウザで開く GitHub CLI の拡張機能を Go 作った"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

自分は普段、1 日平均 20 個程の PR レビューをしており、基本的にはレビュー依頼が来たら即レビューをしています。しかし、MTG の最中などにレビュー依頼が来た場合は、即レビューができずどうしてもレビュー依頼が溜まってしまいます。レビュー依頼が溜まると、レビュー漏れが発生しやすくなったり、全ての PR を一々ブラウザで開くのが面倒臭く感じてました。

そこで今回は、レビュー依頼されている PR をすべてブラウザで開く GitHub CLI の拡張機能を作成したので、その紹介をしたいと思います！

作成した拡張機能のリポジトリは下記になります。
https://github.com/KazukiHayase/gh-requested-prs

# デモ

![](/images/ee9d5010bb8ffd/demo.gif)

# 実装の紹介

## セットアップ

下記のページに従って拡張機能を作成して行きます。
セットアップの方法に関しては、公式や他の記事でも紹介されているので最低限に留めておきます。

https://docs.github.com/ja/github-cli/github-cli/creating-github-cli-extensions

`create`コマンドを実行することで拡張機能の作成に必要なファイル群が生成されます。
言語は普段使い慣れている Go を選定したので、`main.go`に処理を記載していきます。

```bash
$ gh extension create --precompiled=go gh-requested-prs
✓ Created directory gh-requested-prs
✓ Initialized git repository
✓ Made initial commit
✓ Set up extension scaffolding
✓ Downloaded Go dependencies
✓ Built gh-requested-prs binary

gh-requested-prs is ready for development!

Next Steps
- run 'cd gh-requested-prs; gh extension install .; gh requested-prs' to see your new extension in action
- run 'go build && gh requested-prs' to see changes in your code as you develop
- run 'gh repo create' to share your extension with others

For more information on writing extensions:
https://docs.github.com/github-cli/github-cli/creating-github-cli-extensions
```

下記のコマンドでローカルの拡張機能をインストールできるので、インストール後に動作確認をしながら実装を進めて行きます。

```bash
$ gh extension install .
```

拡張機能をリリースするための GitHub Actions も同時に生成されているので、その GitHub Actions を実行することで簡単にリリースができます。
リリース後は下記コマンドで誰でも拡張機能を使用することができます。

```shell
$ gh extension install KazukiHayase/gh-requested-prs
```

また、GitHub が拡張機能を作成する際に便利なパッケージを提供してくれているので、諸々のデータの取得にはそのモジュールを使用します。
https://github.com/cli/go-gh

## Organization の取得

レビュー依頼されている PR を取得するために、Organization を指定するのであらかじめ取得しておきます。
Organization は GitHub CLI のコマンドで取得できるので、`go-gh`の`Exec`関数を使用して、Go のコードから GitHub CLI のコマンドを実行します。
やりたいことが GitHub CLI のコマンドで事足りる場合は、実装も減るので積極的に活用するのがいいかなと思います。

```go:main.go
func main() {
	orgList, _, err := gh.Exec("org", "list")
	if err != nil {
		log.Fatal(err)
	}

	orgs := strings.Split(orgList.String(), "\n")
}
```

## レビュー依頼されている PR の取得

肝心のレビュー依頼されている PR の取得部分ですが、GitHub CLI の`gh pr list`コマンドでは、単一のリポジトリの PR しか取得できないので、こちらは API リクエストで取得することにしました。
API リクエストの際の機能も REST と GraphQL の両方とも`go-gh`が提供してくれています。こちらも自分が慣れている GraphQL を選択しました。

GraphQL API のリクエストは下記の流れで行います。

1. Client の初期化
1. GraphQL オペレーションの構造体定義
1. Variables の map 定義
1. Client の Query メソッドを実行

実際のコードは下記のようになります。

```go
func main() {
	// 1. Client の初期化
	client, err := api.DefaultGraphQLClient()
	if err != nil {
		log.Fatal(err)
	}

	// 2. GraphQL オペレーションの構造体定義
	var query struct {
		Search struct {
			Edges []struct {
				Node struct {
					PullRequest struct {
						URL string
					} `graphql:"... on PullRequest"`
				}
			}
		} `graphql:"search(first: 100, query: $q, type: ISSUE)"`
	}

	filters := []string{
		"is:pr",
		"is:open",
		"review-requested:@me",
	}

	// 省略... Organization の取得 + filters へ条件追加の処理

	// 3. Variables の map 定義
	variables := map[string]any{
		"q": graphql.String(strings.Join(filters, " ")),
	}

	// 4. Client の Query メソッドを実行
	if err := client.Query("ReviewRequestedPullRequests", &query, variables); err != nil {
		log.Fatal(err)
	}
}
```

## PR のリンクをブラウザで開く

API リクエストの結果から PR の URL のスライスを生成します。
その URL スライスに対して、open コマンドを実行して、全ての PR をブラウザ開くようにしています。

```go
func main() {
	// ...省略

	var urls []string
	for _, edge := range query.Search.Edges {
		urls = append(urls, edge.Node.PullRequest.URL)
	}

	if len(urls) == 0 {
		fmt.Println("no review requested pull requests")
		return
	}

	fmt.Println("open pull requests...")
	for _, url := range urls {
		fmt.Println(url)
		err := exec.Command("open", url).Run()
		if err != nil {
			log.Fatal(err)
		}
	}
}
```

# 実装のポイント

## API リクエストの認証

`go-gh`で API リクエストをする際の認証は GitHub CLI と同じ仕組みを利用すると[README](https://github.com/cli/go-gh#go-library-for-the-github-cli)に記載があります。

> GitHub API requests will be authenticated using the same mechanism as gh, i.e. using the values of GH_TOKEN and GH_HOST environment variables and falling back to the user's stored OAuth token.

そのため拡張機能自体に認証情報を持つ必要がないです。
また、パッケージの内部でトークンを取得して、API リクエストのヘッダーに挿入してくれるので、自分で認証周りを実装する必要ない点が実装をする上でかなり楽だったなと感じました。

## GraphQL オペレーションの実行

GraphQL オペレーションの実行のために、`Query`メソッドの第二引数に渡す構造体には、「API レスポンスの Unmarshal」と「GraphQL オペレーションそのものの定義」の 2 つの役割があります。

1 つ目の役割はそのままで、Go ではよく見かける実装だと思います。
メソッドの内部で API レスポンスを構造体に変換して、メソッド実行後の処理で API レスポンスを構造体として扱うことができます。

2 つ目の役割は、`go-gh` が内部で使用している [shurcooL-graphql](https://github.com/cli/shurcooL-graphql) というパッケージによるものです。
先にリクエストしたい GraphQL オペレーションを構造体として定義して、それを内部で GraphQL のリクエストに変換して、API リクエストを実行します。

紹介した実装では最終的に下記の Query を実行しています。

```graphql
query ReviewRequestedPullRequests($q: String!) {
  search(first: 100, query: $q, type: ISSUE) {
    edges {
      node {
        ... on PullRequest {
          url
        }
      }
    }
  }
}
```

これを構造体で定義すると下記の定義になります。

```go
var query struct {
    Search struct {
        Edges []struct {
            Node struct {
                PullRequest struct {
                    URL string
                } `graphql:"... on PullRequest"`
            }
        }
    } `graphql:"search(first: 100, query: $q, type: ISSUE)"`
}
```

GraphQL オペレーションにおけるフィールドが、構造体のフィールドと対応するように定義します。
その際に Query の引数や、Inline Fragments などの GraphQL 特有の記法は構造体のタグとして定義します。
一通りの使用方法は [README](https://github.com/cli/shurcooL-graphql/blob/trunk/README.md) に記載してくれているので、興味のある方は見てみてください。

## 検索条件の定義

検索条件は`search`の引数の`query`で指定しています。その際に`query`で指定できる検索条件は下記にまとまっています。

https://docs.github.com/ja/search-github/searching-on-github/searching-issues-and-pull-requests

これは GitHub のリポジトリの Pull Requests 画面でも同じものが使用されています。
そのため、画面上から操作できる検索であれば、画面上から操作して表示されている検索条件をコピペして使用するのが楽だと思います。

![](/images/ee9d5010bb8ffd/pull-request-page.png)

今回の実装では下記のように検索条件を設定して、所属するの Organization 内の自分がレビュー依頼されている PR の一覧を取得しています。

```
is:pr is:open review-requested:@me org:<Organizationの名前>
```

# まとめ

今回初めて GitHub CLI の拡張機能を作成したのですが、開発の準備が 1 コマンドでできたり、便利なパッケージが提供されていたりと、思ったよりも簡単に拡張機能を作成することができました！
今後も GitHub 周りで便利ツールが欲しくなった時は、積極的に拡張機能の作成をしていこうと思いました！
