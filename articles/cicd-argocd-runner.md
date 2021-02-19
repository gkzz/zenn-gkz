---
title: "CI/CDのCDってなんでCIに含まれないの？という疑問から始まった僕のCI/CD環境構築をふりかえりる[GKE / Gitlab Runner / Argo CD]"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

書くことメモ
- CIとCDの違い
- CIOpsとGitOpsの違い
- マニフェストをアプリケーションのソースコードから分離する意義
- デプロイフェーズを自動化する際にネックとなるコンテナイメージID
  - timestampとユニーク性、時間の概念
- コンテナイメージIDにgitのコミットIDのhash値を採用した
  - cf. [gitで最新のコミットハッシュを取得する (rev-parse) - いろいろ備忘録日記](https://devlights.hatenablog.com/entry/2020/12/16/165245)
  - cf. [gitで特定のcommitバージョン/リビジョンを指すアレをなんと呼ぶか問題 - Qiita](https://qiita.com/bigwheel/items/0b331451558637ee29b3)
- 課題として残ったロールバックの自動化
