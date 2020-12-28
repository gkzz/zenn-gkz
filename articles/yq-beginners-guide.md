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
いいことずくめのyqコマンドなのですが、いざ触ってみたら、以下のような問題点を感じました。

## 2.本記事における問題点の共有

```
- yqはjqに比べてドキュメントの絶対数が少ない
- yqはjqのラッパーなのだからjqのオプションを叩いても微妙に違う気がする
- yqにはjqコマンドのwrapper版とそうではないものの2種類があり、ググる力が問われる
```

そこで、「ググれる」俺得なyqコマンド使い方について調べたことを本記事にまとめていくことにしました。

## 3.環境/バージョン情報

```
- Python 3.8.5
- $ pip 20.3.3 
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
※pythonの仮想環境ツールのvenv上で検証を進めますが、直接`pip install yq`でも大丈夫です。
ご自身の環境に併せてyqをご用意ください。

```
# 仮想環境ツールのvenvを使って38という仮想環境を構築
$ python3 -m venv 38python3 -m venv 38
$ source 38/bin/activate
# pipをアップグレードしてからyqをインストール
(38) $ pip install --upgrade pip
(38) $ pip install yq
```

以降、本記事ではpython3.8の仮想環境上でyqを扱います。
便宜上、仮想環境上で任意の操作を行った場合は、ターミナルの左端に仮想環境名を併記します。

```
(38) $ python --version
Python 3.8.5
```

## 5.yqでyamlを生成
yamlから値を取り出すことより、yaml形式に出力することのほうがカンタンなので、そちらからやりましょう。
ここでは出力結果をyqで操作するyamlへリダイレクトします。

```
# 6-1.キホン
(38)$ echo "{bar: dummy}" | yq -y > input00.yml
(38)$ cat input00.yml
bar: dummy

# 6-2.2重dictの場合
(38) $ echo "foo: {bar: dummy}" | yq -y > input01.yml
(38) $ cat input01.yml 
foo:
  bar: dummy

# 6-3.dictのvalueがlistの場合
(38) $ echo "foo: {bar: [dummy0, dummy1]}" | yq -y > input02.yml
(38) $ cat input02.yml
foo:
  bar:
    - dummy0
    - dummy1

# 6-4.dictのvalueが複数のdictの場合
(38) $ echo "foo: [{bar: dummy0}, {bar: dummy1}]" | yq -y > input03.yml 
(38) $ cat input03.yml 
foo:
  - bar: dummy0
  - bar: dummy1
```

## 6.yamlからkeyを指定してvalueを取得
### 6-1.キホン

- **`[必須]`** keyの直前に **`.(コロン)`** をつけること
  - コロンを付けないとcompile errorになる
```
(38) $ yq 'bar' input00.yml 
jq: error: bar/0 is not defined at <top-level>, line 1:
bar
jq: 1 compile error
```

- オプションなし
  - json形式で出力
```
(38) $ yq '.bar' input00.yml 
"dummy"
```

- **`-r`** をつけて
  - ダブルクオーテーション無し
  - >  -r output raw strings, not JSON texts;
    - 出所： [yq: Command-line YAML/XML processor - jq wrapper for YAML and XML documents](https://kislyuk.github.io/yq/)
  - yqのコマンド結果を後々使う可能性を考慮すると、ダブルクオーテーションを取り除くために-rはデフォでよさそう 
```
(38) $ yq -r '.bar' input00.yml 
dummy
```

### 6-2.2重dictの場合
- keyとkeyの間にコロンを挟む
```
(38) $ yq -r '.foo' input01.yml 
{
  "bar": "dummy"
}
(38) $ yq -r '.foo.bar' input01.yml
"dummy"
```

### 6-3.dictのvalueがlistの場合
- dictのvalueが文字列ではなく、リストである場合も変わらず **`.key.key`** でOK
```
(38) $ yq -r '.foo.bar' input02.yml
[
  "dummy0",
  "dummy1"
]
```

- listの0番目のだけを取得したい場合は **`.key.key[0]`** 
```
(38) $ yq -r '.foo.bar[0]' input02.yml
dummy0
```

- リストの1番目だけを取得したい場合は **`.key.key[1]`** 
```
(38) $ yq -r '.foo.bar[1]' input02.yml
dummy1
```
### 6-4.dictのvalueが複数のdictの場合
- foo.barの全てのvalueを取得
  - **``**と[]と何も指定しなくてOK
```
(38) $ yq -r '.foo[].bar' input03.yml
dummy0
dummy1
```

- fooの0番目のbarをkeyとしたときのvalueを取得した場合
  - [N]などと取得したいリストのN番目を指定してから、続けて取得したいvalueのkeyを指定
```
(38) $ yq -r '.foo[0].bar' input03.yml
dummy0
```

- fooの1番目のbarをkeyとしたときのvalueを取得した場合
```
(38) $ yq -r '.foo[1].bar' input03.yml
dummy1
```


### 6-5. ここまでのおさらいとして長めのyamlでやってみる
たとえば、Argo CDをインストールする際に使うmanifestを例に挙げましょう。
ここではmanifestの上部30行を扱ってみます。

```
(38) $ u=https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml \
> && curl $u -o install.master.yaml
(38) $ cat install.master.yaml | wc -l
2718
(38) $ head -n 30 install.master.yaml > install.master.head30.yaml

# "### 6-3-{数字}"としたところをyqでおもむろに取得していきます。
(38)  cat install.master.head30.yaml 
# This is an auto-generated file. DO NOT EDIT
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  labels:
    app.kubernetes.io/name: applications.argoproj.io　### 6-5-1
    app.kubernetes.io/part-of: argocd                 ### 6-5-1
  name: applications.argoproj.io
spec:
  additionalPrinterColumns:
  - JSONPath: .status.sync.status                     
    name: Sync Status                                 ### 6-5-2
    type: string                                      
  - JSONPath: .status.health.status                   
    name: Health Status                               ### 6-5-2
    type: string                                      
  - JSONPath: .status.sync.revision                   
    name: Revision                                    ### 6-5-2
    priority: 10
    type: string
  group: argoproj.io
  names:
    kind: Application
    listKind: ApplicationList
    plural: applications
    shortNames:
    - app
    - apps
    singular: application
  scope: Namespaced
```

### 6-5-1. **`.metadata.labels`** を指定

```
(38) $ yq -r '.metadata.labels' install.master.head30.yaml 
{
  "app.kubernetes.io/name": "applications.argoproj.io",
  "app.kubernetes.io/part-of": "argocd"
}

# オプションに-yもつけるとyamlフォーマットで出力される
(38) $ yq -ry '.metadata.labels' install.master.head30.yaml 
app.kubernetes.io/name: applications.argoproj.io
app.kubernetes.io/part-of: argocd
```

### 6-5-2. **`.spec.additionalPrinterColumns`** をのN番目のdictのJSONPathをkeyを指定して取得
ココで取得したいnameとは、JSONPath, name, typeで構成される複数のdict群のうち、0,1,2番目のいずれかのnameです。
問題をカンタンにするために、以下の2つに分割します。

- **`.spec.additionalPrinterColumns`** という1個のリストを取得
- 上で取得したリストは複数のdictなので、N番目のdictのnameというkeyを指定してvalueを取得

それではやってみましょう。
```
# **`.spec.additionalPrinterColumns`** という1個のリストを取得
(38) $ yq -r '.spec.additionalPrinterColumns' install.master.head30.yaml 
[
  {
    "JSONPath": ".status.sync.status",
    "name": "Sync Status",
    "type": "string"
  },
  {
    "JSONPath": ".status.health.status",
    "name": "Health Status",
    "type": "string"
  },
  {
    "JSONPath": ".status.sync.revision",
    "name": "Revision",
    "priority": 10,
    "type": "string"
  }
]

## **`yq -r`** でパースした配列の数は、jq '. | length' で取得できます
(38) $ yq -r '.spec.additionalPrinterColumns' install.master.head30.yaml | jq '. |  length'
3 
```
N番目のdictとはどうやって取るのでしょうか？
実は既に **`6-2.dictのvalueが複数のdictである場合`** でやっているんですよ。
> [N]などと取得したいリストのN番目を指定してから、続けて取得したいvalueのkeyを指定
**`.spec.additionalPrinterColumns`** で複数のdict群を取得できることは分かっています。
たとえば、0番目のdictはどうでしょう？
```
$ yq -r '.spec.additionalPrinterColumns[0]' install.master.head30.yaml
{
  "JSONPath": ".status.sync.status",
  "name": "Sync Status",
  "type": "string"
}

```
1,2番目も続けて取得してみましょう。
```
(38) $ yq -r '.spec.additionalPrinterColumns[1]' install.master.head30.yaml
{
  "JSONPath": ".status.health.status",
  "name": "Health Status",
  "type": "string"
}
(38) $ yq -r '.spec.additionalPrinterColumns[2]' install.master.head30.yaml
{
  "JSONPath": ".status.sync.revision",
  "name": "Revision",
  "priority": 10,
  "type": "string"
}
```

ココで取得したいnameは、JSONPath, name, typeで構成される複数のdict群のうち、0,1,2番目のいずれかのnameです。

key指定でvalueを取得する方法だけは、膨大な量のmanifestから任意の値を探すみたいなときにパスを指定する際、煩わしいです。
