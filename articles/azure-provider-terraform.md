---
title: "[Azure]TerraformでWindows Virtual Machineでデプロイしてみた"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure,terraform,cli]
published: false
---

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。


- [gkzz/azure-provider-terraform: Azure Provider with Terraform](https://github.com/gkzz/azure-provider-terraform)

## 1. そもそもTerraformとは

![](https://storage.googleapis.com/zenn-user-upload/f809cb7eed8ba50e03d51f8a.png)

画像は [Terraform by HashiCorp](https://www.terraform.io/) を参考に筆者作成。アイコンは 「x. 参考資料」 にて記載。

## 2. 本記事における問題点の共有
- できあがったtfファイルはGithubに転がっているが、必要最低限のmain.tfに手を加えていく過程が紹介されている資料が少ない。
  - 本記事では、terraform applyするまでにやったこと、main.tfに手を加えていく過程を残したい
  - Terraformのバージョンを引き上げなくちゃいけなくなった場合の対処もしたのでそれも（引き上げる前のバージョンに戻すことも出来るの？

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


## 3. 環境/バージョン情報

```
- Ubuntu 20.04.2 LTS (Terraform, azure-cliを使った実行環境)
  - Terraform v0.15.4
  - azure-cli 2.24.0
```

### 4. Terraform applyするまでに必要なやることリスト

### 5. Terraformのインストール

#### 5-1. Terraformのバージョンをに切り替え

```
## 切り替え＝引き上げ前
$ tfenv list
* 0.12.28 (set by ${HOME}/.tfenv/version)
  0.12.5
## 引き上げ候補はv0.15.x
$ tfenv list-remote | grep 15
0.15.4
## v0.15.4にする
$ tfenv install 0.15.4
$ tfenv list
* 0.15.4 (set by ${HOME}/.tfenv/version)
  0.12.28
  0.12.5
0.15.3
```

### 6. azコマンドのインストール

```
$ curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
$ az login
$ az account show | jq -r '. | {environmentName: .environmentName, name: .name}'{
  "environmentName": "AzureCloud",
  "name": "Azure for Students"
}
```
参考：[Install the Azure CLI for Linux manually | Microsoft Docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux)

### 7. サブスクリプションを作るだけのmain.tfを作る

### 8. VMをデプロイするmain.tfを作る（変数化はまだしない）

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