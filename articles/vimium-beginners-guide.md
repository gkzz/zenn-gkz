---
title: "【Chrome拡張機能】Vim的キーバインドでブラウザを操作するVimiumの設定方法をスクリーンショットで解説"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。本記事は、Vim的キーバインドでブラウザを操作するVimiumの設定方法の解説記事です。

https://twitter.com/gkzvoice/status/1342856632523362307


## 1. Vimiumとは
- Vimライクなキーバインドでブラウザを操作できるChrome/Firefoxの拡張機能
  - [Vimium - Chrome Web Store](https://chrome.google.com/webstore/detail/vimium/dbepggeogbaibhgnhhndojpepiihcmeb)
  - [Vimium-FF – Get this Extension for 🦊 Firefox (en-GB)](https://addons.mozilla.org/en-GB/firefox/addon/vimium-ff/)
- 読み方は「ヴィミィユム」、「ヴィミィウム」？

※ 当方はChromeユーザーなので、本記事ではChromeのVimiumを使って解説していきます。

### 1-1. Vimiumでできること
以下のことをVimキーバインドっぽいキーバインドで操作できます。「Vimキーバインドっぽい」については後述します。
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
map i LinkHints.activateMode
map I LinkHints.activateModeToOpenInNewTab
```

![](https://storage.googleapis.com/zenn-user-upload/h4iixx4s8ur9zxnuqq8fxbogdhzw)

red
#f11341
10px

bk
#182329
50px