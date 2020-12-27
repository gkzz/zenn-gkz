---
title: "yqコマンド(jq wrapper for YAML)使い方備忘録"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [yq,yqml,cli,kubernetes]
published: false
---

## 0.はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。
本記事は、jsonをいいかんじに出力したり、加工できるjqコマンドのラッパーのyqコマンドの使い方備忘録です。

https://twitter.com/gkzvoice/status/1342856632523362307


## 1.yqコマンドとは
- Yaml/XMLをgrepみたいに抽出
- いいかんじに整形もしてくれる

なので、数千行のmanifestやplaybookに対してgrepしたり、Gitlab Runner上でmanifestの一部をsedしていたことをyqコマンドでシュッとすることもできます。
いいことずくめのyqコマンドなのですが、いざ触ってみたら、問題点を感じました。

## 2.本記事における問題点の共有

```
- yqはjqに比べてドキュメントの絶対数が少ない
- yqはjqのラッパーなのだからjqのオプションを叩いても微妙に違う気がする
- yqにはjqコマンドのwrapper版とそうではないものの2種類があり、ググる力が問われる
```

そこで、「ググれる」俺得なyqコマンド使い方について調べたことを本記事にまとめようと思ったわけです。

## 3.環境/バージョン情報

```
- Python 3.8.5
- yq 2.11.1
```
### 3-1.種類のyqコマンド
さて、yqコマンドは上述したとおり2種類合って紛らわしいので本記事で扱うyqコマンドについて確認しておきましょう。

- jqのYAML/XMLラッパー
  - [kislyuk/yq: Command-line YAML and XML processor - jq wrapper for YAML/XML documents](https://github.com/kislyuk/yq)
  - **`jqコマンドのドキュメントを流用できる`**
    - e.g. [jqコマンドを使う日常のご紹介](https://qiita.com/takeshinoda@github/items/2dec7a72930ec1f658af)
- もうひとつのyqコマンド
  - [mikefarah/yq: yq is a portable command-line YAML processor](https://github.com/mikefarah/yq)


## 4.yqのインストール
※pythonの仮想環境venv上で検証を進めますが、直接`pip install yq`でも大丈夫です。
ご自身の環境に併せてyqをご用意ください。

```
$ python3 -m venv 38python3 -m venv 38
$ source 38/bin/activate
```

以降、本記事ではpython3.8の仮想環境上でyqを扱います。
便宜上、ターミナルの書式を以下のようにします。

```
(38) $ python --version
Python 3.8.5
```

## 5.yqでyamlを生成
yamlから値を取り出すことより、yaml形式に出力することのほうがカンタンなので、そちらからやりましょう。
ここでは出力結果をyqで操作するyamlへリダイレクトします。

```
(38)$ echo "{bar: dummy}" | yq -y > input00.yml
(38)$ cat input00.yml
bar: dummy
(38)$ yq -r '.bar' input00.yml 
dummy
(38) $ echo "foo: {bar: dummy}" | yq -y > input01.yml
(38) $ cat input01.yml 
foo:
  bar: dummy
```

## 6.yamlからkeyを指定して値を取り出す

- **``[必須]`** keyの直前に **`.(コロン)`** をつけること
  - コロンを付けないとcompile errorになる
```
(38) $ yq bar input00.yml 
jq: error: bar/0 is not defined at <top-level>, line 1:
bar
jq: 1 compile error
```

- オプションなし
  - json形式で出力
```
(38) $ yq .bar input00.yml 
"dummy"
(38) $ yq .foo input01.yml 
{
  "bar": "dummy"
}
```

- **`-r`** をつけて
  - ダブルクオーテーション無し
  - >  -r output raw strings, not JSON texts;
    - 出所： [yq: Command-line YAML/XML processor - jq wrapper for YAML and XML documents](https://kislyuk.github.io/yq/)
```
(38) $ yq -r .bar input00.yml 
dummy
(38) $ yq .foo input01.yml 
{
  "bar": "dummy"
}
(38) $ yq .foo.bar input01.yml
"dummy"
(38) $ yq -r '.bar' input00.yml 
dummy
```


```
$ u=https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml && curl $u -o install.master.yaml
```

