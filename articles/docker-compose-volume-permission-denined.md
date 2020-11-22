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

@[tweet](https://twitter.com/gkzvoice/status/1329804567458971648)


### 本記事で扱うサンプルコードのリポジトリ
- [https://github.com/gkzz/link-portal](https://github.com/gkzz/link-portal)

## 本記事における問題点の共有
まず、本記事で取り扱いたいdocker-composeで引いたエラーについて書いていきます。
- App/DB/Nginx(Django/Postgres/Nginx)で構成されるコンテナを再ビルドすると、**`Permission denied`** とエラーを引いたこと

```
$ docker-compsoe down -v
$ docker-compose up -d --buuild
```

### エラーの原因
- **`docker-composeのVolumesで指定しているDBコンテナのオーナー権限がコンテナ側とホスト側とで異なっていた`**
- コンテナ側のオーナーはrootユーザー、ホスト側のオーナーは非rootユーザー
```
# docker-compose.yml

db:
(略)
    volumes:
      - ./db/postgres_data:/var/lib/postgresql/data
```

続いて、解決策をいくつかご紹介します。
一概に`これがベストプラクティス！`と言えず、ケースバイケースによるかなと思ったので、対処案ごとにメリットとデメリット、採用基準を簡単に添えています。

## 対処案1 
- **`docker-compose.ymlでvolumeの指定をやめる`**
- メリット
  - 元も子もないですが、後述するどの対処案よりシンプル
- デメリット
  - DBコンテナを落とすとDBコンテナのファイルは消失する
  - ホスト側、外部サービスでDBコンテナのファイルを永続的に使う場合、避けたほうがよい
- 採用基準
  - DBコンテナのデータはビルド時に **`/docker-entrypoint-initdb.d/*に記載されたデータの投入スクリプトやSQLを除いて投入されないこと`**(*1)


## 対処案2 
- **`docker-compose.ymlでvolumesのホスト側のディレクトリを削除してから再ビルド`**

```
$ sudo rm -rf ./path/to/${DB_VOLUME}
```
※ここで挙げているメリットは正確にはVolumesでDBコンテナのデータを指定していることによるものですね。

- メリット
  - 対処案1の逆でDBコンテナを落としてもDBコンテナのファイルがホスト側に残すことができる
  - DBコンテナのデータをホスト、外部サービスで使うことが出来る
- デメリット
  - `rm -rf ./path/to/${DB_VOLUME}`するのがめんどう
- 採用基準
  - DBコンテナを落としてからDBコンテナのファイルを使うか
  - DBコンテナのデータをホスト、外部サービスで使うか


## 対処案3
- **`SELinux系であれば、zフラグを付ける`**

※Dockerの公式ドキュメントで紹介されていた方法ですが、未検証です。

> Configure the selinux label
If you use selinux you can add the z or Z options to modify the selinux label of the host file or directory being mounted into the container. 


以上、3点が表題の **`docker-compose.ymlでVolumesを使ったらPermission deniedとなったときの対処`** です。
最後に上手くいかなかった方法をご紹介します。

## うまくいかなかった方法 
- docker-composeでコンテナをビルドする際、`Permission denined`となったDBコンテナの実行ユーザーをPostgresに切り替える
- ここではDBコンテナの`db/Dockerfile`の末尾に以下のように書いて実行ユーザーをPostgresに切り替えている
```
FROM postgres:11.9-alpine

RUN apk update \
    && apk upgrade \
    && apk add --no-cache --virtual vim

RUN chown -R postgres:postgres /var/lib/postgresql/data \
  && chown -R postgres:postgres /docker-entrypoint-initdb.d
USER postgres
```
- DBコンテナを再ビルドすると、以下のようにVolumesで指定している、`/var/lib/postgresql/data`周りの権限エラーを引いた
```
$ docker-compose logs db
Attaching to db
db     | chmod: /var/lib/postgresql/data: Operation not permitted
db     | chmod: /var/lib/postgresql/data: Operation not permitted
db     | chmod: /var/lib/postgresql/data: Operation not permitted          
db     | chmod: /var/lib/postgresql/data: Operation not permitted          
db     | chmod: /var/lib/postgresql/data: Operation not permitted
db     | chmod: /var/lib/postgresql/data: Operation not permitted
db     | chmod: /var/lib/postgresql/data: Operation not permitted
db     | chmod: /var/lib/postgresql/data: Operation not permitted
```

- DBコンテナと同じくコンテナとホストとでVolumesを指定しているが、Appコンテナはapp/Dockerfileにて、Djangoアプリのルートディレクトリの作成をおこなっている。
- 一方、DBコンテナではイメージで用意されているディレクトリをVolumesで指定している点がAppコンテナと異なる。

```
# (略)

# add user
ARG APPUSER_UID=1000
ARG APPUSER=app
ARG APPUSER_PASSWORD=app
　RUN useradd -m --uid ${APPUSER_UID} --groups sudo ${APPUSER} \
  && echo ${APPUSER}:${APPUSER_PASSWORD} | chpasswd

# (略)

# set work directory
ARG HOME_DIR=/home/app
RUN mkdir -p ${HOME_DIR} \
  && chown -R ${APPUSER}:${APPUSER} ${HOME_DIR}
WORKDIR ${HOME_DIR}

# (略)

# su - ${APPUSER}
USER ${APPUSER}

# (略)
```


## 注釈
- *1 [/docker-entrypoint-initdb.d/init.sh](https://github.com/gkzz/link-portal/blob/main/db/docker-entrypoint-initdb.d/init.sh)では、以下のようにPostgresに初期データを投入している

```
#!/bin/bash

set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" << EOSQL
     CREATE USER $SQL_USER WITH ENCRYPTED PASSWORD '$SQL_PASSWORD';
     CREATE DATABASE $SQL_DB OWNER $SQL_USER;
     GRANT ALL PRIVILEGES ON DATABASE $SQL_DB TO $SQL_USER;
     ALTER USER $SQL_USER CREATEDB;
EOSQL

psql -v ON_ERROR_STOP=1 --username "$SQL_USER" --dbname "$SQL_DB" << EOSQL
     CREATE TABLE links (
         id     serial8  primary key,
         title   text     not null unique,
         url   text     not null unique,
         tag   text not null,
         created_at timestamp,
         updated_at timestamp,
         is_completed boolean default 'f',
         is_active boolean default 't'
    );
    INSERT INTO links (
        title, url, tag
    )
    VALUES (
        'Getting started with Django', 'https://www.djangoproject.com/start/', 'tutorial'
    );
EOSQL
```


## P.S. Twitterもやってるのでフォローしていただけると泣いて喜びます:)
[@gkzvoice](https://twitter.com/gkzvoice)
