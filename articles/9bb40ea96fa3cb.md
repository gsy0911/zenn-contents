---
title: "CloudFrontで画像をリサイズしつつ配信する"
emoji: "🖼️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "AWSCDK", "CloudFront", "S3"]
published: true
---

# はじめに

みなさん画像などの配信はCloudFrontを使っていますでしょうか？
様々な画像サイズのものをあらかじめ用意しておくのは大変だったりしないでしょうか？

https://aws.amazon.com/jp/blogs/news/resizing-images-with-amazon-cloudfront-lambdaedge-aws-cdn-blog/

AWSが公式に、動的に様々なサイズの画像を作成・配信する方法を公開しています。
元記事ではCloudFormationを利用していますが、CDKで書き直してみました。
（提供されている関数も動くように修正などしてあります）

## 利用時のイメージ

![](https://storage.googleapis.com/zenn-user-upload/22a860c7d9a7-20230621.png =600x)

1. サーバーに`https://your.domain.com/images/some_file.jpg?d=200x200` でアクセスする
2. `Lambda@Edge`にてURLが`https://your.domain.com/images/200x200/webp/some_file.jpg` に変換される
3. 変換されたURLでS3にアクセスする
4. S3にファイルが存在した場合6.に飛び、存在しない場合は5.の処理を実施する
5. 元々のアクセス先である`images/some_file.jpg`の画像を変換し、`images/200x200/webp/some_file.jpg`へ保存する
6. 取得したファイルか、変換したファイルを返す

画像をリサイズした後に保存しておくことで、2回目以降のリサイズ処理は無くせます。

# 環境・料金

## 環境

- macOS: 13.4
- Node.js: 18.16
- AWS CDK: 2.85.0

## 料金

利用するCloudFrontとS3は余程のことがない限り0円で実行できます。
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

:::message
リポジトリには、Pythonで画像の変換処理をしているコードもあります。
基本的な考えは同じで言語が違うだけなのでnode版のみの説明をします。
:::


## ディレクトリ構成

ディレクトリ構成は以下のようになっています。
大事なのは`lambda/`にある`viewer_request.js`と`origin_response.js`のファイルです。
XRegionParamの動作については[こちらの記事](https://zenn.dev/gsy0911/articles/820313c08a545922733f)をご覧ください。

```text
/infrastructure/lib
├── lambda/
│  └── image_resize_node/
│     ├── origin_response.js 👈 変換後の画像がない場合に、リサイズ・保存を実施して画像を返す
│     ├── package.json
│     └── viewer_request.js 👈 リクエストのURLをパラメータを元に変更する
├── CloudFrontAssetsStack.ts
├── LambdaEdgeStack.ts
├── common.ts
├── index.ts
├── params.example.ts
├── params.ts
└── XRegionParam.ts
```

`viewer_request.js`と`origin_response.js`のうちの重要な箇所を説明します。
元記事のコードからはほとんど変えていません。

`viewer_request.js`では、アクセスするURIを変更しています。
例えば、`/images/some_file.jpg?d=200x200`のURIへアクセスすると、
`/images/200x200/webp/some_file.jpg`というURIに変更します。


```javascript: infrastructure/lib/lambda/image_resize_node/viewer_request.js
const defaultHandler = (event, context, callback) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;

  const params = querystring.parse(request.querystring);
  let fwdUri = request.uri;

  // パラメータが渡されない場合には、
  // Lambda@Edgeでの処理を終了して、そのままの大きさの画像を返すようにしている
  if (!params.d) {
    callback(null, request);
    return;
  }

  // `d`で与えられたパラメータをxで分解して、widthとheightとして取得
  const dimensionMatch = params.d.split("x");
  let width = dimensionMatch[1];
  let height = dimensionMatch[2];

  // アクセス先のURIをパースする。例えば、/images/image.jpg
  const match = fwdUri.match(/(.*)\/(.*)\.(.*)/);
  let prefix = match[1];
  let imageName = match[2];
  let extension = match[3];

  // 受け付けられる画像サイズのみの判定をしている。
  // 全ての画像サイズを受け付けられるようにすると、ユーザーが無限にファイルを生成できてしまうため
  let matchFound = false;
  let variancePercent = (variables.variance / 100);
  for (let dimension of variables.allowedDimension) {
    let minWidth = dimension.w - (dimension.w * variancePercent);
    let maxWidth = dimension.w + (dimension.w * variancePercent);
    if (width >= minWidth && width <= maxWidth) {
      width = dimension.w;
      height = dimension.h;
      matchFound = true;
      break;
    }
  }
  if (!matchFound) {
    width = variables.defaultDimension.w;
    height = variables.defaultDimension.h;
  }

  // 新しくアクセスするためのURIを構築していく
  let url = [];
  url.push(prefix);
  url.push(width + "x" + height);

  // webpの確認
  let accept = headers['accept'] ? headers['accept'][0].value : "";
  if (accept.includes(variables.webpExtension)) {
    url.push(variables.webpExtension);
  } else {
    url.push(extension);
  }
  url.push(imageName + "." + extension);
  fwdUri = url.join("/");

  // CloudFrontへアクセスする際のURIを`/images/200x200/webp/image.jpg`に変更する
  request.uri = fwdUri;
  callback(null, request);
};
```

`origin_response.js`では、S3からの画像の取得状況に応じて以下のような挙動をします。

- ファイルが存在した場合
  - 何もせずに結果をそのまま返す
- ファイルが存在しない場合
  - 元のURIにアクセスして画像を取得
  - 取得した画像のリサイズ
  - リサイズした画像の保存
  - 結果を返す


```javascript: infrastructure/lib/lambda/image_resize_node/origin_response.js
exports.handler = (event, context, callback) => {
  const response = event.Records[0].cf.response;

  console.log("Response status code: %s", response.status);

  // S3から取得した時のスタータスコードの確認
  // 画像が存在しない場合のみ、リサイズなどの処理を実施
  if (response.status === 403 || response.status === 404 || response.status === "403" || response.status === "404") {

    let request = event.Records[0].cf.request;
    let params = querystring.parse(request.querystring);

    // パラメータがない場合には、本当に画像が無い場合なのでそのままの結果を返す
    if (!params.d) {
      callback(null, response);
      return;
    }

    // URIを取得する。ここでは`path=/images/100x100/webp/image.jpg`
    let path = request.uri;
    // 先頭の`/`を削除して、`key=images/200x200/webp/image.jpg`とする
    let key = path.substring(1);
    // prefix, width, height, imageNameを取得していく
    let prefix, originalKey, match, width, height, requiredFormat, imageName;
    try {
      match = key.match(/(.*)\/(\d+)x(\d+)\/(.*)\/(.*)/);
      prefix = match[1];
      width = parseInt(match[2], 10);
      height = parseInt(match[3], 10);

      // correction for jpg required for 'Sharp'
      requiredFormat = match[4] === "jpg" ? "jpeg" : match[4];
      imageName = match[5];
      originalKey = prefix + "/" + imageName;
    }
    catch (err) {
      // no prefix exist for image
      console.log("no prefix present..");
      match = key.match(/(\d+)x(\d+)\/(.*)\/(.*)/);
      width = parseInt(match[1], 10);
      height = parseInt(match[2], 10);

      // correction for jpg required for 'Sharp'
      requiredFormat = match[3] === "jpg" ? "jpeg" : match[3];
      imageName = match[4];
      originalKey = imageName;
    }

    // パースした結果からS3にアクセスして画像ファイルを取得
    S3.getObject({ Bucket: BUCKET, Key: originalKey }).promise()
      // sharpでのリサイズ処理
      .then(data => Sharp(data.Body)
        .resize(width, height)
        .toFormat(requiredFormat)
        .toBuffer()
      )
      .then(buffer => {
        // リサイズした結果をS3に保存する
        S3.putObject({
            Body: buffer,
            Bucket: BUCKET,
            ContentType: 'image/' + requiredFormat,
            CacheControl: 'max-age=31536000',
            Key: key,
            StorageClass: 'STANDARD'
        }).promise()
        .catch(() => { console.log("Exception while writing resized image to bucket")});

        // CloudFrontからのレスポンスを編集する
        response.status = 200;
        response.body = buffer.toString('base64');
        response.bodyEncoding = 'base64';
        response.headers['content-type'] = [{ key: 'Content-Type', value: 'image/' + requiredFormat }];
        callback(null, response);
      })
    .catch( err => {
      console.log("Exception while reading source image :%j",err);
    });
  }
  else {
    // 画像が存在する場合
    callback(null, response);
  }
};
```

## デプロイ

### デプロイに必要なパラメータの付与

デプロイの前にパラメータの設定を行います。
`infra/`に保存されている`paramsExample.ts`をコピーして`params.ts`を作成します。

```shell
$ cd infrastructure
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
Lambda@Edgeでは環境変数が利用できないためです。

以下のファイルを変更してください。

```diff javascript:infrastructure/lib/lambda/image_resize_node/origin_response.js
~~ 前略 ~~

const Sharp = require('sharp');

// set the S3 endpoints
- const BUCKET = 'your-bucket-here';
+ const BUCKET = 'assets-bucket';

exports.handler = (event, context, callback) => {

~~ 後略 ~~
```

### デプロイ実行

上記の準備が終わったら、デプロイをします。
デプロイは以下のコマンドを実行するだけです。

```shell
# /infrastructureディレクトリにて実行
$ cdk deploy zenn-cf-resize-cloudfront
```

上記のコマンドで、`addDependency`でnodeの`Lambda@Edge`もデプロイされます。

デプロイが完了したら、CloudFrontの画面に行き「オリジン」→「編集」を押下し、

![](https://storage.googleapis.com/zenn-user-upload/994a652fe2c3-20230626.png =600x)

「はい、バケットポリシーを自動で更新します」を選択して「変更を保存」を押下してください。

![](https://storage.googleapis.com/zenn-user-upload/10fd97696f04-20230626.png =600x)

こうすることで、CloudFrontからS3へのアクセスをOAIを介してアクセスできます。

### 挙動の確認

挙動の確認方法としては、適当な画像をS3にアップロードして、
CloudFront経由でアクセスしてみてください。
その際に、URLの末尾に`?d=200x200`などとするとリサイズ処理をかけることができます。

そして、S3にも変換後のファイルが保存されているかの確認もしてください。

## リソースの削除

テストなどで作成する場合は、以下のコマンドを実行してリソースを削除してください。
削除時には、Lambda@Edgeを利用している関係で、少し時間がかかります。

```shell
$ cdk destroy
```

# おわりに

画像のリクエスト時にリサイズして保存して、配信する仕組みを紹介しました。
色々と制約もあって完全に全てのケースを網羅できるわけではないですが、
役に立つシーンはあるかと思います。

# 参考

- （再掲）：[Amazon CloudFront & Lambda@Edge で画像をリサイズする](https://aws.amazon.com/jp/blogs/news/resizing-images-with-amazon-cloudfront-lambdaedge-aws-cdn-blog/)
