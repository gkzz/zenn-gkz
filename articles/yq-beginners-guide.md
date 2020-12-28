---
title: "yqコマンド(jq wrapper for YAML)使い方備忘録"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [yq,yaml,cli,kubernetes]
published: true
---

## 0.はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。本記事は、jsonをいいかんじに出力したり、加工できるjqコマンドのラッパーのyqコマンドの使い方備忘録です。

https://twitter.com/gkzvoice/status/1342856632523362307


## 1.yqコマンドとは
- [kislyuk/yq: Command-line YAML and XML processor - jq wrapper for YAML/XML documents](https://github.com/kislyuk/yq)
- Yaml/XMLをgrepみたいに抽出
- いいかんじに整形もしてくれる

なので、数千行のmanifestやplaybookに対してgrepしたり、Gitlab Runner上でmanifestの一部をsedしていたことをyqコマンドでシュッとすることもできます。いいことずくめのyqコマンドなのですが、いざ触ってみたら、以下のような問題点を感じました。

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
- pip 20.3.3 
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
※pythonの仮想環境ツールのvenv上で検証を進めますが、直接`pip install yq`でも大丈夫です。ご自身の環境に併せてyqをご用意ください。

```
# 仮想環境ツールのvenvを使って38という仮想環境を構築
$ python3 -m venv 38python3 -m venv 38
$ source 38/bin/activate
# pipをアップグレードしてからyqをインストール
(38) $ pip install --upgrade pip
(38) $ pip install yq
```

以降、本記事ではpython3.8の仮想環境上でyqを扱います。便宜上、仮想環境上で任意の操作を行った場合は、ターミナルの左端に仮想環境名を併記します。

```
(38) $ python --version
Python 3.8.5
```

## 5.yqでyamlを生成
yamlから値を取り出すことより、yaml形式に出力することのほうがカンタンなので、そちらからやりましょう。ここでは出力結果をyqで操作するyamlへリダイレクトします。

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
  - **`[]`** と何も指定しなくてOK
```
(38) $ yq -r '.foo[].bar' input03.yml
dummy0
dummy1
```

- **`[]`** をつけないとエラー
```
(38) $ yq -r '.foo.bar' input03.yml
jq: error (at <stdin>:1): Cannot index array with string "bar"
```

- fooの0番目のbarをkeyとしたときのvalueを取得したい場合
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
- https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml

ここではmanifestの上部30行を扱います。

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

### 6-5-1. 入れ子構造のdictに対して.key.keyで取得
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

### 6-5-2. めんどくさい場所にあるdictのvalueをkey指定で取得
たとえば、以下のJSONPathというkeyを指定して、そのvalueを取得するにはどうすればよいでしょうか？
```
(38) $ grep JSONPath install.master.head30.yaml 
  - JSONPath: .status.sync.status
  - JSONPath: .status.health.status
  - JSONPath: .status.sync.revision
```

JSONPathは.spec.additionalPrinterColumnsのひとつのlistのなかの複数のdictのkeyです。
```.yaml
# install.master.head30.yamlから抜粋

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
```
いきなりJSONPathを取ろうとするとたいへんなので、以下の2つに分けて考えましょう。

- a) **`.spec.additionalPrinterColumns`** という1個のリストを取得
- b) 上で取得したリストは複数のdictなので、N番目のdictのJSONPathというkeyを指定してvalueを取得

それではやってみましょう。

#### a) **`.spec.additionalPrinterColumns`** という1個のリストを取得
```
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
できましたね。

#### b) 上で取得したリストは複数のdictなので、N番目のdictのJSONPathというkeyを指定してvalueを取得
N番目のdictはどうやって取るのでしょうか？
**`.spec.additionalPrinterColumns`** に続けて **`.JSONPath`** とkeyを指定してもエラーとなってしまいます。
```
(38) $ yq -r '.spec.additionalPrinterColumns.JSONPath' install.master.head30.yaml
jq: error (at <stdin>:1): Cannot index array with string "JSONPath"
```

実は既に **`6-2.dictのvalueが複数のdictである場合`** でやっています。
> [N]などと取得したいリストのN番目を指定してから、続けて取得したいvalueのkeyを指定

**`.spec.additionalPrinterColumns`** で複数のdict群を取得できることは分かっています。たとえば、0番目のdictはどうでしょう？
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
そうなんです。dictのvalueがlistなので、yqで指定するパスをこのようにN番目のdictと指定する必要があります。
```
- .spec.additionalPrinterColumns.JSONPath
+ .spec.additionalPrinterColumns[0].JSONPath
```

よって、JSONPathの場合はこのようになります。

```
# []と無指定だと、全て取得
(38) $ yq -r '.spec.additionalPrinterColumns[].JSONPath' install.master.head30.yaml
.status.sync.status
.status.health.status
.status.sync.revision

# [0]は0番目
(38) $ yq -r '.spec.additionalPrinterColumns[0].JSONPath' install.master.head30.yaml
.status.sync.status

# [1]は1番目
(38) $ yq -r '.spec.additionalPrinterColumns[1].JSONPath' install.master.head30.yaml
.status.health.status

# [2]は2番目
(38) $ yq -r '.spec.additionalPrinterColumns[2].JSONPath' install.master.head30.yaml
.status.sync.revision
```

## 7.yamlからvalueを指定して逆引きっぽく使う
さて、ここまでの書き方は、あらかじめ取得したいvalueの位置を知っている必要があります。これはめんどくさいです。たとえば、JSONPathが **`.status.sync.revision`** であるdictを取得するには、JSONPathのパス（位置）を事前に知っておかなければ、yqで使うことが出来ません。

```
# .spec.additionalPrinterColumns[2]と2番目と指定しなければならない
(38) $ yq -r '.spec.additionalPrinterColumns[2].JSONPath' install.master.head30.yaml
.status.sync.revision
```

**`取得したいvalueは分かっているけど、それがどこにあるのか分からない。`**
こういう場合、yqではどういった書き方をすればよいでしょうか。いくつか方法はあると思いますが、ここでは select(boolean)を使った方法をご紹介します。

先にお伝えすると、valueを指定してkeyで引っ張るといった逆引きのような書き方です。スマートな逆引きではないです。対象箇所を抽出した後、任意のvalueであるかどうか条件判定をするといった書き方です。

結論としては、このような書き方となります。
```
(38) $ yq -r '.spec.additionalPrinterColumns[] | select(.JSONPath==".status.sync.revision")' install.master.head30.yaml 
{
  "JSONPath": ".status.sync.revision",
  "name": "Revision",
  "priority": 10,
  "type": "string"
}
```
ポイントは以下の2点です。

- 7-1.selectする対象を全指定
- 7-2.select(.key=="value")なboolean


### 7-1.selectする対象を全指定
- 解説はすでに **`6-4.dictのvalueが複数のdictの場合`** でしているので割愛します。
```
$  yq -r '.spec.additionalPrinterColumns[]' install.master.head30.yaml 
{
  "JSONPath": ".status.sync.status",
  "name": "Sync Status",
  "type": "string"
}
{
  "JSONPath": ".status.health.status",
  "name": "Health Status",
  "type": "string"
}
{
  "JSONPath": ".status.sync.revision",
  "name": "Revision",
  "priority": 10,
  "type": "string"
}
```
このように対象範囲をごっそり取った後、後述する条件判定文で使うので、スマートな方法ではないと思います。

### 7-2.select(.key=="value")なboolean
- 検証用yamlを用意
```
(38) $ yq -ry '.spec.additionalPrinterColumns[] | select(.JSONPath==".status.sync.revision")' install.master.head30.yaml > input04.yml
(38) $ cat input04.yml 
JSONPath: .status.sync.revision
name: Revision
priority: 10
type: string
```

- **`select(.JSONPath==".status.sync.revision") `** がTrueである場合、後続の処理が実行される
```
(38) $ yq -r 'select(.JSONPath==".status.sync.revision") | {"name": .name}' input04.yml 
{
  "name": "Revision"
}
```
- Falseである場合、後続の処理は実行されない
```
(38) $ yq -r 'select(.JSONPath==".status.sync.hoge") | {"name": .name}' input04.yml
```
select(.key=="value")の書き方の解説は、以下を参考にしました。
- [jqで階層構造になったデータから特定の値を持った、特定の階層のデータを抽出するとき - つれづれ日記](https://diary.sshida.com/20150503-3-jq%E3%81%A7%E9%9A%8E%E5%B1%A4%E6%A7%8B%E9%80%A0%E3%81%AB%E3%81%AA%E3%81%A3%E3%81%9F%E3%83%87%E3%83%BC%E3%82%BF%E3%81%8B%E3%82%89%E7%89%B9%E5%AE%9A%E3%81%AE%E5%80%A4%E3%82%92%E6%8C%81%E3%81%A3%E3%81%9F%E3%80%81%E7%89%B9%E5%AE%9A%E3%81%AE%E9%9A%8E%E5%B1%A4)

もっといいかんじな書き方があるかもしれませんが、ひとまずこんなかんじです。
```
(38) $ yq -r '.spec.additionalPrinterColumns[] | select(.JSONPath==".status.sync.revision")' install.master.head30.yaml 
{
  "JSONPath": ".status.sync.revision",
  "name": "Revision",
  "priority": 10,
  "type": "string"
}
```

## 8.keyを指定してvalueを書き換える
jq同様、指定したkeyのvalueを書き換えることが出来ます。
sedでもできますが、入れ子になっている箇所を書き換えることは難があります。

```
$ cat input04.sed.yml 
JSONPath: .status.sync.revision
name: Revision
priority: 10
type: string
(38) $ sed -i -e 's|priority: 10|priority: 11|' input04.sed.yml 
(38) $ cat input04.sed.yml 
JSONPath: .status.sync.revision
name: Revision
priority: 11
type: string
```


```
(38) $ yq -ry '.priority|=11' input04.yq.yml > input04.yq2.yml
$ cat input04.yq2.yml 
JSONPath: .status.sync.revision
name: Revision
priority: 11
type: string
(38) $ yq -ry '.priority|=11' input04.yq.yml 
JSONPath: .status.sync.revision
name: Revision
priority: 11
type: string

(38) $ cat input04.yq.yml 
JSONPath: .status.sync.revision
name: Revision
priority: 10
type: string
```


個人的にはyqのほうがsedより可読性が高いのではないか？と思います。書き換える対象が任意のvalueであれば、yqでできないか検討してみてはいかがでしょうか。
yqコマンド(jq wrapper for YAML)使い方備忘録は以上です。
最後に、本記事を書くにあたって参考にした記事をご紹介します。

## 参考
- [kislyuk/yq: Command-line YAML and XML processor - jq wrapper for YAML/XML documents](https://github.com/kislyuk/yq)
- [jqで階層構造になったデータから特定の値を持った、特定の階層のデータを抽出するとき - つれづれ日記](https://diary.sshida.com/20150503-3-jq%E3%81%A7%E9%9A%8E%E5%B1%A4%E6%A7%8B%E9%80%A0%E3%81%AB%E3%81%AA%E3%81%A3%E3%81%9F%E3%83%87%E3%83%BC%E3%82%BF%E3%81%8B%E3%82%89%E7%89%B9%E5%AE%9A%E3%81%AE%E5%80%A4%E3%82%92%E6%8C%81%E3%81%A3%E3%81%9F%E3%80%81%E7%89%B9%E5%AE%9A%E3%81%AE%E9%9A%8E%E5%B1%A4)
- [jqで特定の値を書き換える - notebook](https://swfz.hatenablog.com/entry/2019/04/21/234818)

## P.S. Twitterもやってるのでフォローしていただけると泣いて喜びます:)
[@gkzvoice](https://twitter.com/gkzvoice)
