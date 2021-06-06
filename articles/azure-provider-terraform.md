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

画像は [Terraform by HashiCorp](https://www.terraform.io/) を参考に筆者作成。アイコンの取得先は 「x. 参考資料」 にて記載。

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

※Azureサブスクリプションはすでに作成しているものとします。サブスクリプション自体もTerraformで作れればよかったのですが、チカラ及ばず。。

参考：[追加 Azure サブスクリプションの作成 | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/cost-management-billing/manage/create-subscription)

### 4. Terraform applyするまでに必要なやることリスト

- Terraformのインストール(Terraformのバージョンをアップグレード)
- Azure CLIのインストール
  - azコマンドに認証情報を渡す
- terraform init (Azure Provider Pluginのインストール)
- terraform plan (Dry Run/tfで書いていることが実行できるか確認)
- terraform apply (Wet Run/デプロイ)

### 5. Terraformのインストール

#### 5-1. tfenvを使ってTerraformのバージョンをアップグレード

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

### 6. Azure CLIのインストール

Azure cliをインストールするとazコマンドを使うことができます。このazコマンドがTerraformを介してAzureのサービスを操作するためには必要です。azコマンドはGCPの **`gcloud`** コマンド、AWSの **`aws-cli`** と同じようなものと言ってよいでしょう。

https://twitter.com/tofuoyaco/status/1400097816542806017?s=20

```
$ curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```
参考：[Install the Azure CLI for Linux manually | Microsoft Docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux)

続いて、Azure CLIをインストールして使えるようになったazコマンドに認証情報を渡していきます。ここでは2種類ご紹介します。

#### 6-1. azコマンドに認証情報を渡す方法その1(az loginコマンド)

```
$ az login
The default web browser has been opened at 略

上記のようにターミナルに表示された後、ブラウザが起動します。
AzureにログインするMicorsoft Accountを選択するか、作成するように求められ、淡々と認証手続きを進めます。
画面の指示に従えば、認証手続きを終えることが出来るはずなので、具体的なやり方は割愛します。
```
参考：[Sign in with the Azure CLI | Microsoft Docs](https://docs.microsoft.com/ja-jp/cli/azure/authenticate-azure-cli)

※以下のスクリーンショットの画面上部は **`az login`** コマンドを実行したときのターミナル、画面下部は同コマンドを実行してからブラウザが起動したときのものです。

![](https://storage.googleapis.com/zenn-user-upload/a47d25d1cde316ce62402c9e.png)

##### az loginは何をしているの？
Azureの公式ドキュメントの [Sign in with the Azure CLI | Microsoft Docs](https://docs.microsoft.com/ja-jp/cli/azure/authenticate-azure-cli) を参照してみましょう。ご参考までにGoogle翻訳で日本語にしたものを抜粋します。

> Azure CLI にはいくつかの認証の種類があります。 開始する最も簡単な方法は、自動的にログインする Azure Cloud Shell を使用することです。ローカルでは、az login コマンドを使用して、ブラウザーから対話的にサインインできます。


#### 6-2. azコマンドに認証情報を渡す方法その2(az account set --subscription="SUBSCRIPTION_ID")

**`az login`** 以外にもazコマンドに認証情報を渡す方法はあります。それはazコマンドに紐づけたいサブスクリプションを指定するというものです。こちらを採用する場合、事前にAzure PortalなどでサブスクリプションIDを調べる必要がありますが、ブラウザを立ち上げることが難しい、JenkinsやGitlab Runner、Circle CIのjobにおいては重宝される方法ではないでしょうか。

```
$ az account set --subscription="SUBSCRIPTION_ID"
```

参考：[Azure Provider: Authenticating via the Azure CLI | Guides | hashicorp/azurerm | Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/azure_cli)


### 6-3. azコマンドに認証情報を渡したかどうか確認する方法

左記のドキュメントで紹介されていた、**`az account show`** を実行してサブスクリプション名が返ってくれば、azコマンドに認証情報を渡すことができています。

```
$ az account show | jq -r '. | {environmentName: .environmentName, name: .name}'{
  "environmentName": "AzureCloud",
  "name": "<サブスクリプション名>"
}
```

無事azコマンドをゲットして認証もできましたね。これでTerraformからAzureのサービスを操作するための下準備は終わりました。それではTerraformでVMをデプロイしてみましょう。

----

### 7. VMをデプロイするmain.tfを作る

#### 7-1. 下記のサンプルコードを使ってmain.tfを作る

- [Azure Provider: Authenticating via the Azure CLI | Guides | hashicorp/azurerm | Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/azure_cli#configuring-azure-cli-authentication-in-terraform) を参考にAzure Provider Pluginのバージョン要件を以下のように記載します。

```
terraform {
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
```
- 続いて、[Azure Provider: Authenticating via the Azure CLI | Guides | hashicorp/azurerm | Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/azure_cli#configuring-azure-cli-authentication-in-terraform) を参考にデプロイするVMのresourceを書いていく
  - resourceの意味についてはTerraformの公式ドキュメントに説明があったのでそちらをあたってほしいが、VMや仮想ネットワークなどAzureやAWSが提供するサービス群やそれらを構成する要素のことを指していると言ってよいと思う。
  - 参考：[Resources Overview - Configuration Language - Terraform by HashiCorp](https://www.terraform.io/docs/language/resources/index.html)
- 先ほど書いたものとまとめると以下のようになる

:::details main.tf
```
## 「Azure Provider Plugin」のバージョンを指定しておく
## すると，"terraform init"した際、指定したバージョンが以下の".terraform/providers/registry.terraform.io/"配下にインストールされる
## https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/azure_cli#configuring-azure-cli-authentication-in-terraform
terraform {

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=2.46.0"
    }
  }
}

provider "azurerm" {
  features {}

  ## 事前にazコマンドやAzureポータルで確認しておく
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
}

resource "azurerm_resource_group" "main" {
  name     = "${var.prefix}-resources"
  location = var.location

  tags = {
    environment = "${var.environment}"
  }
}

resource "azurerm_virtual_network" "main" {
  name                = "${var.prefix}-network"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  tags = {
    environment = "${var.environment}"
  }
}

resource "azurerm_subnet" "internal" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]

  # ココに書いてはダメ
  # tags = {
  #  environment = "${var.environment}"
  # }
}

resource "azurerm_network_interface" "main" {
  name                = "${var.prefix}-nic"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  ip_configuration {
    name                          = "ipconfig"
    subnet_id                     = azurerm_subnet.internal.id
    private_ip_address_allocation = "Dynamic"
  }

  tags = {
    environment = "${var.environment}"
  }
}

resource "azurerm_windows_virtual_machine" "main" {
  name                = "${var.prefix}-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_F2"
  admin_username      = "${var.admin_username}"
  admin_password      = "${var.admin_password}"
  network_interface_ids = [
    azurerm_network_interface.main.id,
  ]

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2019-Datacenter"
    version   = "latest"
  }

  os_disk {
    storage_account_type = "Standard_LRS"
    caching              = "ReadWrite"
  }

  tags = {
    environment = "${var.environment}"
  }
}
```
:::

- サブスクリプションIDやパスワードなどは変数化しておく

:::details variable.tf
``` 
variable "prefix" {
  default = "tf"
}

variable "environment" {
  default = "dev"
}

variable "hostname" {
  default = "tf-vm"
}

variable "subscription_id" {

}

variable "tenant_id" {

}

variable "location" {
  default = "eastus"
}

variable "admin_username" {

}

variable "admin_password" {

}
```
:::

- variable.tfにdefault値として定義するのもありだけど、terraform.tfvarsに書いておいてterraform.tfvarsはgitの管理下から外すのもアリ
  - "fix_me"と書いてあるところをazコマンドやAzureポータルで確認して修正
  - なお、admin_usernameとadmin_passwordは、デプロイしたVMにログインする際に使うユーザー名とパスワードであり、こちらは任意に決める

:::details terraform.tfvars

```
subscription_id="fix_me"
tenant_id="fix_me"
location="fix_me"
admin_username="fix_me"
admin_password="fix_me"
```
:::



#### 7-2. terraform init (Azure Provider Pluginのインストール)

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

:::message alert

**「Warning: Provider source not supported in Terraform」というエラーを引いた場合の対処方法**
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
**「5-1. tfenvを使ってTerraformのバージョンをアップグレード」** で紹介している方法を使ってTerraformのバージョンを0.12.x以上に引き上げてから、改めてterraform initを実行してみてください。
:::

#### 7-3. terraform plan (Dry Run/tfで書いていることが実行できるか確認)

```
$ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

略

Plan: 5 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + admin_password            = "<your_password>"
  + admin_username            = "<your_username>"
  + azurerm_subscription_name = "<your_subcriptioname>"
  + environment               = "dev"
  + hostname                  = "tf-vm"

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
```

#### 7-4. terraform plan (Wet Run/デプロイ)

```
$ terraform apply

Plan: 5 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + admin_password            = "<your_password>"
  + admin_username            = "<your_username>"
  + azurerm_subscription_name = "<your_subcriptioname>"
  + environment               = "dev"
  + hostname                  = "tf-vm"

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

## "yes" と入力してapplyを続ける
  Enter a value: <yes>


## "yes" と入力してapplyを続ける

略

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

admin_password = "<your_password>"
admin_username = "<your_username>"
azurerm_subscription_name = "<your_subcriptioname>"
environment               = "dev"
hostname = "tf-vm"

```

無事、デプロイできたでしょうか？？

:::message alert

**パブリックIPアドレスが付与されていない！**
画面では映っていないですが、RDPのポートも開放できていない
![](https://storage.googleapis.com/zenn-user-upload/abe3cc1a86fe60e665ba2c76.png)
:::

おそらく下記のように「パブリックIPアドレス」がVMに付与されずにデプロイされてしまったのではないでしょうか？それでは、パブリックIPアドレスとRDPポートが追加されるようにtfファイルを修正しましょう。

#### 7-5. パブリックIPアドレスの付与とRDPポートの開放をするようにtfファイルを修正

- 先ほどapplyする際に使ったtfファイル（main.tf）azurerm_public_ipとazurerm_network_security_groupを追加

:::details main.tf抜粋
```

resource "azurerm_network_interface" "main" {
  name                = "${var.prefix}-nic"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  ip_configuration {
    name                          = "ipconfig"
    subnet_id                     = azurerm_subnet.internal.id
    private_ip_address_allocation = "Dynamic"
    
    ## 追加
    public_ip_address_id          = azurerm_public_ip.main.id
  
  }

  tags = {
    environment = "${var.environment}"
  }
}


## 以下を参考に追加
## https://github.com/terraform-providers/terraform-provider-azurerm/blob/master/examples/virtual-machines/virtual_machine/multiple-network-interfaces/main.tf#L35
resource "azurerm_public_ip" "main" {
  name                = "${var.prefix}-pip"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Dynamic"
  sku                 = "Basic"

  tags = {
    environment = "${var.environment}"
  }
}

## 以下を参考に追加
## https://github.com/terraform-providers/terraform-provider-azurerm/blob/master/examples/virtual-machines/virtual_machine/multiple-network-interfaces/main.tf
resource "azurerm_network_security_group" "main" {
  name                = "${var.prefix}-nsg"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ### RDPポートを開放する
  security_rule {
    name                       = "allow_RDP"
    description                = "Allow RDP access"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "3389"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = {
    environment = "${var.environment}"
  }
}

```
:::

#### 7-6. 再度terraform planとterraform apply

- terraform planで差分を確認
  - 実行結果: **`Apply complete! Resources: 2 added, 1 changed, 0 destroyed.`**
- 差分の内訳
  - add: azurerm_public_ipリソースとazurerm_network_security_groupリソースで2点
  - change: azurerm_network_interfaceリソースのpublic_ip_address_idで1点

※実行結果はterraform applyと対して変わらないので割愛

- もういちどterraform apply
  - **`Apply complete! Resources: 2 added, 1 changed, 0 destroyed.`** とplanと同じであればok

:::details terraform apply
```
$ terraform apply

略

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_network_security_group.main will be created
  + resource "azurerm_network_security_group" "main" {
      + id                  = (known after apply)
      + location            = "eastus"
      + name                = "hoge-nsg"
      + resource_group_name = "hoge-resources"
      + security_rule       = [
          + {
              + access                                     = "Allow"
              + description                                = "Allow RDP access"
              + destination_address_prefix                 = "*"
              + destination_address_prefixes               = []
              + destination_application_security_group_ids = []
              + destination_port_range                     = "3389"
              + destination_port_ranges                    = []
              + direction                                  = "Inbound"
              + name                                       = "allow_RDP"
              + priority                                   = 110
              + protocol                                   = "Tcp"
              + source_address_prefix                      = "*"
              + source_address_prefixes                    = []
              + source_application_security_group_ids      = []
              + source_port_range                          = "*"
              + source_port_ranges                         = []
            },
        ]
    }

  # azurerm_public_ip.main will be created
  + resource "azurerm_public_ip" "main" {
      + allocation_method       = "Dynamic"
      + fqdn                    = (known after apply)
      + id                      = (known after apply)
      + idle_timeout_in_minutes = 4
      + ip_address              = (known after apply)
      + ip_version              = "IPv4"
      + location                = "eastus"
      + name                    = "hoge-pip"
      + resource_group_name     = "hoge-resources"
      + sku                     = "Basic"
    }

Plan: 2 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + public_ip_id = [
      + (known after apply),
    ]

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

## "yes" と入力してapplyを続ける
  Enter a value: <yes>
  
  
Apply complete! Resources: 2 added, 1 changed, 0 destroyed.

Outputs:

admin_password = "<your_password>"
admin_username = "<your_username>"
azurerm_subscription_name = "<your_subcriptioname>"
environment               = "dev" 
hostname = "tf-vm"
public_ip_id = [
  "/subscriptions/xxxxxxxxx/resourceGroups/tf-resources/providers/Microsoft.Network/publicIPAddresses/tf-pip",
]

```
:::

### 8. お片付け

:::details $ terraform destroy
```
$ terraform destroy

略

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

略

Plan: 0 to add, 0 to change, 7 to destroy.

Changes to Outputs:
  - admin_password = "<your_password>" -> null
  - admin_username = "<your_username>" -> null
  - azurerm_subscription_name = "<your_subcriptioname>" -> null
  - environment               = "dev" -> null
  - hostname                  = "tf-vm" -> null
  - public_ip_id = [
      "/subscriptions/xxxxxxxxx/resourceGroups/tf-resources/providers/Microsoft.Network/publicIPAddresses/tf-pip",] -> null




hostname = "tf-vm"name = "Azure for Students"
hostname = "tf-vm"
public_ip_id = [
  "/subscriptions/xxxxxxxxx/resourceGroups/tf-resources/providers/Microsoft.Network/publicIPAddresses/tf-pip",
]


Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

## "yes" と入力してdestroyを続ける
  Enter a value: 

Destroy complete! Resources: 7 destroyed.
```
:::


### 9. 小ネタ

- 変数をterraform planやterraform applyを実行するときに上書きしたい

```
$ terraform plan -var 'environment=dev2'

Changes to Outputs:
  + admin_password            = "xxxxxxxxx"
  + admin_username            = "xxxxxxxxx"
  + azurerm_subscription_name = "xxxxxxxxx"
  + environment               = "dev2"
  + hostname                  = "tf-vm"
  + public_ip_id              = [
      + (known after apply),
    ]
```


### x. 接続元のIP指定(課題

手元はフルオープンのはずなので、こんな具合に動的の接続元のipを指定して絞れれば多少マシにはなるかな
```
resource "azurerm_public_ip" "こんなぐあい" {
## https://github.com/terraform-providers/terraform-provider-azurerm/blob/master/examples/virtual-machines/virtual_machine/multiple-network-interfaces/main.tf#L35
 ip="$(curl inet-ip.info)/32"
} 
```

### x. 参考資料

- Terraformの基本的な使い方
  - [Pragmatic Terraform on AWS - KOS-MOS - BOOTH](https://booth.pm/ja/items/1318735)
- 冒頭の図で使ったアイコンの取得先
  - [AWS](https://icon-icons.com/icon/aws/146074)
  - [Azure](https://icon-icons.com/icon/microsoft-azure-logo/170956)
  - [GCP](https://icon-icons.com/icon/google-cloud-logo/171058)
