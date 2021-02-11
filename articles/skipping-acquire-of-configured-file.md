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

- 基本的にはエラーメッセージでググって見つけたコチラの記事に従って対処。
 [linux - Skipping acquire of configured file 'main/binary-i386/Packages' - Stack Overflow](https://stackoverflow.com/questions/61523447/skipping-acquire-of-configured-file-main-binary-i386-packages)



## 参考
- [linux - Skipping acquire of configured file 'main/binary-i386/Packages' - Stack Overflow](https://stackoverflow.com/questions/61523447/skipping-acquire-of-configured-file-main-binary-i386-packages)
- [【 apt 】コマンド／【 apt-mark 】コマンド――パッケージを一括更新する：Linux基本コマンドTips（141） - ＠IT](https://www.atmarkit.co.jp/ait/articles/1709/07/news016.html)

## P.S. Twitterもやってるのでフォローしていただけると泣いて喜びます:)
[@gkzvoice](https://twitter.com/gkzvoice)
