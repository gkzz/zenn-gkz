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


## 1.本記事のサマリー

- yqでできること
  - yqはjsonをyamlに変換
  - yamlからjson/yaml形式に出力
  - yamlの任意の箇所を書き換える

yqコマンドなら、数千行のmanifestもいいかんじに該当箇所を探すことができます。
Gitlab Runner上でmanifestの一部をsedしていたことをyqコマンドでシュッとすることもできます。

いいことずくめのyqコマンドなのですが、使ってみようと触ってみたら問題点を感じました。

## 2.本記事における問題点の共有

```
- yqはjqに比べてドキュメントの絶対数が少ない
- yqはjqのラッパーなのだからjqのオプションを叩いても微妙に違う気がする
- yqにはjqコマンドのwrapper版とそうではないものの2種類があり、ググる力が問われる
```

そこで、「ググれる」俺得なyqコマンド使い方について調べたことを本記事にまとめようと思ったわけです。

## 3.環境/バージョン情報

```
$ yq --version
yq 2.11.1
```
### 3-1.種類のyqコマンド
さて、yqコマンドは上述したとおり2種類合って紛らわしいので本記事で扱うyqコマンドについて確認しておきましょう。

- jqのYAML/XMLラッパー
  - [kislyuk/yq: Command-line YAML and XML processor - jq wrapper for YAML/XML documents](https://github.com/kislyuk/yq)
  - **`jqコマンドのドキュメントを流用できる`**
    - e.g. [jq コマンドを使う日常のご紹介](https://twitter.com/gkzvoice/status/1337681052639227910?s=20)
- もうひとつのyqコマンド
  - [mikefarah/yq: yq is a portable command-line YAML processor](https://github.com/mikefarah/yq)


## 4.yqのインストール
※pythonの仮想環境venv上で検証を進めますが、直接`pip install yq`でも大丈夫です。
ご自身の環境に併せてyqをご用意ください。

```
$ python3 -m venv 38python3 -m venv 38
$ source 38/bin/activate
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


```
$ u=https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml && curl $u -o install.master.yaml
```

