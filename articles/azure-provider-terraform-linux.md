---
title: "Azureのブート診断の有効化をTerraformだけで完結できないからGUIとの合わせ技でゴリ押しする"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure,terraform,cli]
published: true
---

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。

AzureでLinux VMをデプロイする必要に迫られ、Azureのドキュメントを読み漁っていたところ、「ブート診断」なるものを知りました。後述しますが、このブート診断はかゆいところに手が届くサービスだと感じたので、terraformでブート診断の設定を試みることにしました。

## 1.本記事における問題意識の共有
表題のとおり、terraformだけでブート診断の設定をおこなうことはできませんでした。
そこで、本記事にterraformだけでブート診断を設定するためにおこなったこと、引いたエラー、そして今できる打ち手を書き残しておくこととします。

※ 本記事では、Terraformについては必要以上にフォーカスしません。Terraformの基本的な使い方については、手前味噌ですが、以下の記事をご参照ください。

[[Azure]TerraformでWindows Virtual Machineでデプロイするまでにおこなったこと](https://zenn.dev/gkz/articles/azure-provider-terraform)

## 1. 「Azureのブート診断」を使うとできること
Azureのドキュメントでは、以下のような説明がされています。

> ブート診断は、VM ブート エラーの診断を可能にする、Azure の仮想マシン (VM) のデバッグ機能です。 ブート診断を使用すると、ユーザーは、シリアル ログ情報とスクリーンショットを収集して、起動中の VM の状態を確認できます。

参考：[Azure のブート診断 - Azure Virtual Machines | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/virtual-machines/boot-diagnostics)

ブート診断を使うと、以下のようなログなどが確認できます。

- VMをデプロイした後に表示されるVMのログイン画面のスクリーンショット(以下の画像の左側)
- VMをデプロイするまでのシリアルログ(以下の画像の右側)
  - ログの末尾には **`successfully running`** と出力されている

![](https://storage.googleapis.com/zenn-user-upload/7471e17e1d706c920d0534bb.png)

### 1-1.ユースケース（予想）

実務で使ったことがなく、個人の検証環境で使ってみたという前提での予想ですが、以下のようなユースケースは考えられるのではないでしょうか？

- **`起動しているはずだけどVMにリモートアクセスできない`** といった際にデバッグし、原因の究明に役立てる

### 1-2.VMのログイン画面のスクリーンショットとデプロイするまでのログの格納場所
VMのログイン画面のスクリーンショットとデプロイするまでのログの格納場所は、Azure Storage アカウントです。

なお、IaaSやオンプレミスでVMをデプロイする際にも聞く「ディスク」はAzure Storage アカウントとは別物です。「ディスク」は以下の画像でいう[「管理ディスク」](https://docs.microsoft.com/ja-jp/azure/virtual-machines/managed-disks-overview)にあたります。

![](https://storage.googleapis.com/zenn-user-upload/fba8956580712b94ee58aae9.png)

画像の出所: [Azure での Windows VM の実行 - Azure Reference Architectures | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/architecture/reference-architectures/n-tier/windows-vm) （赤い枠は筆者が編集）

## 2. 環境情報

```
- ローカル（Terraform実行環境）
  - Ubuntu 20.04.2 LTS
  - Terraform v1.0.0
  - azure-cli 2.61.0

- TerraformでデプロイするVM
　 - Ubuntu 18.04
   - Standard F2
```

## 3. TerraformだけでAzureのブート診断を有効化するためにtfファイルに書いたこと

- 以下を参考にtfファイルを作成
  - [Configure a Linux VM with infrastructure in Azure using Terraform | Microsoft Docs](https://docs.microsoft.com/en-us/azure/developer/terraform/create-linux-virtual-machine-with-infrastructure) 
  - [terraform azure boot_diagnostics](https://gist.github.com/archmangler/4f99d94c230dfe7f01a8e6997f9a7eb9)
- 作成したtfファイル
  - https://github.com/gkzz/azure-provider-terraform-linux/blob/main/main.tf
- ポイントは以下の2点

#### [ポイント1]ブート診断のスクリーンショットやシリアルログを用意するために"azurerm_storage_account"リソースを使う

:::messages
- ストレージアカウントはオブジェクトストレージなので、名前はユニークとしなければならない。
- そこで"random_id"リソースを使ってストレージアカウントの名前を生成することとした。(*1)

> ストレージ アカウントでは、世界中のどこからでも HTTP または HTTPS 経由でアクセスできる Azure Storage データ用の一意の名前空間が提供されます。

参考:[ストレージ アカウントの作成 - Azure Storage | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-account-create?tabs=azure-portal)
:::

```code: main.tfより抜粋

resource "azurerm_storage_account" "main" {
    ## *1: "random_id"リソースを使ってストレージアカウントの名前を生成
    ## ここでは"boot diagnostics"の"diag"をprefixとしている
    name                        = "diag${random_id.main.hex}"
    location            = azurerm_resource_group.main.location
    resource_group_name         = azurerm_resource_group.main.name
    account_replication_type    = "LRS"
    account_tier                = "Standard"
}

resource "random_id" "main" {
    keepers = {
        # Generate a new ID only when a new resource group is defined
        resource_group = azurerm_resource_group.main.name
    }

    byte_length = 8
}
```
参考：https://github.com/gkzz/azure-provider-terraform-linux/blob/main/main.tf#L171:#L191

#### [ポイント2]"azurerm_linux_virtual_machine"リソースのboot_diagnosticsブロックにて、ポイント1で作成したストレージアカウントを指定

```code: main.tfより抜粋

resource "azurerm_linux_virtual_machine" "main" {
  name                = "${var.prefix}-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_F2"
  admin_username      = "${var.admin_username}"
  network_interface_ids = [
    azurerm_network_interface.main.id,
  ]

  admin_ssh_key {
    #略
  }

  os_disk {
    #略
  }

  source_image_reference {
    #略
  }

  boot_diagnostics {
        enabled = "true"
        storage_account_uri = azurerm_storage_account.main.primary_blob_endpoint
  }
}
```
参考：https://github.com/gkzz/azure-provider-terraform-linux/blob/main/main.tf#L66:#L169

それでは、tfファイルが書けているか確認するためにterraform planしてみましょう。

## 3. terraform planしてみると、、、

エラーとなってしまいました。以下のエラーメッセージを読むと原因は
"azurerm_linux_virtual_machine"リソースのboot_diagnosticsブロックで、**`enabled = "true"`** と書いてしまったことと考えてよさそうです。

:::message alert
```
$ terraform plan
╷
│ Error: Unsupported argument
│ 
│   on main.tf line 99, in resource "azurerm_linux_virtual_machine" "main":
│   99:         enabled = "true"
│ 
│ An argument named "enabled" is not expected here.
╵
$ 
```
:::

```code: main.tfより抜粋

resource "azurerm_linux_virtual_machine" "main" {
  name                = "${var.prefix}-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_F2"
  admin_username      = "${var.admin_username}"
  network_interface_ids = [
    azurerm_network_interface.main.id,
  ]

  #略

  boot_diagnostics {
        enabled = "true"
        storage_account_uri = azurerm_storage_account.main.primary_blob_endpoint
  }
}
```


## 4. "azurerm_linux_virtual_machine"リソースでenabled=trueとすることができないか調べたこと

- Azureのドキュメントによると、Azure Resource Manager (ARM) テンプレートでは使うことができるみたい
  - ※ 手元で未検証

```
 "diagnosticsProfile": {
     "bootDiagnostics": {
         "enabled": true
      }
 }
```
参考：[Azure のブート診断 - Azure Virtual Machines | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/virtual-machines/boot-diagnostics#enable-managed-boot-diagnostics-using-azure-resource-manager-arm-templates)

- Terraformのドキュメントでも、boot_diagnosticsブロックは**`enabled`**をサポートしていると記載されていた

  - > A boot_diagnostics block supports the following:
    >
    > ・enabled - (Required) ty
    >
    > ・storage_uri - (Required) The Storage Account's Blob Endpoint which should hold the virtual machine's diagnostic files.

参考：[azurerm_virtual_machine | Resources | hashicorp/azurerm | Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/virtual_machine)

::: messages
- 調べた結果、Terraformにおけるboot_diagnosticsブロックの書き方は合っていると考えてよさそう
- しかし、先述したとおりエラーの原因は **`enabled = "true"`** と書いてしまったことというのも合っているはず
- 今の段階ではTerraformでブート診断を有効化することは出来ないと考え、issueをあげた
  - [Cannot get boot_diagnostics enabled since boot_diagnostics block seems NOT support "enabled=true" · Issue #13023 · hashicorp/terraform-provider-azurerm](https://github.com/hashicorp/terraform-provider-azurerm/issues/13023)

:::

## 5. TerraformだけでAzureのブート診断を有効化できないからどうしたか
TerraformとGUIの合わせ技でブート診断を有効化としました。Terraformでストレージアカウントを作成し、GUIでブート診断を有効化とするようなかんじです。

### 5-1.Terraformでストレージアカウントを作成

- "azurerm_linux_virtual_machine"リソースのboot_diagnosticsブロックから、enabled = "true"を削除
- "azurerm_storage_account"リソースと"random_id"リソースは上記で書いたものをそのまま使う

:::details main.tfから"azurerm_linux_virtual_machine"リソースと"azurerm_storage_account"リソースと"random_id"リソースを抜粋
```
略

resource "azurerm_linux_virtual_machine" "main" {
  name                = "${var.prefix}-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_F2"
  admin_username      = "${var.admin_username}"
  network_interface_ids = [
    azurerm_network_interface.main.id,
  ]

  admin_ssh_key {
    #略
  }

  os_disk {
    #略
  }

  source_image_reference {
    #略
  }

  boot_diagnostics {
        # enabled = "true"
        storage_account_uri = azurerm_storage_account.main.primary_blob_endpoint
  }
}

resource "azurerm_storage_account" "main" {
    name                        = "diag${random_id.main.hex}"
    location            = azurerm_resource_group.main.location
    resource_group_name         = azurerm_resource_group.main.name
    account_replication_type    = "LRS"
    account_tier                = "Standard"
}

resource "random_id" "main" {
    keepers = {
        # Generate a new ID only when a new resource group is defined
        resource_group = azurerm_resource_group.main.name
    }

    byte_length = 8
}
```
:::

- "azurerm_storage_account"のnameのoutputを設定(後続の設定で"azurerm_storage_account"のnameを使うので)

:::details output.tfから抜粋
```
output "azurerm_storage_account_name" {
  value = azurerm_storage_account.main.name
}

```
:::


### 5-2.GUIでブート診断を有効化

- terraform apply後、出力される「ストレージアカウント」を確認するでデプロイしたVMの画面左のタブを下にスクロールして、「診断設定」をクリック

![](https://storage.googleapis.com/zenn-user-upload/0334a68171f0a9215d42e247.png)

- 「ストレージアカウントを選択します」の下のタブから"azurerm_storage_account"のnameを選択
  - **`prefixが"diag"となっているはず！`**
- 「ゲスト レベルの診断を有効にする」をクリック

![](https://storage.googleapis.com/zenn-user-upload/b4c4ef3f303c04b51657b463.png)


- しばらくすると以下のような画面に切り替わるので「ブート診断を表示する」をクリック
  - 画面右上のポップアップにて、ブート診断が有効となったと通知が届く
![](https://storage.googleapis.com/zenn-user-upload/915b86bc824c2102b2885430.png)

- 以下のようにVMのスクリーンショット(画面左)とログ(画面右)が表示され、ブート診断が有効となっていることが確認できる

![](https://storage.googleapis.com/zenn-user-upload/7471e17e1d706c920d0534bb.png)


## 6. 今後の展望
現時点ではブート診断の設定がTerraformだけで完結することは難しいようですが、ドキュメントが先行している？ことから、Terraformだけでブート診断の有効化ができる日はそう遠くはないのかもしれません。

もっといいブート診断の有効化の設定方法があればコメントなどで教えていただけるとうれしいです。

### P.S. AzureでTerraformを使ってLinux VMをデプロイするサンプルコード

https://github.com/gkzz/azure-provider-terraform-linux


## P.P.S. Twitterもやってるのでフォローしていただけると泣いて喜びます:)

[@gkzvoice](https://twitter.com/gkzvoice)