---
title: "go testのキャッシュの仕組みを理解して、テストコード変えずにCIを高速化する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "test", "githubactions", "ci", "zennfes2025free"]
published: true
published_at: 2025-10-06 07:30
publication_name: "drsprime"
---

# はじめに

普段の開発でGoを使用しているのですが、CIでのテスト実行時間に課題感を感じていました。
テストを高速化したいと思ったのですが、テストコード自体を改善するには結構大きな変更が必要な状況でした。

もっと簡単にできないかと考えていたところ、ローカル環境では`go test`のキャッシュが効いている場面がよくあり、「これはCIでも使えるのでは？」と思いました。ただ、テスト結果の不整合など起きそうで不安だったので、詳しく調べてみることにしました。

今回は、調査してわかった`go test`のキャッシュの仕組みと、GitHub ActionsでCI実行時間を短縮する方法について紹介したいと思います！
調査の過程でGoのソースコードも読んでみたので、内部実装についても触れながら解説していきます！

# go testのキャッシュとは

Go 1.10からテストキャッシュ機能が導入されており、テストの実行結果をキャッシュして、同じ条件で実行する際には前回の結果を再利用できるようになっています。

実際にキャッシュが利用されると、テスト実行時に`(cached)`と表示されます。

```bash
$ go test ./...
ok      github.com/example/pkg1    0.123s
ok      github.com/example/pkg2    (cached)
ok      github.com/example/pkg3    0.456s
```

この機能を活用することで、変更されていないパッケージのテストをスキップできるので、テストの実行時間を短縮できます。

# キャッシュの仕組み

`go test`のキャッシュがどのように動作しているのか、Goのソースコード（`src/cmd/go/internal/test/test.go`）を読みながら詳しく見ていきます。
参照するソースコードのバージョンは1.25です。

https://github.com/golang/go/blob/release-branch.go1.25/src/cmd/go/internal/test/test.go

キャッシュの判定処理は[tryCacheWithID](https://github.com/golang/go/blob/release-branch.go1.25/src/cmd/go/internal/test/test.go#L1748)関数で行われており、この関数の処理フローを理解することで、キャッシュの仕組みが明確になります。

## パッケージ単位でのキャッシュ管理

まず重要なポイントとして、**Goのテストキャッシュはパッケージ単位で管理されています**。

`go test ./...`のように複数のパッケージをテストする場合、各パッケージごとに個別にテストが実行され、それぞれのテスト結果が別々にキャッシュされます。

```bash
$ go test ./...
ok      github.com/example/pkg1    0.123s
ok      github.com/example/pkg2    (cached)  # pkg2だけキャッシュヒット
ok      github.com/example/pkg3    0.456s
```

この例では、`pkg2`だけがキャッシュヒットしていますが、これは`pkg2`のソースコードやテストコード、依存するファイルや環境変数が前回の実行から変更されていないことを意味します。一方で、`pkg1`や`pkg3`は何らかの変更があったため、実際にテストが実行されています。

このパッケージ単位のキャッシュ管理により、大規模なプロジェクトでも効率的にキャッシュを活用できるようになっています。あるパッケージを変更しても、変更していない他のパッケージのキャッシュは維持されるため、変更した部分のテストだけが実行されます。

## キャッシュ判定の全体フロー

`go test`を実行すると、各パッケージごとに以下のようなフローでキャッシュが判定されます。

1. **前提条件のチェック** - ローカルディレクトリモードやパッケージのルートチェック
2. **テスト引数の検証** - キャッシュ可能なフラグのみが使用されているかチェック
3. **2段階のキャッシュ検索**
   - 第1段階：テストIDの計算とテストログの取得
   - 第2段階：テストインプットIDの計算と結果の取得

それぞれのステップも詳しく解説していきます！

## 1. 前提条件のチェック

まず、キャッシュを使用できる条件を満たしているかをチェックします。

### ローカルディレクトリモードの判定

```go
func (c *runCache) tryCacheWithID(b *work.Builder, a *work.Action, id string) bool {
    if len(pkgArgs) == 0 {
        // Caching does not apply to "go test",
        // only to "go test foo" (including "go test .").
        if cache.DebugTest {
            fmt.Fprintf(os.Stderr, "testcache: caching disabled in local directory mode\n")
        }
        c.disableCache = true
        return false
    }

    // ...省略
}
```

`pkgArgs`が空の場合、つまり`go test`のように引数なしで実行された場合は、キャッシュが無効化されます。`go test ./...`や`go test ./pkg`のように明示的にパッケージを指定する必要があります。

### パッケージルートの確認

```go
if a.Package.Root == "" {
    // Caching does not apply to tests outside of any module, GOPATH, or GOROOT.
    if cache.DebugTest {
        fmt.Fprintf(os.Stderr, "testcache: caching disabled for package outside of module root, GOPATH, or GOROOT: %s\n", a.Package.ImportPath)
    }
    c.disableCache = true
    return false
}
```

テスト対象のパッケージが、モジュール、`GOPATH`、`GOROOT`のいずれかの配下にある必要があります。これらの外にあるパッケージはキャッシュの対象外となります。

## 2. テスト引数の検証

次に、コマンドライン引数がキャッシュ可能かどうかをチェックします。

```go
var cacheArgs []string
for _, arg := range testArgs {
    i := strings.Index(arg, "=")
    if i < 0 || !strings.HasPrefix(arg, "-test.") {
        if cache.DebugTest {
            fmt.Fprintf(os.Stderr, "testcache: caching disabled for test argument: %s\n", arg)
        }
        c.disableCache = true
        return false
    }
    switch arg[:i] {
    case "-test.benchtime",
        "-test.cpu",
        "-test.list",
        "-test.parallel",
        "-test.run",
        "-test.short",
        "-test.skip",
        "-test.timeout",
        "-test.failfast",
        "-test.v",
        "-test.fullpath":
        // These are cacheable.
        // Note that this list is documented above,
        // so if you add to this list, update the docs too.
        cacheArgs = append(cacheArgs, arg)
    case "-test.coverprofile",
        "-test.outputdir":
        // These are cacheable and do not invalidate the cache when they change.
        // Note that this list is documented above,
        // so if you add to this list, update the docs too.
    default:
        // nothing else is cacheable
        if cache.DebugTest {
            fmt.Fprintf(os.Stderr, "testcache: caching disabled for test argument: %s\n", arg)
        }
        c.disableCache = true
        return false
    }
}
```

この処理では、すべてのテスト引数をループして、キャッシュ可能なフラグのリストに含まれているかをチェックしています。
キャッシュ可能なフラグは上記のコードに記載されている通りで、それ以外のフラグが含まれている場合はキャッシュは利用されません。

`-test.coverprofile`と`-test.outputdir`は「キャッシュ可能だが、値が変わってもキャッシュを無効化しない」という特別な扱いになっています。
これらは出力先を変更するだけで、テストの実行結果には影響しないためです。

## 3. 2段階のキャッシュ検索

ここからがキャッシュの核心部分です。Goのテストキャッシュは**2段階の検索**を行います。

この仕組みにより、単純に**ソースコードが同じならキャッシュを使う**というだけでなく、**テストが依存する外部のファイルや環境変数の内容が同じか**までチェックすることで、キャッシュの検索を効率化しつつ、信頼性も高めています。

ソースコードにも以下のようなコメントがあります。

```go
// The test cache result fetch is a two-level lookup.
//
// First, we use the content hash of the test binary
// and its command-line arguments to find the
// list of environment variables and files consulted
// the last time the test was run with those arguments.
// (To avoid unnecessary links, we store this entry
// under two hashes: id1 uses the linker inputs as a
// proxy for the test binary, and id2 uses the actual
// test binary. If the linker inputs are unchanged,
// this way we avoid the link step, even though we
// do not cache link outputs.)
//
// Second, we compute a hash of the values of the
// environment variables and the content of the files
// listed in the log from the previous run.
// Then we look up test output using a combination of
// the hash from the first part (testID) and the hash of the
// test inputs (testInputsID).
```

このコメントが示すように、2つの異なるIDを組み合わせた**最終的なキー**でキャッシュを検索します。

- **`testID`**: テストのコード、ビルドフラグ、Goのバージョンなど、**テストバイナリ自体を決定する要素**から作られるID
- **`testInputsID`**: テスト実行時に**読み込まれる外部ファイル**や**参照される環境変数**の内容から作られるID

この2つが前回と今回で一致して初めて、「テストを再実行する必要はない」と判断し、キャッシュを利用します。

### 第1段階：テストIDの計算とテストログの取得

まず、テストバイナリの内容とコマンドライン引数から`testID`を計算します。

```go
h := cache.NewHash("testResult")
fmt.Fprintf(h, "test binary %s args %q execcmd %q", id, cacheArgs, work.ExecCmd)
testID := h.Sum()
if c.id1 == (cache.ActionID{}) {
    c.id1 = testID
} else {
    c.id2 = testID
}
if cache.DebugTest {
    fmt.Fprintf(os.Stderr, "testcache: %s: test ID %x => %x\n", a.Package.ImportPath, id, testID)
}
```

この`testID`は、テストバイナリの内容（ソースコードやテストコード）、コマンドライン引数、実行コマンドから計算されます。ソースコードが変更されると、この`testID`も変わります。

次に、この`testID`を使って、前回のテスト実行時に記録された「テストログ」を取得します。

```go
// Load list of referenced environment variables and files
// from last run of testID, and compute hash of that content.
data, entry, err := cache.GetBytes(cache.Default(), testID)
if !bytes.HasPrefix(data, testlogMagic) || data[len(data)-1] != '\n' {
    if cache.DebugTest {
        if err != nil {
            fmt.Fprintf(os.Stderr, "testcache: %s: input list not found: %v\n", a.Package.ImportPath, err)
        } else {
            fmt.Fprintf(os.Stderr, "testcache: %s: input list malformed\n", a.Package.ImportPath)
        }
    }
    return false
}
```

テストログ(`data`)とは、前回のテスト実行時に**どの環境変数を参照し、どのファイルを読み込んだか**を記録したものです。

例えば、テストが`config.json`を読み、環境変数`API_KEY`を参照した場合、テストログには以下のような内容が記録されます。

```
# test log
getenv API_KEY
open /home/user/project/testdata/config.json
stat /home/user/project/testdata
```

### 第2段階：テストインプットIDの計算と結果の取得

取得したテストログを元に、現在の環境変数とファイルの状態から`testInputsID`を計算します。

```go
testInputsID, err := computeTestInputsID(a, data)
if err != nil {
    return false
}
if cache.DebugTest {
    fmt.Fprintf(os.Stderr, "testcache: %s: test ID %x => input ID %x => %x\n", a.Package.ImportPath, testID, testInputsID, testAndInputKey(testID, testInputsID))
}
```

この`computeTestInputsID`関数が、2段階キャッシュの最も重要な処理です。
先ほど取得したテストログ（`data`）を元に、そこに書かれているファイルや環境変数を**現在のファイルシステムや環境から実際に読み込み**、読み込んだ**現在の内容**すべてのハッシュ値を計算します。

この仕組みにより、「テストコードは全く同じでも、`config.json`の中身が書き換わっている」といった状況を検知できます。
その場合、`testInputsID`が前回と異なる値になるため、キャッシュはヒットしません。

`testInputsID`の計算では、テストログに記録された操作（`getenv`、`chdir`、`stat`、`open`）を1行ずつ処理し、それぞれの現在の状態をハッシュに含めていきます。

```go
func computeTestInputsID(a *work.Action, testlog []byte) (cache.ActionID, error) {
    testlog = bytes.TrimPrefix(testlog, testlogMagic)
    h := cache.NewHash("testInputs")
    // The runtime always looks at GODEBUG, without telling us in the testlog.
    fmt.Fprintf(h, "env GODEBUG %x\n", hashGetenv("GODEBUG"))
    pwd := a.Package.Dir
    for _, line := range bytes.Split(testlog, []byte("\n")) {
        if len(line) == 0 {
            continue
        }
        s := string(line)
        op, name, found := strings.Cut(s, " ")
        if !found {
            if cache.DebugTest {
                fmt.Fprintf(os.Stderr, "testcache: %s: input list malformed (%q)\n", a.Package.ImportPath, line)
            }
            return cache.ActionID{}, errBadTestInputs
        }
        switch op {
        default:
            if cache.DebugTest {
                fmt.Fprintf(os.Stderr, "testcache: %s: input list malformed (%q)\n", a.Package.ImportPath, line)
            }
            return cache.ActionID{}, errBadTestInputs
        case "getenv":
            fmt.Fprintf(h, "env %s %x\n", name, hashGetenv(name))
        case "chdir":
            pwd = name // always absolute
            fmt.Fprintf(h, "chdir %s %x\n", name, hashStat(name))
        case "stat":
            if !filepath.IsAbs(name) {
                name = filepath.Join(pwd, name)
            }
            if a.Package.Root == "" || search.InDir(name, a.Package.Root) == "" {
                // Do not recheck files outside the module, GOPATH, or GOROOT root.
                break
            }
            fmt.Fprintf(h, "stat %s %x\n", name, hashStat(name))
        case "open":
            if !filepath.IsAbs(name) {
                name = filepath.Join(pwd, name)
            }
            if a.Package.Root == "" || search.InDir(name, a.Package.Root) == "" {
                // Do not recheck files outside the module, GOPATH, or GOROOT root.
                break
            }
            fh, err := hashOpen(name)
            if err != nil {
                if cache.DebugTest {
                    fmt.Fprintf(os.Stderr, "testcache: %s: input file %s: %s\n", a.Package.ImportPath, name, err)
                }
                return cache.ActionID{}, err
            }
            fmt.Fprintf(h, "open %s %x\n", name, fh)
        }
    }
    sum := h.Sum()
    return sum, nil
}
```

### 環境変数のハッシュ計算

環境変数が存在しない場合は`0`、存在する場合は`1`に続けて値をハッシュに含めます。
環境変数の値が変わると、このハッシュ値も変わるため、キャッシュが無効化されます。

```go
func hashGetenv(name string) cache.ActionID {
    h := cache.NewHash("getenv")
    v, ok := os.LookupEnv(name)
    if !ok {
        h.Write([]byte{0})
    } else {
        h.Write([]byte{1})
        h.Write([]byte(v))
    }
    return h.Sum()
}
```

### ファイルのハッシュ計算

ファイルのハッシュ計算では、ファイルの内容をハッシュ化するのではなく、ファイルサイズとmodtime（更新時刻）をハッシュ化しています。

コメントにもあるように、ファイルが非常に大きい可能性があるため、内容全体をハッシュ化するのを避け、mtimeとサイズで代用しています。
さらに、`modTimeCutoff`（2秒）よりも新しいファイルは、ファイルシステムの精度の問題で変更を検知できない可能性があるため、キャッシュを拒否します。

```go
func hashOpen(name string) (cache.ActionID, error) {
    h := cache.NewHash("open")
    info, err := os.Stat(name)
    if err != nil {
        fmt.Fprintf(h, "err %v\n", err)
        return h.Sum(), nil
    }
    hashWriteStat(h, info)
    if info.IsDir() {
        files, err := os.ReadDir(name)
        if err != nil {
            fmt.Fprintf(h, "err %v\n", err)
        }
        for _, f := range files {
            fmt.Fprintf(h, "file %s ", f.Name())
            finfo, err := f.Info()
            if err != nil {
                fmt.Fprintf(h, "err %v\n", err)
            } else {
                hashWriteStat(h, finfo)
            }
        }
    } else if info.Mode().IsRegular() {
        // Because files might be very large, do not attempt
        // to hash the entirety of their content. Instead assume
        // the mtime and size recorded in hashWriteStat above
        // are good enough.
        //
        // To avoid problems for very recent files where a new
        // write might not change the mtime due to file system
        // mtime precision, reject caching if a file was read that
        // is less than modTimeCutoff old.
        if time.Since(info.ModTime()) < modTimeCutoff {
            return cache.ActionID{}, errFileTooNew
        }
    }
    return h.Sum(), nil
}

func hashWriteStat(h io.Writer, info fs.FileInfo) {
    fmt.Fprintf(h, "stat %d %x %v %v\n", info.Size(), uint64(info.Mode()), info.ModTime(), info.IsDir())
}
```

### 最終的なキャッシュキーでテスト結果を取得

`testID`と`testInputsID`の両方が計算できたら、それらを組み合わせた最終的なキャッシュキーで、実際のテスト結果を取得します。

```go
// Parse cached result in preparation for changing run time to "(cached)".
// If we can't parse the cached result, don't use it.
data, entry, err = cache.GetBytes(cache.Default(), testAndInputKey(testID, testInputsID))
```

`testAndInputKey`関数で、2つのIDを組み合わせて最終的なキャッシュキーを生成します。
この最終キーを使って、キャッシュから実際のテスト結果（標準出力やエラー情報が記録されたテストログ）を取得します。

ここでデータが正しく取得できれば、テストはキャッシュヒットと見なされます。ただし、キャッシュエントリが見つかっても、キャッシュの有効期限が切れている場合はキャッシュを使用しません。

```go
if entry.Time.Before(testCacheExpire) {
    if cache.DebugTest {
        fmt.Fprintf(os.Stderr, "testcache: %s: test output expired due to go clean -testcache\n", a.Package.ImportPath)
    }
    return false
}
```
### キャッシュがヒットした場合の処理

キャッシュがヒットした場合、実行時間を`(cached)`に書き換えて出力します。
これにより、普段よく目にするキャッシュされたテスト結果の形式になります。

```go
// Committed to printing.
c.buf = new(bytes.Buffer)
c.buf.Write(data[:j])
c.buf.WriteString("(cached)")
for j < len(data) && ('0' <= data[j] && data[j] <= '9' || data[j] == '.' || data[j] == 's') {
    j++
}
c.buf.Write(data[j:])
return true
```

# GitHub Actionsでの実装

ここまでで`go test`のキャッシュの仕組みを理解できたので、次はGitHub ActionsでCIを高速化する方法について説明していきます。

## Goのキャッシュ機構を活用する

これまで見てきたように、`go test`は以下の要素が変更されていない場合、自動的にキャッシュを利用します。

- ソースコードやテストコード（パッケージ単位）
- テスト実行時に参照した環境変数
- テスト実行時に読み込んだファイル

そのため、CI上で実行する場合も**基本的にはGoのキャッシュ機構に任せれば良い**です。
GitHub Actionsの`setup-go`アクションを使っている場合は、ビルドキャッシュ（`~/.cache/go-build`）が流用されるため、Goのテストキャッシュも自然と利用されます。

```yaml
name: Test

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: 'go.mod'

      - name: Run tests
        run: go test -v ./...
        # 変更されていないパッケージは (cached) と表示される
```

自分のプロジェクトでは、`~/.cache/go-build`自体は復元されていたものの、テストコマンドに`-race`などの非キャッシュ可能なフラグが含まれていたため、キャッシュが全く利用されていませんでした。
これらのフラグを削除することで、キャッシュが正しく機能するようになりました。

## Goが検知できない変更への対処

しかし、1つ注意点があります。
`go test`のキャッシュは**テスト実行時に実際に読み込まれたファイル**のみを追跡します。

例えば、以下のようなケースでは、Goのキャッシュ機構だけでは不十分です。

- 設定ファイル（`.yaml`など）やマイグレーションファイルを変更したが、ソースコードには含まれていない
- Dockerfileやデプロイ設定を変更した
- ドキュメントだけを更新した

これらの変更は、テスト実行に影響を与える可能性がありますが、Goのキャッシュ機構では検知できません。
その結果、本来テストすべき変更があるのにキャッシュがヒットしてしまうため、偽陰性（テストが通ったように見えるが実際には問題がある）が発生するリスクがあります。

### 条件付きキャッシュクリア

この問題に対処するため、Goファイル以外の変更があった場合にキャッシュをクリアするGitHub Actionsを作成しました。
`dorny/paths-filter`を使って、変更されたファイルがGoファイルのみかどうかを判定し、Goファイル以外の変更がある場合やmainブランチの場合は`go clean -testcache`でキャッシュをクリアします。
これにより、Goファイル以外の変更があった場合に、テストが確実に実行されるようにすることで、偽陰性のリスクを回避できます。

```yaml
name: Clean Go Cache Conditionally

runs:
  using: "composite"
  steps:
    - name: Check non go file changes
      id: changes
      uses: dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36 # v3.0.2
      with:
        predicate-quantifier: 'every'
        filters: |
          has-not-go-file:
            - '!**/*.go'
            - '!**/*.mod'
            - '!**/*.sum'
    - name: Clean test cache if needed
      shell: bash
      run: |
        if [ "${{ steps.changes.outputs.has-not-go-file }}" == "true" ] || [ "${{ github.ref }}" == "refs/heads/main" ]; then
          echo "Cleaning test cache due to non-go file changes or main branch"
          go clean -testcache
        else
          echo "Skipping cache clean - only go files changed and not on main branch"
        fi
```

このアクションを`.github/actions/clean-go-cache/action.yml`として保存し、ワークフローから以下のように呼び出せます。

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@08eba0b27e820071cde6df949e0beb9ba4906955 # v4.3.0

      - uses: actions/setup-go@d35c59abb061a4a6fb18e82ac0862c26744d6ab5 # v5.5.0
        with:
          go-version-file: 'go.mod'

      - name: Clean cache conditionally
        uses: ./.github/actions/clean-go-cache

      - name: Run tests
        run: go test -v ./...
```

これにより、Goファイルのみの変更の場合はキャッシュを保持してCIを高速化し、Goファイル以外の変更がある場合とmainブランチの場合はキャッシュを削除して完全なテストを保証できます。

# まとめ

今回は`go test`のキャッシュの仕組みを調査して、GitHub ActionsでCIを高速化する方法について紹介しました！
キャッシュの仕組みを理解することで、意図しないキャッシュミスを防げるようになり、より効率的にCIを運用できるようになります。

実際に自分が開発しているプロダクトでは、今回の方法を使用することで、CIでのテストの実行時間を約9分から約3.5分へと、半分以下に短縮することができました！

今回の方法の導入自体は非常に簡単に行えると思うので、テストの実行時間に悩んでいる方はぜひ試してみてください！
