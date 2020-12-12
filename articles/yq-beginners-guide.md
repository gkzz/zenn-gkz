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

## 1.本記事における問題点の共有
突然ではありますが、Kubernetesのmanifestファイルの中に書かれている一部の値をCLIで、あるいはCIツールの実行中に書き換えるケースってないでしょうか？
僕もそういったケースに遭遇しました。
当初はsedでやろうかと思ったのですが、友人からyqいいよと教えていただき、yqについて調べたところ、いくつかの問題点を感じました。
具体的には以下のとおりです。

```
- yqにはjqコマンドのwrapper版とそうではないものがある
  - そのため、ググって見つかった記事はどちらのyqを使っているか分からないことがある
  - yqの使い方を調べることはたいへん！！
- yqはjqに比べてドキュメントの絶対数も少ない
```

そこで、「ググれる」俺得なyqコマンド使い方について調べたことを本記事にまとめようと思いました。

#### 2種類のyqコマンド
さて、yqコマンドは上述したとおり2種類合って紛らわしいです。
本記事で取り扱うyqコマンドはこちらです。
[kislyuk/yq: Command-line YAML and XML processor - jq wrapper for YAML/XML documents](https://github.com/kislyuk/yq)

もうひとつのyqコマンドはこちらです。
[mikefarah/yq: yq is a portable command-line YAML processor](https://github.com/mikefarah/yq)

なお、jqコマンドについては、すでに使い方に関する記事が出ています。
本記事では取り上げることができていないyqコマンドの使い方についてご存知の方は、ぜひ本記事のコメント欄にてご共有いただけるとうれしいです。

[jq コマンドを使う日常のご紹介](https://twitter.com/gkzvoice/status/1337681052639227910?s=20)




