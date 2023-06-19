---
title: "CloudFrontで画像をリサイズしつつ配信する"
emoji: "🖼️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "AWSCDK", "CloudFront", "S3"]
published: false
---

# はじめに

みなさん画像などの配信はCloudFrontを使っていますでしょうか？
様々な画像サイズのものをあらかじめ用意しておくのは大変だったりしないでしょうか？

https://aws.amazon.com/jp/blogs/news/resizing-images-with-amazon-cloudfront-lambdaedge-aws-cdn-blog/

AWSが公式に、動的に様々なサイズの画像を作成・配信する方法を公開しています。
元記事ではCloudFormationを利用していますが、CDKで書き直してみました。
（提供されている関数も動くように修正などしてあります）

## 利用時のイメージ

画像のURLにパラメータを付与して取得。パラメータに応じて変換された画像を取得。

# 環境・料金

## 環境

- macOS: 13.4
- Node.js: 18.16
- AWS CDK: 2.84.0

## 料金

利用するのはCloudFrontとS3のみなので、余程のことがない限り0円で実行できます。
唯一、Route 53でドメインを利用する必要があり、そこだけ料金がかかります。
取得するドメインによっても変わりますが、年額で1500円〜程度です。

## 準備

Route 53とACMを設定する必要があります。
ドメインとACMは、以下の内容が設定されていれば大丈夫です。

詳しくは[こちらの記事](https://zenn.dev/gsy0911/articles/da47b660b7dd2b7d1ae7)と同じ内容になるようにしてください。

また、本記事では説明しませんが、環境にCDKv2を利用する準備も必要です。

# コード

プログラムはGitHub: [zenn-cloudfront-resize-image](https://github.com/gsy0911/zenn-cloudfront-resize-image)にて公開してあります。
実際に動かしたい場合はcloneして実行してみてください。

## ディレクトリ構成

ディレクトリ構成は以下のようになっています。
lambdaはnodeとPythonで動くものは作ってあります。
nodeは参考にしてあった記事のコードが、Pythonはそれをベースに作成したコードがあります。
nodeの方が機能としては豊富になっているので、nodeのコードを利用することを推奨します。

```text
/infrastructure/lib
├── CloudFrontAssetsStack.ts
├── common.ts
├── index.ts
├── lambda
│  ├── image_resize_node
│  │  ├── origin_response.js
│  │  ├── package.json
│  │  └── viewer_request.js
│  └── image_resize_python
│     ├── origin_response
│     │  ├── handler.py
│     │  └── requirements.txt
│     └── viewer_request
│        └── handler.py
├── LambdaEdgeStack.ts
├── params.example.ts
├── params.ts
└── XRegionParam.ts
```

## デプロイ

### デプロイに必要なパラメータの付与

デプロイの前にパラメータの設定を行います。
`infra/`に保存されている`paramsExample.ts`をコピーして`params.ts`を作成します。

```shell
$ cd infra
$ cp paramsExample.ts params.ts
```

以下の`params.ts`にある項目を更新します。

```diff typescript:params.ts
import { IAssetsLambdaEdgeStack } from './LambdaEdgeStack'
import { ICfAssetsStack } from './CloudFrontAssetsStack';
import { Environment } from 'aws-cdk-lib';

- const accountId: string = "000011112222"
- const s3BucketName: string = "your-buket"
+ const accountId: string = "777788889999"
+ const s3BucketName: string = "assets-bucket"

/**  */
export const cfAssetsParams: ICfAssetsStack = {
  cloudfront: {
-   certificate: "arn:aws:acm:us-east-1:000011112222:certificate/aaaabbbb-cccc-dddd-eeee-ffffgggghhhh",
+   certificate: "arn:aws:acm:us-east-1:777788889999:certificate/iiiijjjj-kkkk-llll-mmmm-ooooppppqqqq",
-   route53DomainName: "your.domain.com",
-   route53RecordName: "record.your.domain.com",
+   route53DomainName: "example.com",
+   route53RecordName: "assets.example.com",
    s3BucketName,
  }
}

export const assetsLambdaEdgeParams: IAssetsLambdaEdgeStack = {
  s3BucketName
}

export const envApNortheast1: Environment = {
  account: accountId,
  region: "ap-northeast-1"
}

export const envUsEast1: Environment = {
  account: accountId,
  region: "us-east-1"
}

```

加えて、利用する言語に応じてBucketの名前を直接記述してください。
Lambda@Edgeは環境変数も利用できないため、ハードコードが必要になります。

node版は以下のファイルを変更してください。

```diff javascript:infrastructure/lib/lambda/image_resize_node/origin_response.js
~~ 前略 ~~

const Sharp = require('sharp');

// set the S3 endpoints
- const BUCKET = 'your-bucket-here';
+ const BUCKET = 'assets-bucket';

exports.handler = (event, context, callback) => {

~~ 後略 ~~
```

Python版は以下のファイルを変更してください。

```diff python:infrastructure/lib/lambda/image_resize_python/origin_response/handler.py
~~ 前略 ~~

DEFAULT_QUALITY = 50
- BUCKET = "your-bucket-here"
+ BUCKET = "assets-bucket"


def resize_image(

~~ 後略 ~~
```

### デプロイ実行

上記の準備が終わったら、デプロイをします。


## リソースの削除

テストなどで作成する場合は、以下のコマンドを実行してリソースを削除してください。

```shell
$ cdk destroy
```

# おわりに
