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

## デプロイ

### デプロイに必要なパラメータの付与

デプロイの前にパラメータの設定を行います。
`infra/`に保存されている`paramsExample.ts`をコピーして`params.ts`を作成します。

```shell
$ cd infra
$ cp paramsExample.ts params.ts
```

```diff typescript:params.ts
+ a
- b
```

### デプロイ実行

上記の準備が終わったら、デプロイをします。


## リソースの削除

テストなどで作成する場合は、以下のコマンドを実行してリソースを削除してください。

```shell
$ cdk destroy
```

# おわりに
