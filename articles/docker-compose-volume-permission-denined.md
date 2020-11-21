---
title: "docker-compose.ymlでVolumesを使ったらPermission deniedとなったときの対処"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "dockercompose"]
published: false
---

## はじめに
本記事は社内テックレビューにて表題の件についてご相談していただいたアドバイスを自分自身の備忘録です。先日いくつも対処案を教えていただき、ありがとうございました。

## 本記事における問題点の共有
- App/DB/Nginx(Django/Postgres/Nginx)で構成されるコンテナを再ビルドすると、`Permission denied` とエラーを引く

```
$ docker-compsoe down -v
$ docker-compose up -d --buuild
```

- 原因はdocker-composeのボリュームで指定しているDBコンテナのオーナー権限がコンテナ側とホスト側とで異なっていたため
- コンテナ側のオーナーはrootユーザー、ホスト側のオーナーは非rootユーザー

`docker-compose.yml`より該当箇所抜粋
```
db:
(略)
    volumes:
      - ./db/postgres_data:/var/lib/postgresql/data
```

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

- `docker-compose.yml`でvolumeのホスト側のディレクトリを削除してから再ビルド

```
$ sudo rm -rf ./path/to/${DB_VOLUME}
```
