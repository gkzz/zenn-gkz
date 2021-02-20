---
title: "CI/CDのCDってなんでCIに含まれないの？という疑問を持ったあなたにこそ触ってほしいArgo CD"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。ここ半年ほどCI/CD環境構築に携わっていますが、当初はこのような疑問を抱えていました。

```
CI/CDのCDってなんでCIに含まれないの？
```
ググっても辞書的な意味しか表示されませんでした。そこで文字通り試行錯誤しながらCI/CD環境を作っては壊して繰り返して、上記の疑問に対する自分の考えはじめ、いくつかのことを学びました。そこでそれらを忘れないうちに書き残そうと思います。


### ありがとうございます

https://twitter.com/gkzvoice/status/1362733588316233735


## 1. 目次

```
- 2. CI/CDのCDってなんでCIに含まれないの？
- 3. アプリケーションリポジトリからマニフェストを切り出して別のリポジトリにする意義と課題
- 4. コンテナイメージIDにgitのコミットのハッシュを採用した
- 5. デモCI/CD環境
- 6. サンプルコードの解説
- 7. 課題として残ったロールバックの自動化
```

## 2. CI/CDのCDってなんでCIに含まれないの？

```
ビルド=デプロイ@環境A
↓
テスト@環境A
```
テストがコケたら、、、

```
ビルド@環境A
↓
テスト@環境A
↓
デプロイ@環境B
```
デプロイ環境のソースコードはテストが通過していることが保証される

## 3. アプリケーションリポジトリからマニフェストを切り出して別のリポジトリにする意義と課題

https://speakerdeck.com/masayaaoyama/cndt2020-k8s-amsy810?slide=11

責任共有モデル

- アプリケーション開発に専念出来る
 

### 3-x. マニフェストリポジトリにアプリケーションコンテナの更新をどう反映させるか

- デプロイフェーズを自動化する際にネックとなるコンテナイメージID
  - timestampとユニーク性、時間の概念

## 4. コンテナイメージIDにgitのコミットのハッシュを採用した

- cf. [gitで最新のコミットハッシュを取得する (rev-parse) - いろいろ備忘録日記](https://devlights.hatenablog.com/entry/2020/12/16/165245)
- cf. [gitで特定のcommitバージョン/リビジョンを指すアレをなんと呼ぶか問題 - Qiita](https://qiita.com/bigwheel/items/0b331451558637ee29b3)


## 5. デモCI/CD環境

draw.ioで構成図をお絵描きします。

```
Appリポジトリ更新 --------> Gitlab Runnerがbuild/test ------> testが通ったらマニフェストリポジトリをgit cloneして更新

↓
↓

Argo CDがマニフェストリポジトリの更新を取り込む

```

## 6. サンプルコードの解説

CIはGitlab Runnerが実行し、CIが完了したらArgo CDがデプロイします。

- コンテナをbuild -> run -> test -> イメージをレジストリにpush
  - DockerコマンドはMakefileに寄せている
```
before_script:
  - COMMIT_HASH=$(git rev-parse --short HEAD)
  略
script:
 - make build
 - make run
 - make coverage

 略(coverageの結果を鑑みて後続の処理を続けるか判定が必要。ここでは割愛)

 - make push
 - マニフェストリポジトリをgit clone
 - デプロイ先となるdemoブランチにcheckout
 - yqコマンドでマニフェストに記載されているコンテナイメージIDを更新
 - git add deployment.yml
 - 'git commit -m "RUNNER: ${COMMIT_HASH}" deployment.yml'
 - git push -fu origin demo
```

## 7. 課題として残ったロールバックの自動化