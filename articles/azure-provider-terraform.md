---
title: "[Azure]TerraformでWindows Virtual Machineでデプロイするまでにおこなったこと"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure,terraform,cli]
published: false
---

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。

https://twitter.com/gkzvoice/status/1395776522112229380?s=20

- [gkzz/azure-provider-terraform: Azure Provider with Terraform](https://github.com/gkzz/azure-provider-terraform)

## 1. そもそもTerraformとは

![](https://storage.googleapis.com/zenn-user-upload/f809cb7eed8ba50e03d51f8a.png)

画像は [Terraform by HashiCorp](https://www.terraform.io/) を参考に筆者作成。アイコンは 「x. 参考資料」 にて記載。

## 2. 本記事における問題点の共有
- できあがったtfファイルはGithubに転がっているが、必要最低限のmain.tfに手を加えていく過程が紹介されている資料が少ない。
  - 本記事では、terraform applyするまでにやったこと、main.tfに手を加えていく過程を残したい
  - Terraformのバージョンを引き上げなくちゃいけなくなった場合の対処もしたのでそれも（引き上げる前のバージョンに戻すことも出来るようにする


## 3. 環境/バージョン情報

```
- Ubuntu 20.04.2 LTS (Terraform, azure-cliを使った実行環境)
  - Terraform v0.15.4
  - azure-cli 2.61.0
```

※Azureサブスクリプションはすでに作成しているものとします。サブスクリプション自体もTerraformで作れればよかったのですが、僕にはまだチカラが足りませんでした。

参考：[追加 Azure サブスクリプションの作成 | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/cost-management-billing/manage/create-subscription)

### 4. Terraform applyするまでに必要なやることリスト

### 5. Terraformのインストール

#### 5-1. Terraformのバージョンをに切り替え

後述する **`terraform init`** でAzure Provider Pluguinをダウンロードするためには、terraformのバージョンを0.12.x以上に引き上げる必要があります。

> Version 2.x of the AzureRM Provider requires Terraform 0.12.x and later.

参考：[terraform-providers/terraform-provider-azurerm: Terraform provider for Azure Resource Manager](https://github.com/terraform-providers/terraform-provider-azurerm)

指定されたterraformのバージョンより下位のバージョンを使うと、以下のようなエラーを引きます。

:::details $ terraform init
```
$ terraform init
Initializing the backend...
Initializing provider plugins...
Warning: Provider source not supported in Terraform v0.12
  on main.tf line 6, in terraform:
   6:     azurerm = {
   7:       source = "hashicorp/azurerm"
   8:       version = "=2.46.0"
   9:     }
```
:::

ここでは、バージョンを引き上げる前のバージョンに戻す選択肢も残しておきたいので、tfenvを使ってバージョンを0.12.x以上に切り替える方法を採用します。

```
## 切り替え前＝引き上げ前のバージョンを確認
$ tfenv list
* 0.12.28 (set by ${HOME}/.tfenv/version)
  0.12.5

## 引き上げ候補はv0.15.x以上とする
$ tfenv list-remote | grep 15
0.15.4

## v0.15.4にする
$ tfenv install 0.15.4

## 無事、引き上げることができた
$ tfenv list
* 0.15.4 (set by ${HOME}/.tfenv/version)
  0.12.28
  0.12.5
0.15.3

## terraform initしてから以下のコマンドを実行すると、Azure Provider Pluguinのバージョン（ここではv2.46.0）も表示される
$ terraform version
Terraform v0.15.4
on linux_amd64
+ provider registry.terraform.io/hashicorp/azurerm v2.46.0
```

### 6. azコマンド(Azure CLI)のインストール

```
$ curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

## azコマンドに認証情報を渡すために、az loginコマンドを実行。
## https://docs.microsoft.com/ja-jp/cli/azure/authenticate-azure-cli
$ az login
The default web browser has been opened at 略

などと表示された後、ブラウザが起動します。
AzureにログインするMicorsoft Accountを選択するか、作成するように求められ、淡々と認証手続きを進めます。
画面の指示に従えば、認証手続きを終えることが出来るはずなので、具体的なやり方は割愛します。
```
参考：[Install the Azure CLI for Linux manually | Microsoft Docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux)

※以下のスクリーンショットの画面上部は **`az login`** コマンドを実行したときのターミナル、画面下部は同コマンドを実行してからブラウザが起動したときのものです。

![](https://storage.googleapis.com/zenn-user-upload/a47d25d1cde316ce62402c9e.png)

#### az loginは何をしているの？
Azureの公式ドキュメントの [Sign in with the Azure CLI | Microsoft Docs](https://docs.microsoft.com/ja-jp/cli/azure/authenticate-azure-cli) を参照してみましょう。ここではGoogle翻訳で日本語にしたものを紹介します。

> Azure CLI にはいくつかの認証の種類があります。 開始する最も簡単な方法は、自動的にログインする Azure Cloud Shell を使用することです。ローカルでは、az login コマンドを使用して、ブラウザーから対話的にサインインできます。

お！？ **`Azure CLI `**！！新しい単語ですね。そもそも「Azure CLI」と「az command」は同じものなのか、あるいはどちらかがもう片方を包含しているのか、分からないですよね。Azure CLIというCLIで操作する系のものがあって、そのなかのひとつに「az command」はじめ、いろいろあるのでしょうか、、？


### Azure CLI！az commandとの違いとは！？
"azure cli v.s. az command" などとググったのですが、違いは分かりませんでした。。Twitterでボソッとつぶいやいたら教えていただきました。ありがとうございました！！！！

https://twitter.com/tofuoyaco/status/1400097816542806017?s=20

そんなAzure CLIをインストールして使えるようになるazコマンドを使った、**`azlogin`**。これの自分なりの理解はこのようなものです。

#### 僕のaz loginの理解
- GCPでいうところの **`gcloud auth login`**
- azコマンドに認証情報を渡すことで、ブラウザでAzureサービスを利用するように、CLIで同等のことが出来るようになる（間違っていたらコメントで教えてください。）

ちなみに、左記のドキュメントで紹介されていた、**`az account show`** を実行すると、サブスクリプション名が返ってくる。
```
$ az account show | jq -r '. | {environmentName: .environmentName, name: .name}'{
  "environmentName": "AzureCloud",
  "name": "<サブスクリプション名>"
}
```


無事azコマンドをゲットして認証もできましたね。これでTerraformからAzureのサービスを操作するための下準備は終わりました。それではTerraformでVMをデプロイしてみましょう。

----

### 7. VMをデプロイするmain.tfを作る（変数化はまだしない）

下記のサンプルコードを使ってmain.tfを作る

参考：[terraform-provider-azurerm/main.tf at master · terraform-providers/terraform-provider-azurerm](https://github.com/terraform-providers/terraform-provider-azurerm/blob/master/examples/virtual-machines/windows/basic-password/main.tf)

:::details main.tf
```
terraform {

  ## 「Azure Provider Plugin」のバージョンを指定しておく
  ## すると，"terraform init"した際、指定したバージョンが以下の".terraform/providers/registry.terraform.io/"配下にインストールされる
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=2.46.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "main" {
  name     = "hoge-resources"
  location = var.location
}

resource "azurerm_virtual_network" "main" {
  name                = "hoge-network"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_subnet" "internal" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]
}

resource "azurerm_network_interface" "main" {
  name                = "hoge-nic"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.internal.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_windows_virtual_machine" "main" {
  name                            = "hoge-vm"
  resource_group_name             = azurerm_resource_group.main.name
  location                        = azurerm_resource_group.main.location
  size                            = "Standard_F2"
  admin_username                  = "adminuser"
  admin_password                  = "P@ssw0rd1234!"
  network_interface_ids = [
    azurerm_network_interface.main.id,
  ]

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2016-Datacenter"
    version   = "latest"
  }

  os_disk {
    storage_account_type = "Standard_LRS"
    caching              = "ReadWrite"
  }
}
```
:::

- terraform initでAzure Provider Pluguinをインストール

:::details $ terraform init
```
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/azurerm versions matching "2.46.0"...
- Installing hashicorp/azurerm v2.46.0...
- Installed hashicorp/azurerm v2.46.0 (signed by HashiCorp)　## <--- tfファイルで指定したAzure Provider Pluginのバージョンがインストールされました

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!  ## <--- うまくいきました！

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```
:::

- terraform plan(Dry Run/terraform applyしたら、できあがる「成果物」が表示される)

```
$ terraform plan

Plan: 5 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't
guarantee to take exactly these actions if you run "terraform apply" now.

```


- terraform apply

```
$ terraform apply

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: 

## "yes" と入力してapplyを続ける
```

![](https://storage.googleapis.com/zenn-user-upload/abe3cc1a86fe60e665ba2c76.png)

```
$ terraform destroy

Plan: 0 to add, 0 to change, 5 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

略

Destroy complete! Resources: 5 destroyed.
```

### 8. 

### 9. output.tfを使う

### 10. variable.tfとtf.varsで変数化

### x. 接続元のIP指定(課題

手元はフルオープンのはずなので、こんな具合に動的の接続元のipを指定して絞れれば多少マシにはなるかな
```
resource "azurerm_public_ip" "こんなぐあい" {
## https://github.com/terraform-providers/terraform-provider-azurerm/blob/master/examples/virtual-machines/virtual_machine/multiple-network-interfaces/main.tf#L35
 ip="$(curl inet-ip.info)/32"
} 
```

### x. 参考資料

- 冒頭の図で使ったアイコンの取得先
  - [AWS](https://icon-icons.com/icon/aws/146074)
  - [Azure](https://icon-icons.com/icon/microsoft-azure-logo/170956)
  - [GCP](https://icon-icons.com/icon/google-cloud-logo/171058)