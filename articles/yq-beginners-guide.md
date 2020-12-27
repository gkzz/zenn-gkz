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

https://twitter.com/gkzvoice/status/1342856632523362307

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

## 2.環境/バージョン情報

```
$ yq --version
yq 2.11.1
```
### 2種類のyqコマンド
さて、yqコマンドは上述したとおり2種類合って紛らわしいです。

- 本記事で取り扱うyqコマンド
  - jqのYAML/XMLラッパー
  - **`jqコマンドのドキュメントを流用できる`**
    - e.g. [jq コマンドを使う日常のご紹介](https://twitter.com/gkzvoice/status/1337681052639227910?s=20)
[kislyuk/yq: Command-line YAML and XML processor - jq wrapper for YAML/XML documents](https://github.com/kislyuk/yq)

- もうひとつのyqコマンド
[mikefarah/yq: yq is a portable command-line YAML processor](https://github.com/mikefarah/yq)


### yqのインストール
※pythonの仮想環境venv上で検証を進めますが、直接`pip install yq`でも大丈夫です。
ご自身の環境に併せてyqをご用意ください。

```
$ python3 -m venv 38python3 -m venv 38
$ source 38/bin/activate
```
$ u=https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml && curl $u -o install.master.yaml
```

```
$ echo "{bar: dummy}" | yq -y > input00.yml
$ cat input00.yml
bar: dummy
$ yq -r '.bar' input00.yml 
dummy
```

```
$ echo "{bar: dummy}" | yq -y > input00.yml
$ cat input00.yml
bar: dummy
$ yq -r '.bar' input00.yml 
dummy
$ echo "foo: {bar: dummy}" | yq -y > input01.yml
$ cat input01.yml 
foo:
  bar: dummy


$ cat input01.yml 
foo: 
  bar: dummy
$ yq .foo input01.yml 
{
  "bar": "dummy"
}
$ yq .foo.bar input01.yml
"dummy"
```
