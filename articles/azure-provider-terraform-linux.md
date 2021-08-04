---
title: "Azureのブート診断の有効化をTerraformだけで完結できないからGUIとの合わせ技でゴリ押しする"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure,terraform,cli]
published: false
---

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。

AzureでLinux VMをデプロイする必要に迫られ、以下2つの課題に直面しましたが、無事デプロイすることが出来ました。

- デプロイしたいVMのskuを調べる
- **`Azureのブート診断の有効化をTerraformだけで完結したい`**

本記事では後者の課題に対してどういうアプローチを取ったか？書いていきます。前者の課題に対してどう取り組んだか？については、[[Azure]TerraformでLinux VMをデプロイするイメージのバージョンをjqでイイカンジに調べる方法](https://zenn.dev/gkz/articles/azure-provider-terraform-jq)をご参照ください。


![](https://storage.googleapis.com/zenn-user-upload/7471e17e1d706c920d0534bb.png)

:::details terraform applyしたときのログ（抜粋）

```
$ terraform apply

略

Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:

admin_username = "*********"
azurerm_storage_account_name = "diag*********"
azurerm_subscription_name = "*********"
environment = "dev"
hostname = "tf-vm"
public_ip_id = [
  "/subscriptions/*********/resourceGroups/tf-resources/providers/Microsoft.Network/publicIPAddresses/tf-pip",
]
ssh_pub_key_path = "*********"
$
```
:::

## 1. そもそも「Azureのブート診断」とは
Azureのドキュメントでは、このような説明がされています。

> ブート診断は、VM ブート エラーの診断を可能にする、Azure の仮想マシン (VM) のデバッグ機能です。 ブート診断を使用すると、ユーザーは、シリアル ログ情報とスクリーンショットを収集して、起動中の VM の状態を確認できます。

参考：[Azure のブート診断 - Azure Virtual Machines | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/virtual-machines/boot-diagnostics)

### 1-1.ユースケース（予想）
実務で使ったことがなく、個人の検証環境で使ってみたという前提での予想ですが、このようなユースケースは考えられるのではないでしょうか。

- **`起動しているはずだけどVMにリモートアクセスできない`** といった際にデバッグし、原因の究明に役立てる

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

- [Configure a Linux VM with infrastructure in Azure using Terraform | Microsoft Docs](https://docs.microsoft.com/en-us/azure/developer/terraform/create-linux-virtual-machine-with-infrastructure) を参考にtfファイルを作成
- ポイントは以下の2点

#### [ポイント1]"azurerm_linux_virtual_machine"リソースで **`boot_diagnostics`** でstorage_account_uriを指定し、**`enabled=true`** とする

:::details main.tfから"azurerm_linux_virtual_machine"リソースを抜粋
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
        enabled = "true"
        storage_account_uri = azurerm_storage_account.main.primary_blob_endpoint
  }
}
```
:::

#### [ポイント2]"azurerm_storage_account"リソースと"random_id"リソースを使う

:::messages
"azurerm_storage_account"リソースのnameはランダム値としたいので、"random_id"リソースも使う
:::

:::details main.tfから"azurerm_storage_account"リソースと"random_id"リソースを抜粋
```
略

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

※ "random_id"リソースは"azurerm_storage_account"リソースのnameの値を決めるために使った

https://gist.github.com/archmangler/4f99d94c230dfe7f01a8e6997f9a7eb9


:::message alert
**`enabled = "true"`** を書いてterraform planしたらエラー
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


## 4. "azurerm_linux_virtual_machine"リソースでenabled=trueとすることができないか調べたこと

- [Azure のブート診断 - Azure Virtual Machines | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/virtual-machines/boot-diagnostics#enable-managed-boot-diagnostics-using-azure-resource-manager-arm-templates) によると、ARMでは使われている
  - ※手元で未検証

```
 "diagnosticsProfile": {
     "bootDiagnostics": {
         "enabled": true
      }
 }
```

- [azurerm_virtual_machine | Resources | hashicorp/azurerm | Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/virtual_machine) で書かれている以下のことはどういうことだろう？まだenabledが書くことが出来るようになっていないということなのか？
  - > A boot_diagnostics block supports the following:
    >
    > ・enabled - (Required) Should Boot Diagnostics be enabled for this Virtual Machine?
    >
    > ・storage_uri - (Required) The Storage Account's Blob Endpoint which should hold the virtual machine's diagnostic files.

## 5. TerraformだけでAzureのブート診断を有効化できないからどうしたか
TerraformとGUIの合わせ技でブート診断を有効化としました。Terraformでストレージアカウントを作成し、GUIでブート診断を有効化とするようなかんじです。

### 5-1.Terraformでストレージアカウントを作成

- "azurerm_linux_virtual_machine"リソースのboot_diagnosticsから、enabled = "true"を削除
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