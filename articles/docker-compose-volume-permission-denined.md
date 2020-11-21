---
title: "docker-compose.ymlでVolumesを使ったらPermission deniedとなったときの対処"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "dockercompose"]
published: true
---

## はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。
本記事は、社内テックレビューにて表題の件についてご相談していただいたアドバイスと自身で調べたことの備忘録です。
いくつも対処案を教えていただき、ありがとうございました。

@[tweet](https://twitter.com/gkzvoice/status/1329804567458971648?s=20)


### 本記事で扱うサンプルコードのリポジトリ
- [https://github.com/gkzz/link-portal](https://github.com/gkzz/link-portal)

## 本記事における問題点の共有
まず、本記事で取り扱いたいdocker-composeで引いたエラーについて書いていきます。
- App/DB/Nginx(Django/Postgres/Nginx)で構成されるコンテナを再ビルドすると、`Permission denied` とエラーを引いたこと

```
$ docker-compsoe down -v
$ docker-compose up -d --buuild
```

### エラーの原因
- docker-composeのボリュームで指定しているDBコンテナのオーナー権限がコンテナ側とホスト側とで異なっていた
- コンテナ側のオーナーはrootユーザー、ホスト側のオーナーは非rootユーザー

`docker-compose.yml`より該当箇所抜粋
```
db:
(略)
    volumes:
      - ./db/postgres_data:/var/lib/postgresql/data
```

続いて、解決策をいくつかご紹介します。
一概に`これがベストプラクティス！`と言えず、ケースバイケースによるかなと思ったので、解決策ごとにメリットとデメリット、採用基準を簡単に添えています。

## 解決策1 
`docker-compose.yml`でvolumeの指定をやめる

- メリット
  - 元も子もないですが、後述するどの解決策よりシンプル
  - DBコンテナのデータはビルド時に`/docker-entrypoint-initdb.d/*`に記載されたデータの投入スクリプトやSQLを除いて投入されることがなければ、
- デメリット
  - DBコンテナを落とすとDBコンテナのファイルは消失する
  - ホスト側、外部サービスでDBコンテナのファイルを永続的に使う場合、避けたほうがよい
- 採用基準
  - DBコンテナのデータはビルド時に`/docker-entrypoint-initdb.d/*`に記載されたデータの投入スクリプトやSQLを除いて投入されなければ。


## 解決策2 

`docker-compose.yml`でvolumesのホスト側のディレクトリを削除してから再ビルド

```
$ sudo rm -rf ./path/to/${DB_VOLUME}
```
※ここで挙げているメリットは正確にはVolumesでDBコンテナのデータを指定していることによるものですね。

- メリット
  - 解決策1の逆でDBコンテナを落としてもDBコンテナのファイルがホスト側に残すことができる
  - DBコンテナのデータをホスト、外部サービスで使うことが出来る
- デメリット
  - `rm -rf ./path/to/${DB_VOLUME}`するのがめんどう
- 採用基準
  - DBコンテナを落としてからDBコンテナのファイルを使うか
  - DBコンテナのデータをホスト、外部サービスで使うか
