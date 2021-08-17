---
title: "[Azure]TerraformでデプロイするLinux VMイメージの情報をjqでイイカンジに調べる方法"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure,terraform,cli]
published: true
---

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。

AzureでLinux VMをデプロイする必要に迫られたので、先日書いた記事と以下のAzureのドキュメントを頼りにtfファイルの作成を進めたのですが、**`デプロイするイメージの情報、とりわけskuとversionを調べること`** に苦戦しました。

- [[Azure]TerraformでWindows Virtual Machineでデプロイするまでにおこなったこと](https://zenn.dev/gkz/articles/azure-provider-terraform)
- [Terraform を使用して Azure で Linux VM とインフラストラクチャを構成する | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/developer/terraform/create-linux-virtual-machine-with-infrastructure)


そこで、VMをデプロイするイメージの情報について調べて分かったことから、skuとversionを始めとする、デプロイするVMのイメージの情報の調べ方まで書いていきます。

## 1. 環境/バージョン情報

```
- ローカル
  - Ubuntu 20.04.2 LTS
  - azure-cli 2.61.0
  - jq-1.6
```

デプロイするディスクなど他のスペックの情報はここでは重要ではないので割愛します。

## 2. TerraformでVMをデプロイするにあたって必要な情報

 VMをデプロイするにあたって必要な情報は、以下の4点です。

| 属性(Attribute)|説明 |
|---------------|-----|
|publisher      |イメージを作成した組織です。 例: Canonical、MicrosoftWindowsServer|
|offer          |発行元によって作成された関連するイメージのグループ名です。 例: UbuntuServer、WindowsServer|
|sku            |ディストリビューションのメジャー リリースなど、プランのインスタンス。 例: 18.04-LTS、2019-Datacenter|
|version        |イメージの SKU のバージョン番号。|

参考：[CLI を使用してマーケットプレース購入プランの情報を検索および使用する - Azure Virtual Machines | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/virtual-machines/linux/cli-ps-findimage)

VMをデプロイする際に使うTerraformでは、**`"azurerm_linux_virtual_machine"リソースのsource_image_referenceブロック`** で以下のように書かれています。

:::details main.tf（抜粋）
```
resource "azurerm_linux_virtual_machine" "main" {
  # 略

  admin_ssh_key {
    # 略
  }

  os_disk {
    # 略
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }
}
```
:::

## 3. デプロイするVMのイメージの情報の調べ方

上記publisher, offer, sku, versionで指定できる値は、az vm image listコマンドで確認することが出来ます。

```
$ az vm image list --output table
You are viewing an offline list of images, use --all to retrieve an up-to-date list
Offer                         Publisher               Sku                 Urn                                                             UrnAlias             Version
----------------------------  ----------------------  ------------------  --------------------------------------------------------------  -------------------  ---------
CentOS                        OpenLogic               7.5                 OpenLogic:CentOS:7.5:latest                                     CentOS               latest
debian-10                     Debian                  10                  Debian:debian-10:10:latest                                      Debian               latest
flatcar-container-linux-free  kinvolk                 stable              kinvolk:flatcar-container-linux-free:stable:latest              Flatcar              latest
openSUSE-Leap                 SUSE                    42.3                SUSE:openSUSE-Leap:42.3:latest                                  openSUSE-Leap        latest
RHEL                          RedHat                  7-LVM               RedHat:RHEL:7-LVM:latest                                        RHEL                 latest
SLES                          SUSE                    15                  SUSE:SLES:15:latest                                             SLES                 latest
UbuntuServer                  Canonical               18.04-LTS           Canonical:UbuntuServer:18.04-LTS:latest                         UbuntuLTS            latest
WindowsServer                 MicrosoftWindowsServer  2019-Datacenter     MicrosoftWindowsServer:WindowsServer:2019-Datacenter:latest     Win2019Datacenter    latest
略  
$ 
```

デプロイするVMのイメージの情報であるpublisher, offer, sku, versionのうち、offerとpublisherはそれぞれ決まりましたね。

```
  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "???"
    version   = "???"
  }
```

skuとversionに関する情報はaz vm image listコマンドでpublisherとofferを指定することで確認できます。

```
$ az vm image list -l eastus -p Canonical -f UbuntuServer
You are viewing an offline list of images, use --all to retrieve an up-to-date list
[
  {
    "offer": "UbuntuServer",
    "publisher": "Canonical",
    "sku": "18.04-LTS",
    "urn": "Canonical:UbuntuServer:18.04-LTS:latest",
    "urnAlias": "UbuntuLTS",
    "version": "latest"
  }
]
```

**`You are viewing an offline list of images, use --all to retrieve an up-to-date list`** と **`--all`** オプションを使うと最新のイメージの情報リストが取得できるみたいなので使ってみましょう。

:::message alert
**`--all`** を使うとVMの情報がなんと778個も返ってきてしまいます、、。

```
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | \
> jq -r '. | length'
778
```

また、**`--all`** を使うと、 **`"version": "latest"`** に該当するVMの情報は出力されないようです。

```
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | \
> jq -r '.[] | select(.version=="latest") | {offer: .offer, publisher: .publisher, sku: .sku, version: .version}' 
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | \
> jq -r '.[] | select(.version|match(".*latest.*")) | {offer: .offer, publisher: .publisher, sku: .sku, version: .version}' 
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | grep latest | wc -l
0
$ 
```
:::

これは困りましたね。。

## 4. az vm image listコマンドの実行結果を絞り込みながらskuとversionの情報を調べる方法

結論としては以下のようにaz vm image listコマンドの結果をパイプでjqに渡します。

```
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | \
> jq -c 'sort_by(.sku) | reverse | limit(10; .[] | select(.sku|match(".*LTS.*")) | {sku: .sku, version: .version})'
{"sku":"18.04-LTS","version":"18.04.202107200"}
{"sku":"18.04-LTS","version":"18.04.202106220"}
{"sku":"18.04-LTS","version":"18.04.202106040"}
{"sku":"18.04-LTS","version":"18.04.202105120"}
{"sku":"18.04-LTS","version":"18.04.202105080"}
{"sku":"18.04-LTS","version":"18.04.202105010"}
{"sku":"18.04-LTS","version":"18.04.202104150"}
{"sku":"18.04-LTS","version":"18.04.202103250"}
{"sku":"18.04-LTS","version":"18.04.202103151"}
{"sku":"18.04-LTS","version":"18.04.202102240"}
$
```

## 5. skuとversionの情報を調べる方針

- “LTS”を含むsku
- skuを降順としてソート（[18|20]を含む？）
- 最大10件を出力

![](https://storage.googleapis.com/zenn-user-upload/e8ae038760876681dc4a3974.png)

調べたいskuとversionのうち、skuに対して"LTS"で検索し、かつversionで降順にソートすることで、最新のversionを選定するという方針でいくことにしました。

改めて「4. az vm image listコマンドの実行結果を絞り込みながらskuとversionの情報を調べる方法」で貼ったjqの実行結果を見ていただきたいのですが、skuは"18.04-LTS"が並んでいる一方で、versionは"18.04"の後にタイムスタンプが続いており、ユニークです。そこでskuで検索してから、versionでソートするという方針でskuとversionを選定していくこととしました。

## 6. 今回使ったjqの解説

#### sort_by(.sku) | reverse
- skuを降順でソート
- reverseをつけず、"sort_by(.sku)"のみの場合はskuを昇順でソート
  - e.g. skuを昇順でソート 

```
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | \
> jq -c 'sort_by(.sku) | limit(10; .[] | select(.sku|match(".*LTS.*")) | {sku: .sku, version: .version})'
{"sku":"12.04.3-LTS","version":"12.04.201401270"}
{"sku":"12.04.3-LTS","version":"12.04.201401300"}
{"sku":"12.04.4-LTS","version":"12.04.201402270"}
{"sku":"12.04.4-LTS","version":"12.04.201404080"}
{"sku":"12.04.4-LTS","version":"12.04.201404280"}
{"sku":"12.04.4-LTS","version":"12.04.201405140"}
{"sku":"12.04.4-LTS","version":"12.04.201406060"}
{"sku":"12.04.4-LTS","version":"12.04.201406190"}
{"sku":"12.04.4-LTS","version":"12.04.201407020"}
{"sku":"12.04.4-LTS","version":"12.04.201407170"}
$
```
#### limit(10; .[])
- 最大10件を出力する件数とする
  - e.g. 最大100件を出力件数とする 
```
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | jq -c 'sort_by(.sku) | reverse | limit(100; .[] | select(.sku|match(".*LTS.*")))' | wc -l
100
$
```

#### select(.sku|match(".*LTS.*"))
- select(.key|match("value"))で正規表現を使って絞り込む
  - e.g. skuを降順でソートして、`limit(3; .[] | select(.sku|match(".*12.*LTS.*")))`

```
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | jq -c 'sort_by(.sku) | reverse | limit(3; .[] | select(.sku|match(".*12.*LTS.*")))'
{"offer":"UbuntuServer","publisher":"Canonical","sku":"12.04.5-LTS","urn":"Canonical:UbuntuServer:12.04.5-LTS:12.04.201705020","version":"12.04.201705020"}
{"offer":"UbuntuServer","publisher":"Canonical","sku":"12.04.5-LTS","urn":"Canonical:UbuntuServer:12.04.5-LTS:12.04.201704270","version":"12.04.201704270"}
{"offer":"UbuntuServer","publisher":"Canonical","sku":"12.04.5-LTS","urn":"Canonical:UbuntuServer:12.04.5-LTS:12.04.201704170","version":"12.04.201704170"}
$
```

※ "12"をselect()の検索条件から外すと、18.04-LTSが出力される（skuを降順でソートしているので）
```
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | jq -c 'sort_by(.sku) | reverse | limit(3; .[] | select(.sku|match(".*LTS.*")))'
{"offer":"UbuntuServer","publisher":"Canonical","sku":"18.04-LTS","urn":"Canonical:UbuntuServer:18.04-LTS:18.04.202107200","version":"18.04.202107200"}
{"offer":"UbuntuServer","publisher":"Canonical","sku":"18.04-LTS","urn":"Canonical:UbuntuServer:18.04-LTS:18.04.202106220","version":"18.04.202106220"}
{"offer":"UbuntuServer","publisher":"Canonical","sku":"18.04-LTS","urn":"Canonical:UbuntuServer:18.04-LTS:18.04.202106040","version":"18.04.202106040"}
$
```

#### `jq -c` の-cオプション

- 出力結果をコンパクトにしてくれる
- `-c`オプションを付けないとjsonフォーマットで出力
```
$ jq --help | grep "compact"
  -c               compact instead of pretty-printed output;
$ az vm image list -l eastus -p Canonical -f UbuntuServer --all | jq -r 'sort_by(.sku) | reverse | limit(2; .[] | select(.sku|match(".*LTS.*")))'
{
  "offer": "UbuntuServer",
  "publisher": "Canonical",
  "sku": "18.04-LTS",
  "urn": "Canonical:UbuntuServer:18.04-LTS:18.04.202107200",
  "version": "18.04.202107200"
}
{
  "offer": "UbuntuServer",
  "publisher": "Canonical",
  "sku": "18.04-LTS",
  "urn": "Canonical:UbuntuServer:18.04-LTS:18.04.202106220",
  "version": "18.04.202106220"
}
$
```

#### 出力したいkey/valueを指定する

- `{key: .key}` とする
- このとき、 `.key` は渡される配列のパスを考えて指定
  - e.g. echoで出力する以下のjsonのうち、`{item_id: .item_id, name: .name}` のみ出力
  - `jq '.samples[] | {item_id: .item_id, name: .name}'` と出力したい箇所の手前までを `.samples[]` といった具合に指定する
```
$ echo '{"samples":[{"item_id":1,"name":"alice", "age": 10},{"item_id":2,"name":"bob", "age": 12}]}' | jq '.'
{
  "samples": [
    {
      "item_id": 1,
      "name": "alice",
      "age": 10
    },
    {
      "item_id": 2,
      "name": "bob",
      "age": 12
    }
  ]
}
$ echo '{"samples":[{"item_id":1,"name":"alice", "age": 10},{"item_id":2,"name":"bob", "age": 12}]}' | jq '.samples[] | {item_id: .item_id, name: .name}'
{
  "item_id": 1,
  "name": "alice"
}
{
  "item_id": 2,
  "name": "bob"
}
$ 
```
※ `.samples[]` の "[]" は".samples"をkeyとする配列のことを指す

```
$ echo '{"samples":[{"item_id":1,"name":"alice", "age": 10},{"item_id":2,"name":"bob", "age": 12}]}' | jq '.samples | length'
2
$ echo '{"samples":[{"item_id":1,"name":"alice", "age": 10},{"item_id":2,"name":"bob", "age": 12}]}' | jq '.samples[0]'
{
  "item_id": 1,
  "name": "alice",
  "age": 10
}
$ echo '{"samples":[{"item_id":1,"name":"alice", "age": 10},{"item_id":2,"name":"bob", "age": 12}]}' | jq '.samples[1]'
{
  "item_id": 2,
  "name": "bob",
  "age": 12
}
$
```
## 7. Terrafformで指定しなければならないpublisher, offer, sku, version全て調べて指定することが出来た


:::details main.tf（抜粋）
```
resource "azurerm_linux_virtual_machine" "main" {
  # 略

  admin_ssh_key {
    # 略
  }

  os_disk {
    # 略
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    #version   = "latest"
    version   = "18.04.202107200"
  }
}
```
:::


## P.S. Twitterもやってるのでフォローしていただけると泣いて喜びます:)

[@gkzvoice](https://twitter.com/gkzvoice)


### P.P.S. AzureでTerraformを使ってLinux VMをデプロイするサンプルコード

https://github.com/gkzz/azure-provider-terraform-linux