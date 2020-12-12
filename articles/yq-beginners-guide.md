---
title: "yqコマンド(jq wrapper for YAML)使い方備忘録"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [yq,yqml,cli,kubernetes]
published: false
---

## 0.はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。
本記事は、なんかいいかんじにパースしてくれるjqコマンドのyaml版のyqコマンドの使い方備忘録です。
本題に入る前に、なぜyqコマンドの使い方に関する備忘録を書こうと思ったのか、その理由をお伝えします。

#### 2種類のyqコマンド
yqコマンドは2種類合って紛らわしいので、本記事で取り扱うyqコマンドについて認識を合わせましょう。
- [kislyuk/yq: Command-line YAML and XML processor - jq wrapper for YAML/XML documents](https://github.com/kislyuk/yq)
もうひとつのyqコマンドもご参考までに掲載します。
- [mikefarah/yq: yq is a portable command-line YAML processor](https://github.com/mikefarah/yq)

## 1.本記事における問題点の共有
Kubernetesのmanifestファイルの中に書かれている一部の値をCIツールが書き換える必要がありました。
当初はsedでやろうかと思ったのですが、友人からyqいいよと教えていただき、yqについて調べたところ、いくつかの問題点を感じました。
具体的には以下のとおりです。

```
- yqにはjqコマンドのwrapper版とそうではないものがある
  - そのため、「yq　使い方」でググって見つかる記事はどちらのyqを使っているのか分からず、yqの使い方をググるのに苦労
- yqはjqに比べてドキュメントが少ない 
```

そこで、「ググれる」俺得なyqコマンド使い方について調べたことを本記事にまとめようと思いました。
なお、jqコマンドについては、すでに使い方に関する記事が出ています。
本記事では取り上げることができていないyqコマンドの使い方についてご存知の方は、ぜひ本記事のコメント欄にてご共有いただけるとうれしいです。

[jq コマンドを使う日常のご紹介](https://twitter.com/gkzvoice/status/1337681052639227910?s=20)



yq
https://github.com/kislyuk/yq/
