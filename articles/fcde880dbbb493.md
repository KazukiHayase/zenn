---
title: "AWS SDK for Go v2 を使ってS3に画像をアップロードする"
emoji: "🍣"
type: "tech"
topics:
  - "aws"
  - "go"
  - "s3"
published: true
published_at: "2021-07-14 21:06"
publication_name: "buyselltech"
---

# 概要
APIGatewayにPOSTされた画像を、lambda経由でS3にアップロードして登録したファイル名の配列を返すAPIを実装したいと思います。デプロイにAWS SAM、API定義にOpenAPIを使用していきます。また、せっかくOpenAPIでAPIの定義をしているので、OpenAPI Generatorを使用してサーバーサイドで使用するコードを自動生成する方法も合わせて紹介したいと思います。

# AWS SDK for Go v2とは
[AWS SDK for Go v2](https://aws.github.io/aws-sdk-go-v2/docs/)とはAWSサービスを使用するGoアプリケーションを構築するために、使用できるAPIやユーティリティを提供してくれる公式のSDKです。2021年の1月にv2が一般公開されたばかりで、今回はこちらを使用してlambdaからS3に画像をアップロードする実装をしていきたいと思います。

# 環境
- AWS CLI 2.1.4
- SAM CLI 1.23.0
- Golang 1.16

# プロジェクト生成
まずはプロジェクトを生成していきます。
```sh
$ sam init

-----------------------
Generating application:
-----------------------
Name: sam-app
Runtime: go1.x
Dependency Manager: mod
Application Template: hello-world
Output Directory: .
```

生成されたプロジェクトをもとに下記のような下記のような構成にします。
```
 ├── document
 │   └── openapi.yaml
 │   └── Makefile
 ├── upload-image
 │   └── main.go
 ├── pkg
 │   └── s3
 │       └── s3.go
 ├── go.mod
 ├── go.sum
 ├── docker-compose.yaml
 └── template.yaml
```
今回は`openapi.yaml`からコードを自動生成するので`document`ディレクトリ配下にファイルを作成しています。

## ローカル環境構築
ローカル環境を構築するために`docker-compose.yaml`を記述していきます。ローカルでのS3は[locakstack](https://github.com/localstack/localstack)というツールを使用して構築します。また`sam local`で起動するhttpサーバーやlambdaはDocekr上で起動するので、その際に独自で定義したDockerとSAMで起動するDockerで通信ができるようにnetworksも定義します。

```yaml:docker-compose.yaml
version: '3.0'

services:
  localstack:
    image: localstack/localstack
    environment:
      - SERVICES=s3
      - DEFAULT_REGION=ap-northeast-1
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - ./data/localstack:/tmp/localstack
    ports:
      - 4566:4566
    networks:
      sam-app:

networks:
  sam-app:
    name: sam-app
    driver: bridge
```

## OpenAPI定義
`openapi.yaml`にAPIの定義をしていきます。
```yaml:openapi.yaml
openapi: 3.0.0
info:
  title: OpenAPI sam-app
  description: sam-appのAPI
  version: 1.0.0

paths:
  /images:
      post:
        summary: 画像アップロードAPI
        description: 画像をS3にアップロードする
        requestBody:
          required: true
          content:
            multipart/form-data:
              schema:
                type: object
                properties:
                  file:
                    description: 画像のバイナリ
                    type: string
                    format: binary
                required:
                  - file
        responses:
          '200':
            description: S3の画像のファイル名を返却
            content:
              application/json:
                schema:
                  $ref: '#/components/schemas/UploadImageResponse'
                example:
                  url: [ 'example1.jpg', 'example2.jpg' ]
        x-amazon-apigateway-integration:
          credentials:
            Fn::Sub: ${ApiRole.Arn}
          uri:
            Fn::Sub: arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${UploadImageFunction.Arn}/invocations
          passthroughBehavior: when_no_templates
          httpMethod: POST
          type: aws_proxy

components:
  schemas:
    UploadImageResponse:
      description: 画像アップロードAPIレスポンス
      type: object
      properties:
        file_names:
          type: array
          description: S3の画像バイナリのファイル名の配列
          items:
            type: string
          example: [ 'example1.jpg', 'example2.jpg' ]
      required:
        - file_names
```

次に`openapi.yaml`からGolangのコードを自動生成するためにMakefileを定義します。
```make:Makefile
DIR := $(shell pwd)

.PHONY: generate_server

generate_server:
	mkdir ${DIR}/Server
	docker run --rm \
	-v ${DIR}:/local \
	-e JAVA_OPTS="-Dlog.level=warn" \
	openapitools/openapi-generator-cli:v5.0.1 generate \
		-i /local/openapi.yaml \
		-g go \
		-o /local/Server

```
これでGolangのコードが生成できるようになりました！今回は生成されたファイルの内、`model_upload_image_response.go`を使用していきます。こちらにはOpenAPIで定義したレスポンスの構造体とその構造体に関連する関数が定義されています。また、生成されたモジュールを使用するためのコードを`go.mod`に追加します。詳細は[こちら](https://github.com/golang/go/wiki/Modules#can-i-work-entirely-outside-of-vcs-on-my-local-filesystem)をご覧ください。
```go:model_upload_image_response.go
 *
 * API version: 1.0.0
 */

// Code generated by OpenAPI Generator (https://openapi-generator.tech); DO NOT EDIT.

package openapi

import (
	"encoding/json"
)

// UploadImageResponse 画像アップロードAPIレスポンス
type UploadImageResponse struct {
	// S3の画像バイナリのファイル名の配列
	FileNames []string `json:"file_names"`
}

// NewUploadImageResponse instantiates a new UploadImageResponse object
// This constructor will assign default values to properties that have it defined,
// and makes sure properties required by API are set, but the set of arguments
// will change when the set of required properties is changed
func NewUploadImageResponse(fileNames []string, ) *UploadImageResponse {
	this := UploadImageResponse{}
	this.FileNames = fileNames
	return &this
}

// 省略

```
```go:go.mod
module sam-app

go 1.16

require (
	github.com/aws/aws-lambda-go v1.23.0
	sam-app/openapi v1.0.0 // 追加
)

replace sam-app/openapi v1.0.0 => ./document/Server // 追加

```

# S3へのアップロード実装
次に実際にS3にアップロードする部分の処理を書いていきます。s3へのアップロードする処理は、パッケージとして切り出して定義して汎用的にします。
（今回はs3へのアップロードの処理にフォーカスするのでエラーハンドリングは全てpanicで処理してます）
## s3.go
```go:s3.go
package s3

import (
	"context"
	"io"
	"os"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/s3"
)

var (
	client *s3.Client
)

type S3ObjectAPI interface {
	PutObject(
		ctx context.Context,
		params *s3.PutObjectInput,
		optFns ...func(*s3.Options),
	) (*s3.PutObjectOutput, error)
}

func PutFile(c context.Context, api S3ObjectAPI, input *s3.PutObjectInput) (*s3.PutObjectOutput, error) {
	return api.PutObject(c, input)
}

func init() {

	customResolver := aws.EndpointResolverFunc(func(service, region string) (aws.Endpoint, error) {
		if os.Getenv("AWS_SAM_LOCAL") == "true" {
			return aws.Endpoint{
				PartitionID:   "aws",
				URL:           "http://localstack:4566",
				SigningRegion: "ap-northeast-1",
			}, nil
		}
		return aws.Endpoint{}, &aws.EndpointNotFoundError{}
	})

	cfg, err := config.LoadDefaultConfig(context.TODO(), config.WithEndpointResolver(customResolver))
	if err != nil {
		panic(err)
	}

	client = s3.NewFromConfig(cfg, func(o *s3.Options) {
		o.UsePathStyle = true
	})
}

func Put(bucket string, filename string, file io.Reader) error {
	input := &s3.PutObjectInput{
		Bucket: aws.String(bucket),
		Key:    aws.String(filename),
		Body:   file,
	}

	_, err := PutFile(context.TODO(), client, input)
	if err != nil {
		return err
	}

	return nil
}
```
ポイントとしては下記の2点です。

- 環境によってアップロード先のS3を変更

SAMでは`AWS_SAM_LOCAL`という[環境変数](https://github.com/aws/aws-sam-cli/commit/5e56d94c22d71b1ae2a18057ba9182372cade11b)でローカル環境かどうかを判断できます。ローカル環境の場合はエンドポイントにlocalstackを指定してます。前述しましたがSAMとlocalstackはDocker間通信になるので`localhost`ではなく`docker-compose.ymal`で定義したサービス名の`localhost`を指定しています。

- pathStyleを指定

デフォルトの`s3.Client`では仮想ホスト形式を使用してるのですが、これだとs3アップロード時にエラーが出てしまうのでパス形式を使用するようにClient生成時のオプションで指定しています。

今回はS3の`PutObject`の実装のみですが、他の関数については[公式のExamples](https://aws.github.io/aws-sdk-go-v2/docs/code-examples/s3/putobject/)がとても参考になるのでそちらをご覧ください。
## main.go
s3へのアップロード部分の実装が完了したので次に、APIのリクエストから画像を取得してアップロードする部分の処理を実装していきます。
```go:main.go
package main

import (
	"bytes"
	"encoding/base64"
	"mime"
	"mime/multipart"
	"net/http"
	"sam-app/openapi"
	"sam-app/pkg/s3"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

// APIGatewayProxyRequestのHeaderをhttpモジュールのHeaderに変換
func convertToHttpHeader(rh map[string][]string) http.Header {
	headers := http.Header{}
	for h, values := range rh {
		for _, v := range values {
			headers.Add(h, v)
		}
	}
	return headers
}

func handler(request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {

	var (
		headers   = convertToHttpHeader(request.MultiValueHeaders)
		fileNames = []string{}
	)

	_, params, err := mime.ParseMediaType(headers.Get("Content-Type"))
	if err != nil {
		panic(err)
	}
	// リクエストボディをbase64でデコード
	recBody, err := base64.StdEncoding.DecodeString(request.Body)
	if err != nil {
		panic(err)
	}

	// multipart/form-dataをパース
	boundary := params["boundary"]
	r := bytes.NewReader(recBody)
	mr := multipart.NewReader(r, boundary)
	form, err := mr.ReadForm(2 * 1_000_000) // 2MB
	if err != nil {
		panic(err)
	}

	// 送られてきた画像のループ処理
	for _, fh := range form.File["file"] {
		fileName := fh.Filename
		fileNames = append(fileNames, fileName)
		f, err := fh.Open()
		if err != nil {
			panic(err)
		}

		// S3にアップロード
		if err := s3.Put("sam-app-images", fileName, f); err != nil {
			panic(err)
		}
		f.Close()
	}

	resBody := openapi.UploadImageResponse{FileNames: fileNames}
	jsonBody, err := resBody.MarshalJSON()
	if err != nil {
		panic(err)
	}

	return events.APIGatewayProxyResponse{
		Body:       string(jsonBody),
		StatusCode: 200,
	}, nil
}

func main() {
	lambda.Start(handler)
}
```
ポイントは3点です。

- リクエストヘッダー


公式の[Issue](https://github.com/aws/aws-lambda-go/issues/117)にもあるように`APIGatewayProxyRequest`のHeaderはcase sensitiveであり、ローカルと本番環境ではHeaderの値を取得する際に指定する文字列が変わってしまうので、case insensitiveな[net/http](https://pkg.go.dev/net/http)のHeaderに変換してます。

- リクエストボディのデコード

リクエストボディは`APIGatewayProxyRequest.Body`から取得できるのですが、この値がString型なのでmultipart/form-dataはbase64でエンコードされて格納されています。なのでbase64でデコードしてファイルのデータをなどを扱えるようにします。

- multipart/form-dataのパース

上記で説明したようにbase64でデコードしたmultipart/form-dataを扱うために、[mime](https://pkg.go.dev/mime/multipart)を使用してGoの構造体にパースします。パースした構造体からファイル名や実際の画像を取得して先ほど定義したS3へアップロードする関数を呼び出して画像をアップロードします。

# 動作確認
ここまでの実装が完了したら動作確認をしていきます！
## ローカル環境
コンテナを起動後、下記コマンドでlocalstackにバケットを作成します。
```sh
$ docker-compose up -d
$ aws s3 mb s3://sam-app-images --endpoint-url http://localhost:4566
```
下記コマンドでローカルのサーバーを立てます。localstackのDockerにアクセスできるようにネットワークをオプションで指定します。
```sh
$ sam build
$ sam local start-api --docker-network sam-app

Mounting UploadImageFunction at http://127.0.0.1:3000/images [POST]
You can now browse to the above endpoints to invoke your functions. You do not need to restart/reload SAM CLI while working on your functions, changes will be reflected instantly/automatically. You only need to restart SAM CLI if you update your AWS SAM template
2021-06-27 14:03:19  * Running on http://127.0.0.1:3000/ (Press CTRL+C to quit)
```
上記のコマンドで出力されたエンドポイントに向けてcurlを叩きます。
```sh
$ curl --request POST \
  --url http://127.0.0.1:3000/images \
  --form file=@example.jpg 

{"file_names":["example.jpg"]}
```
localstackのS3にアップロードされているのが確認できたらOKです！
```sh
$ aws s3 ls s3://sam-app-images --endpoint-url=http://localhost:4566

2021-06-27 14:29:28       6306 example.jpg
```

## デプロイ
ローカルでの動作確認が取れたらデプロイして確認します。下記コマンドで対話形式でパラーメーターを入力していきます。
```sh
$ sam deploy --capabilities CAPABILITY_NAMED_IAM --guided
```
作成したAPIのエンドポイントを叩いてS3に画像がアップロードされていれば全て完了です！

# まとめ
今回はAWS SDK for Go v2を使ってS3に画像をアップロードしました。リリースされてまもないので出回ってる情報が少なかったのですが、公式がとても参考になりました。
またOpenAPIからGoのコードを自動生成することにより、独自で定義するコードが減るので開発効率も上がると思うので自動生成できるコードは積極的に使用するのがいいなと感じました！

