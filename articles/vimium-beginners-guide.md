---
title: "【Chrome拡張機能】Vimライクなキーバインドでブラウザを操作するVimiumのオレオレ設定方法をスクリーンショットで解説"
emoji: "😸"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

本記事では、2021年1月24日(日)に開催された(開催される)July Tech Festa 2021 Winter(以下、JTF2021w)の発表資料からはみ出てしまったネタを取り扱っていきます。いわゆる「没ネタの供養」ですね笑。

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。本記事は、Vimライクなキーバインドでブラウザを操作するVimiumの設定方法の解説記事です。

https://twitter.com/gkzvoice/status/1351682676021817344

もともとスライド用にスクリーンショットを大量に取っていたので、スクリーンショットが多いです。そのため、Chromeの拡張機能の設定が初めてという方でも、本記事を読めばVimiumをすんなり導入できるのではないか？と思います。

### 「Vimライクなキーバインド」とは？
そもそもキーバインドとはなんでしょう。JTF2021wの発表資料でいい感じに書いたので取り上げたいと思います。

```
Vimのキーバインド
↓
Vimのキーに対する機能の割り当て
```
参考：[#JTF2021W_D VimiumではじめるよちよちVimmerライフ / Vimmer beginners start with Vimium - Speaker Deck](https://speakerdeck.com/gkzz/vimmer-beginners-start-with-vimium)

## 1. Vimiumとは
- Vimライクなキーバインドでブラウザを操作できるChrome/Firefoxの拡張機能
  - [Vimium - Chrome Web Store](https://chrome.google.com/webstore/detail/vimium/dbepggeogbaibhgnhhndojpepiihcmeb)
  - [Vimium-FF – Get this Extension for 🦊 Firefox (en-GB)](https://addons.mozilla.org/en-GB/firefox/addon/vimium-ff/)
- 読み方は「ヴィミィユム」、「ヴィミィウム」？

※ 当方はChromeユーザーなので、本記事ではChromeのVimiumを使って解説していきます。

### 1-1. Vimiumでできること
以下のことをVimライクなキーバインドで操作できます。
```
- ページの上下スクロール
- 左/右隣のタブへ移動
- ページの戻る/進む
- ページ内リンク先移動
- ページ内キーワード検索
```

## 2. 本記事における問題点の共有
```
- 設定したVimiumのキーバインドが動かないことがあり、試行錯誤してややツラかった
- 一部検索バーのハック術と併用するとVimiumの威力が倍増したので、Vimiumのドキュメントで共有してほしかった
```

## 3. 環境/バージョン情報

- ブラウザ
```
$ google-chrome --version
Google Chrome 87.0.4280.141
```
- Vimium
```
- バージョン 1.66
- 更新 2020年3月2日
- サイズ 258KiB
- 言語 English
```

## 4. Vimiumのインストールと初期設定

- [Vimium - Chrome Web Store](https://chrome.google.com/webstore/detail/vimium/dbepggeogbaibhgnhhndojpepiihcmeb) からVimiumをインストール
  - **`Add to Chrome`** → **`Add extension`** をクリック
  - **`Add to Chrome`** のボタンが **`Remove frome Chrome`** に変わっていればインストール完了
  
![](https://storage.googleapis.com/zenn-user-upload/5m2e1tyf9spfmv0539wqzoi5limd)


![](https://storage.googleapis.com/zenn-user-upload/538cqkm2tsd52g8hdf8lxjtfdgcb)

- キーバインドの追加設定
  - ブラウザの右上の方にカーソルを移動させ、 **`Extension`** → **`"︙"`** → **`Option`** をクリック
  - 表示された設定画面の **`Custom key mappings`** に以下の設定をコピペして **`Save Changes`** をクリック

```
# Custom key mappingsの入力タブ
# Insert your preferred key mappings here.
map h goBack
map l goForward
map i LinkHints.activateMode
map I LinkHints.activateModeToOpenInNewTab
```

![](https://storage.googleapis.com/zenn-user-upload/h4iixx4s8ur9zxnuqq8fxbogdhzw)

![](https://storage.googleapis.com/zenn-user-upload/gwv46ztsk33s2koxruod62153iv0)

![](https://storage.googleapis.com/zenn-user-upload/zuvkotk8lx38c9zdwp9knvi5cweb)

![](https://storage.googleapis.com/zenn-user-upload/v8ffir25v1qlu0mbjglf0b7jb75r)


## 5. Vimiumのキーバインドのオレオレ一覧

- ページの上下スクロール
  - jは上
  - kは下
- タブの移動
  - J(shift + j)は左
  - K(shift + k)は右
- ページの戻る/進む
  - hは戻る
  - lは進む
- リンククリック
  - iは現タブ上からリンク先のページへ移動
  - I(shift + i)は現タブの右側に新規タブを作り、そちらでリンク先のページへ移動
- ページの最上部と最下部
  - ggは最上部
  - Gは最下部
- ページ内検索は「/」

### 5-1. Vimiumのキーバインドじゃないけどおすすめ設定
Vimiumはカーソルが検索バーなどページ外にいってしまうとキーバインドが効かなくなってしまいます。そのような場合、「Esc」キーを押すか、検索バーに「javascript:」と入力すると、カーソルをページ内に戻すことが出来ます。検索バーにイチイチ「javascript:」なんて入力してられないので、ChromeのManage search engines(検索エンジンの設定画面でこのように設定しています。

![](https://storage.googleapis.com/zenn-user-upload/xfmbifimv8c37rwqyxpua65igsgc)

|Search engine|Keyword|URL with %s in place of query|
|-------------|-------|-----------------------------|
|movepage     |j      |javascript:                  |
|copyTitle    |t      |javascript:prompt('Title%20+%20URL','['+document.title+']('+location.href+')')();|


- カーソルをページ内に戻す(javascript:)

![](https://storage.googleapis.com/zenn-user-upload/ev4ajgyyj3ryre2peayez3gm006p)

- タイトルとURLをコピー(javascript:prompt('Title%20+%20URL','['+document.title+']('+location.href+')')();)

![](https://storage.googleapis.com/zenn-user-upload/5jppi4il1vjhngmq63r5j4ms0xmj)


## 参考

- [philc/vimium: The hacker's browser.](https://github.com/philc/vimium)
- [Chromeをvimライクに使えるようにするvimium](https://qiita.com/satoshi03/items/9fdfcd0e46e095ec68c1)
- [検索エンジンはカスタマイズしてほしいです](https://zenn.dev/eetann/articles/2020-12-09-custom-search-engine)

## P.S. Twitterもやってるのでフォローしていただけると泣いて喜びます:)

[@gkzvoice](https://twitter.com/gkzvoice)
