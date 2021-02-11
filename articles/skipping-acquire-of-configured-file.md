---
title: "apt upgradeしたらSkipping acquire of configured fileと言われた"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ubuntu,apt]
published: false
---

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。apt upgradeしたら以下のようなエラー？を引きました。本記事では、その対処方法と/etc/apt/sources.list及び/etc/apt/sources.list.d/\*について調べたことを共有します。

https://twitter.com/gkzvoice/status/1359843501307957248

結論からいうと、/etc/apt/sources.list.d/\*を編集すればよかったのですが、編集用のコマンドを使うことを推奨されており、自分は危うくvimで編集してしまうところでした。ご注意ください。

## 1. エラーメッセージを引く直前に実行したコマンドとそのエラーメッセージ

```
$ sudo apt upgrade
N: Skipping acquire of configured file 'main/binary-i386/Packages' as repository 'http://apt.postgresql.org/pub/repos/apt focal-pgdg InRelease' doesn't support architecture 'i386'
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

## 2. 環境/バージョン情報
```
- ubuntu 20.04.2 LTS (Focal Fossa)
```

## 3. エラー対処方法

### 3-1. 基本方針
エラーメッセージでググって見つけたコチラの記事に従って対処。

[linux - Skipping acquire of configured file 'main/binary-i386/Packages' - Stack Overflow](https://stackoverflow.com/questions/61523447/skipping-acquire-of-configured-file-main-binary-i386-packages)

### 3-2. やったこと

- postgresqlパッケージの取得先について書かれたファイルを特定
```
$ grep "http://apt.postgresql.org/pub/repos/apt" /etc/apt/sources.list.d/*
/etc/apt/sources.list.d/pgdg.list:deb http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
/etc/apt/sources.list.d/pgdg.list:# deb-src http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
/etc/apt/sources.list.d/pgdg.list.save:deb http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
/etc/apt/sources.list.d/pgdg.list.save:#deb-src http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
```
- バックアップをとる
```
$ sudo cp /etc/apt/sources.list.d/pgdg.list /etc/apt/sources.list.d/pgdg.list.bk.$(date +"%Y%m%d%H%M%S")
$ ls /etc/apt/sources.list.d/pgdg.list.bk*
/etc/apt/sources.list.d/pgdg.list.bk.20210211231747
```

- geditで特定したファイルを編集
```
$ sudo gedit /etc/apt/sources.list.d/pgdg.list

deb [arch=amd64] http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
```
※テキストファイルがポップアップ画面で表示されて新鮮だったｗ

<img src="https://storage.googleapis.com/zenn-user-upload/f8ri2mgjtfyc0kb0p1q09ag2diu3" style="max-width:75%;">

※編集した後、以下のような　WARNINGが表示されたがスルーした。理由は注釈に記載。[^1]

```
$ sudo gedit /etc/apt/sources.list.d/pgdg.list

(gedit:26316): Tepl-WARNING **: 23:34:13.143: GVfs metadata is not supported. Fallback to TeplMetadataManager. Either GVfs is not correctly installed or GVfs metadata are not supported on this platform. In the latter case, you should configure Tepl with --disable-gvfs-metadata.
```

※ edit-sourcesではコマンドをインストールする必要があるのか使えなかった。(参照した記事が古かったので当時と状況が変わったのかも)

- diffでバックアップを取ったファイルと差分確認
```
$ sudo diff /etc/apt/sources.list.d/pgdg.list /etc/apt/sources.list.d/pgdg.list.bk.20210211231747 
1c1
< deb [arch=amd64] http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
---
> deb http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
```

- 改めてapt upgrade
```
$ sudo apt upgrade
N: Ignoring file 'pgdg.list.bk.20210211231747' in directory '/etc/apt/sources.list.d/' as it has an invalid filename extension
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
N: Ignoring file 'pgdg.list.bk.20210211231747' in directory '/etc/apt/sources.list.d/' as it has an invalid filename extension
```

- mvでバックアップファイルを適当な場所に退避させるか、rmでバックアップファイルを削除（僕は$HOMEディレクトリに退避させた）
- 改めてapt upgrade

```
$ sudo apt upgrade
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

## 4. 疑問点
- dot saveファイル(.save)はなにものか？
- /etc/apt/sources.list\*のバックアップファイルだとしたら、どういうサイクルでバックアップしているか？
- geditコマンドで/etc/apt/sources.list.d/pgdg.listを編集した直後は.saveでバックアップをとっているような様子は確認できなかった
```
$ sudo ls -la /etc/apt/sources.list.d/pgdg.list*
[sudo] password for gkz: 
-rw-r--r-- 1 root root 141 Feb 11 23:34 /etc/apt/sources.list.d/pgdg.list
-rw-r--r-- 1 root root 127 Jan 28 19:35 /etc/apt/sources.list.d/pgdg.list.save

$ sudo diff /etc/apt/sources.list.d/pgdg.list /etc/apt/sources.list.d/pgdg.list.save 
1,2c1,2
< deb [arch=amd64] http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
< # deb-src http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
---
> deb http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
> #deb-src http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
```

## 参考
- [linux - Skipping acquire of configured file 'main/binary-i386/Packages' - Stack Overflow](https://stackoverflow.com/questions/61523447/skipping-acquire-of-configured-file-main-binary-i386-packages)
- [【 apt 】コマンド／【 apt-mark 】コマンド――パッケージを一括更新する：Linux基本コマンドTips（141） - ＠IT](https://www.atmarkit.co.jp/ait/articles/1709/07/news016.html)
- [【 sudo 】コマンド――スーパーユーザー（rootユーザー）の権限でコマンドを実行する：Linux基本コマンドTips（68） - ＠IT](https://www.atmarkit.co.jp/ait/articles/1611/28/news036.html)

## P.S. Twitterもやってるのでフォローしていただけると泣いて喜びます:)
[@gkzvoice](https://twitter.com/gkzvoice)


[^1]: 参考にしたStack Overflowの記事では、「ignore any warning messages」と-HオプションをつけてWARNINGをignoreしていた。sudo -Hは「環境変数「HOME」をrootユーザーのホームディレクトリに変更してコマンドを実行する」オプションと記載されています。『[【 sudo 】コマンド――スーパーユーザー（rootユーザー）の権限でコマンドを実行する：Linux基本コマンドTips（68） - ＠IT](https://www.atmarkit.co.jp/ait/articles/1611/28/news036.html)』より。