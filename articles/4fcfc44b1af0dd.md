---
title: "Docker Engine stoppedってなった時の対処法"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker"]
published: true
---

# ある日

いつものようにDockerを触っていると、急に`docker`コマンドが動かなくなりました。docker desktopをみてみると以下の画面で止まっており、再起動や停止なども効かない状態になりました。

![](/images/docker_engine_stopped/docker_engine_stopped.png =600x)
*停止したdocker desktop*

私の環境依存の可能性もあるのですが、ここ4~5回くらい起きているため、備忘録がてら対処法をまとめます。

# 環境

環境は以下の通りです。

- Mac Book Air M2 (Sonoma 14.6)
- docker desktop: v4.33.0

体感的に、docker desktopを`v4.33.0`に更新してから同事象が頻繁に起きている気がします。

# 対象法

対象法は、以下のコマンドを実行するだけです。

```shell
$ rm -rf ~/Library/Caches/com.docker.docker/Data
$ rm -rf ~/Library/Group\ Containers/group.com.docker
$ rm -rf ~/Library/Containers/com.docker.docker
$ rm -rf ~/Library/Application\ Support/Docker\ Desktop
$ rm -rf ~/.docker 
```

すると、docker desktopのログイン情報なども削除され、無事動くようになります。

::: message
もしかしたら実行が不要なコマンドがあるかもですが、細かい精査はしていないです。
:::

::: message alert
dockerが完全に停止ている状態で上記のコマンドを実行してください。実行中だと消せないファイルなどが存在するためです。
:::

以上です。誰かの参考になれば幸いです。


# 参考

https://forums.docker.com/t/docker-desktop-is-stopped-on-mac-m1-monterey/137918/2
