---
title: "[Gitlab RunnerとArgo CD使用]GitOpsスタイルなCI/CDパイプラインを構築したのでふりかえる"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "gitlab"]
published: false
---

## 0. はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。Gitlab RunnerとArgo CDを使って、GitOpsの考え方を取り入れたCI/CDパイプラインを構築したのでそのふりかえりをします。構築する当初、そもそもなぜGitOpsという考え方が台頭したのか？について分からず、調べたので、併せてお話しできればと思います。目次は以下のとおりです。

```
0. はじめに
  0-1. 前提
  0-2. 2つのCDの個人的なメモ

1. GitOpsとはなにか？
  1-1. GitOpsの個人的なメモ

2. そもそもなぜGitOpsという考え方が台頭したのか？
  2-1. CIOpsが抱えていた課題とはなにか？

3. サンプルのGitOpsスタイルなCI/CDパイプラインの構成図とその技術構成
  3-1. サンプルのCI/CDパイプラインの技術構成
  3-2. サンプルのCI/CDパイプラインでおこなうこと

4. サンプルのCI/CDパイプラインを構築するにあたり工夫した2つのこと
  4-1. manifestのイメージタグにアプリケーションリポジトリのgit commit hashを渡す方法
  4-2. なぜmanifestに書くイメージタグはgit commitのhashとしたのか？
  
5. .gitlab-ci.yml解説
 5-1. Runnerが最初に読み込む.gitlab-ci.ymlの解説
 5-2. アプリケーションコンテナのイメージをbuildするところからGCRにpushするところまでの解説
 5-3. manifestのイメージタグを更新し、manifestリポジトリへpushするところまでの解説

6. トラブルシュート

7. おわりに

8. 記事を書く際、参考にした記事のリンク集
```

- CI/CDパイプラインが完走した様子(画面左)ととmanifestのイメージタグを書き換えている様子(画面右)
![](https://storage.googleapis.com/zenn-user-upload/vwm196o5eirzrnpo2udk66dmey0x)

- Runnerが作成したMRからマージすると(画面左)、Argo CDが同期(sync)してくれる(画面右)
![](https://storage.googleapis.com/zenn-user-upload/f6w4k37mxm6igx2lw4w83y16rtyk)

### 0-1. 前提
- ここで扱うCI/CDパイプラインがデプロイするアプリケーションは以下のとおり簡素なものとします。そのため、パイプラインは実運用には足らないところがあるかもしれません。
  - DB無し
  - アプリケーションはFastAPIで作成
  - GKE上で動く(GKEは以下の資料を参考に作成)
  - 参考0-1-1. [Quickstart  |  Kubernetes Engine Documentation  |  Google Cloud][ref-0-1-1]
- また、ここでの **`CDとは「 継続的デリバリー(Continuous Delivery)」 `** を指します。「 継続的デプロイメント(Continuous Deployment)」ではありません。両者の違いについてはRed Hat社の記事から引用します。
  - > 継続的デリバリーは一般に、開発チームによるアプリケーションへの変更に対してバグがないか自動でテストを行い、変更をリポジトリ (GitHub やコンテナレジストリなど) にアップロードします。 **`ここで、変更が運用チームによって本番環境に導入されます。`**
  - > 継続的デプロイでは、新しいソフトウェアのリリースプロセスを通じてさらにいくつかのステップをカバーします。これには通常、開発者による変更をリポジトリから本番環境に自動的にリリースし、顧客が使用できるようにするプロセスが含まれます。運用チームが担当する手動プロセスが多すぎて、アプリケーションの提供が遅れるという問題に対処します。
  - 参考0-1-2：[継続的デリバリーとは][ref-0-1-2]
- 構成図はdraw.ioにて作成しました。アイコンはこちらから拝借しました。
  - [Press kit | GitLab](https://about.gitlab.com/press/press-kit/)
  - [CNCF Branding | Argo](https://cncf-branding.netlify.app/projects/argo/)


### 0-2. 2つのCDの個人的なメモ
- **`「継続的デリバリー(Continuous Delivery)」`** では商用環境へのデプロイの手前までパイプラインでおこない、同環境へのデプロイは手動でおこなう
- 一方、 **`「 継続的デプロイメント(Continuous Deployment)」`** では、商用環境へのデプロイをも自動化するということ。
  - つまり、**`「 継続的デプロイメント(Continuous Deployment)」では、開発者のpushに始まるパイプラインが完走した場合、商用環境までpushされた結果が反映される`** ことになる。


## 1. GitOpsとはなにか？
GitOpsとは、Weaveworks社が提唱した、CDの方法のひとつです。同社のブログでは、GitOpsについて以下のように書かれています。（記事をGoogle翻訳に書けた結果を引用する。2020/04/04現在。）

- > GitOpsは、Kubernetesクラスター管理とアプリケーション配信を行う方法です。**`これは、宣言型インフラストラクチャとアプリケーションの信頼できる唯一の情報源としてGitを使用する`**  ことで機能します。
- 出所：[Guide To GitOps][ref-1-0-0]

### 1-1. GitOpsの個人的なメモ
- GitOpsとは、**`manifestの更新をデプロイ環境に引き込むデリバリー方法`** のこと
  - この「デプロイ環境への引き込み」はCIツールではなく、**`Argo CDをはじめとするOperator`** と呼ばれるツールがおこなう
  - そのデリバリー方法の特徴から「Pull Model」と評されることもある
- GitOpsにおいて重要な考え方は、**`"a single source of truth"`**
  - アプリケーションをデプロイする際に使うmanifestをGitのバージョン管理下におく
  - ロールバックも容易なものとなるように目指す
- GitOpsの真の力を引き出すには、 **`アプリケーションコードとmanifest(インフラストラクチャコード)を2つのリポジトリに分けること`** が必要ではないか？
  - アプリケーションコードとmanifest(インフラストラクチャコード)を2つのリポジトリに分けることで、**`CIを頻繁に実行してもCDまで実行されるかどうかはパイプラインの設定に「カンタンに」委ねる`** ことが出来る
  - これがよくいわれる **`CIとCDを分離する`** ということだと思う
  - つまり、もちろん別々のリポジトリに分けなくてもCIが実行されてもCDは実行しないという制御はできるけど、別々にリポジトリを分ける場合に比べて設定ファイルの見通しは悪くなり、保守もされなくなり、使いづらいパイプラインが生まれやすいということかなと

![](https://storage.googleapis.com/zenn-user-upload/lmmirpg4h9fawc6ylin1u1v6firy)

## 2. そもそもなぜGitOpsという考え方が台頭したのか？
続いて、GitOpsという考え方が台頭したのか？について考えましょう。この問いは次のように言い換えることができます。

```
GitOpsが台頭する以前のデリバリー方式が抱えていた課題はなんだったのか？
```
ここでは、GitOpsが台頭する以前のデリバリー方式をCIOpsとして話を進めます。CIOpsのツールは、たとえば、JenkinsやTravis CIが挙げられれます。

### 2-1. CIOpsが抱えていた課題とはなにか？
端的に言えば、**`CIとCDは密結合の関係`** となってしまっているという点でしょう。

これの何が面倒かというとたとえば、以下のような点があげられるのではないでしょうか。

- デプロイ環境へのアクセスやデプロイ方法が変われば、CD構築者は **`CIOpsのツールの設定内容を更新しなければならない`**
- CDを進める際に使うツールは「CI」でも使われているものが兼務する場合、CD構築者は **`CIOpsのツールの権限を拡大せざるを得ない`**
  - CIOpsのツールはインテグレーションのみならずデリバリーの責務も追うことになる
  - たとえば、コンテナレジストリーとクラスター双方へのアクセス権限がCIOpsのツールに付与されるなど
- CDの実行状況はGitの管理下から外れ、CI/CDパイプラインの実行状況を追うことが煩わしい
  - CDの実行状況はCDを担うCIOpsのツールが提供するログを追うことになるけど。。

![](https://storage.googleapis.com/zenn-user-upload/wl1igyjf9iab74gjqaoaju76vrzn)

※「Push Model」とはCIOpsのデリバリー方法の特徴を表現するもの。

さて、これまでGitOpsの概要とそれが台頭してきた背景＝CIOpsが抱えていた課題についてお話してきました。

それではいよいよ、GitOpsスタイルなCI/CDパイプラインとはどんなものか。サンプルの構成図を使ってお話しましょう。

---

## 3. サンプルのGitOpsスタイルなCI/CDパイプラインの構成図とその技術構成

![](https://storage.googleapis.com/zenn-user-upload/syjdas7c7cgto1ciayhlmmlfmsnj)

### 3-1. サンプルのCI/CDパイプラインの技術構成
- Gitリポジトリサービス: Gitlab
- CI: Gitlab Runner
  - 設定方法については手前味噌ですがこちらに譲ります。
  - [Gitlab RunnerをGKE上で実行するまでの設定方法[Google Cloud SDKとHelmら使用]][ref-3-1-1]
- CD(Operator): Argo CD
  - Argo CDの設定については、公式チュートリアルに従って進めました。一部つまずいたところがあったので、スクラップに備忘録として残しました。また、Argo CDとアプリケーションは同じGKEクラスター上にかまえ、GKEクラスターやArgo CD、アプリケーションは全てシングル構成です。
  - [Getting Started - Argo CD - Declarative GitOps CD for Kubernetes][ref-3-1-2]
  - [FATA[0000] Argo CD server address unspecified][ref-3-1-3]
- アプリケーション: GKE上で動くPythonアプリ(FAST APIで作成)
- コンテナレジストリサービス: GCR

### 3-2. サンプルのCI/CDパイプラインでおこなうこと
#### CI＝Gitlab Runnerがおこなうこと
- buildしたイメージを使ってcoverageの測定
- coverageの結果がしかるべき場合のみ、GCRへイメージをpush
- manifestのイメージタグの書き換え及びmanifestリポジトリへpush 

![](https://storage.googleapis.com/zenn-user-upload/ecbf2vbntvvu3rocipzdjokz9328)

#### CD=Argo CDがおこなうこと
- manifestリポジトリの任意のブランチに更新が走った場合、その更新をデプロイ環境に引き込むようにする
  - ここでは、${BRANCHNAME}というブランチに更新が走ったら自動で同期＝manifestのリモートリポジトリの更新をローカルへ引き込むようにしている
  - なお、`--dest-server https://kubernetes.default.svc` はデプロイするアプリケーションをArgo CDと同じクラスター内とするというオプションである

```
$ argocd app create hello-python \
--repo git@gitlab.com::${ACCOUNTNAME}/${MANIFEST_ROOT_DIR}.git  \
--path manifests \
--dest-server https://kubernetes.default.svc \
--dest-namespace ${APPNAMESPACE} \
--revision ${BRANCHNAME} \
--sync-policy automated --auto-prune --self-heal
```

※アプリケーションがあるクラスターはArgo CDからするとクラスタの外にある場合の書き方は[Multicluster GitOps with ArgoCD][ref-3-2-1]によるとこのようなものらしい(筆者、未検証。)
```
$ argocd cluster add $CLUSTER_NAME
# これをおこなったあと、デプロイしたいクラスターのMASTER_IPをdest_serverに渡す
```
出所：[Multicluster GitOps with ArgoCD][ref-3-2-1]


## 4. サンプルのCI/CDパイプラインを構築するにあたり工夫したこと
ところで、リポジトリをアプリケーションとmanifestの2つに分けるということは、GitOpsの重要な考え方である、**`"a single source of truth"`** に反しているとはならないのでしょうか？

リポジトリをアプリケーションリポジトリとmanifestリポジトリの2つに分けることになっても、「a single source of truth」をどうやって守るか？その鍵を握るのが工夫したこととして挙げる、**`manifestに書くイメージタグはgit commitのhashを採用`** することです。

![](https://storage.googleapis.com/zenn-user-upload/tu4f6k5pht91pcl24atceim559wx)

![](https://storage.googleapis.com/zenn-user-upload/um5iqs9cryz77j4i7svclf8sov9c)

### 4-1. manifestのイメージタグにアプリケーションリポジトリのgit commit hashを渡す方法
- manifestで使うアプリケーションコンテナのイメージタグをdocker buildする際にタグの値を **`git rev-parse --short HEAD`** で渡す
  - dockerコマンドはオプションが似たようなものが多く、可読性が低いことからdockerコマンドはMakefileに寄せ、Runnerはmakeコマンド経由でイメージタグを付与した上でdocker buildするようにしている
- manifestのイメージタグの書き換えはRunnerがyqコマンドで書き換える
  - yqコマンドの使い方も以前書いた記事があるのでそちらをご参照ください。
  - [yqコマンド(jq wrapper for YAML)使い方備忘録](https://zenn.dev/gkz/articles/yq-beginners-guide)

```
- >
  docker run --rm -v "$PWD:$PWD" -w="$PWD"
  --entrypoint yq linuxserver/yq
  -ry '.spec.template.spec.containers[0].image
  |="gcr.io/'${PROJECT_ID}'/'${CONTAINER_NAME}':'{COMMIT_HASH}'"'
  deployment.yml.tmpl > deployment.yml
```

### 4-2. なぜmanifestに書くイメージタグはgit commitのhashとしたのか？
当初はタイムスタンプを採用していましたが、やめました。理由は、以下の2点です。

- 仮にRunnerが並列で実行された場合のパイプラインの実行順序とタグが生成される順序がイコールであるとは保証できないため
- 一般的にタグの命名規則が推測されることはあまりよろしくないとされているため

タグの命名規則について考えるにあたり、以下の2点は学びがありました。

- [System Design Interview – An insider's guide, Second Edition | Xu, Alex](https://www.amazon.co.jp/dp/B08CMF2CQF)の「Chapter 7: Design A Unique Id Generator In Distributed Systems」
- [Snowflake形式のIDを採用した場合の苦労ポイント - yoskhdia’s diary](https://yoskhdia.hatenablog.com/entry/2018/01/05/124633)

とりわけ前者の書籍では複数のサーバー間で採番する難しさとして、このように書かれていたことが印象的です。
> auto_inrements(DBが提供する自動採番)の課題
> - Hard to scale up with multiple data centers(Only writer is responsible for generating IDs)
> - IDs do not go up with time across multiple svs
> - It does not scale well when a sv is added/removed.

他にも学びがあったので、zennのスクラップにメモを残しました。

- https://zenn.dev/gkz/scraps/fe827186c5d8bc#comment-5d8009d756861e

## 5. .gitlab-ci.yml解説
### 5-1. Runnerが最初に読み込む.gitlab-ci.ymlの解説
- 一部抜粋
- .gitlab-ci.ymlはincludeとextendsを使って、.gitlab-ci.ymlがincludeに書かれたymlを読み込むようにしている(参考5-1-1及び5-1-2)
```
include:
  - .gitlab-ci.d/.build_dev.yml
  - .gitlab-ci.d/.deploy_dev.yml

stages:
  - build
  - deploy

pages:  ## 5-2
  stage: build
  extends: .build_dev

deploy_dev: ## 5-3
  stage: deploy
  extends: .deploy_dev
```
- 参考5-1-1. [includefile | Keyword reference for the .gitlab-ci.yml file | GitLab](https://docs.gitlab.com/ee/ci/yaml/#includefile)
- 参考5-1-2. [extends | Keyword reference for the .gitlab-ci.yml file | GitLab](https://docs.gitlab.com/ee/ci/yaml/#extends)

### 5-2. アプリケーションコンテナのイメージをbuildするところからGCRにpushするところまでの解説
```
.build_dev:
  image: google/cloud-sdk:330.0.0-slim
### cloud-sdkコンテナのなかでアプリケーションコンテナをbuildしていく「Docker in Docker」に必要な設定
  services:
    - docker:19.03.13-dind
#    - docker:20-dind
#    - docker:20.10-dind
#    - docker:dind   ## 「6. トラブルシュート」に記載
#### 参考5-2-1
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
    DOCKER_TLS_VERIFY: 1
    DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"
  before_script:
    - PROJECT_ID=$(gcloud config list project --format="value(core.project)")
#### gcloudコマンドをサービスアカウント権限で使うようにする
    - echo $SERVICE_ACCOUNT_KEY > ${HOME}/gcloud-service-key.json
#### 参考5-2-2
    - gcloud auth activate-service-account --key-file ${HOME}/gcloud-service-key.json
    - gcloud config set project $PROJECT_ID
#### 参考5-2-3
    - gcloud auth configure-docker
    - COVERAGE=0
  script:
    - make build
    - make run
#### 参考5-2-4
    - make pytest
    - COVERAGE=$(grep -E '<span class="pc_cov">([0-9]{1,3})%</span>' app/htmlcov/index.html | grep -o '[[:digit:]]*')
    - mv app/htmlcov/ public
    - if [ $COVERAGE -lt 80 ]; then exit 1 ;fi
    - make push
#### 参考5-2-4, 5-2-5
#### coverageレポートはGitlab Pagesで見るようにした
  artifacts:
    paths:
#      - app/htmlcov
      - public
```
- 参考5-2-1. [Use Docker to build Docker images | GitLab](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#kubernetes) 
- 参考5-2-2. [gcloud auth activate-service-account  | Cloud SDK Documentation](https://cloud.google.com/sdk/gcloud/reference/auth/activate-service-account)
- 参考5-2-3. [gcloud auth configure-docker  |  Cloud SDK Documentation  |  Google Cloud](https://cloud.google.com/sdk/gcloud/reference/auth/configure-docker)
- 参考5-2-4. [pytestのすぐに使えるカバレッジ計測 - Qiita](https://qiita.com/kg1/items/e2fc65e4189faf50bfe6)
- 参考5-2-5 [Publish code coverage report with GitLab Pages | GitLab](https://about.gitlab.com/blog/2016/11/03/publish-code-coverage-report-with-gitlab-pages/)

- カバレッジレポートの一例
![](https://storage.googleapis.com/zenn-user-upload/d94fsbo29pcoefghamoumbsv70ki)


- Runnerが使っているMakefile抜粋
  - イメージタグ付与 
  - $(shell <COMMAND>)でMakefileにコマンドを書いている
```
CONTAINER_NAME := hello-python
#### 参考5-2-6
PROJECT_ID:= $(shell gcloud config list project --format="value(core.project)")
COMMIT_HASH := $(shell git rev-parse --short HEAD)

.PHONY: build
build: ## make build
        docker build -f Dockerfile -t gcr.io/${PROJECT_ID}/${CONTAINER_NAME}:${COMMIT_HASH} .

.PHONY: run
run: ## make run
        docker run -d -p 80:80 --name ${CONTAINER_NAME}-${COMMIT_HASH} -v ${PWD}/app:/app gcr.io/${PROJECT_ID}/${CONTAINER_NAME}:${COMMIT_HASH}

.PHONY: push
push: ## make push
        docker push gcr.io/${PROJECT_ID}/${CONTAINER_NAME}:${COMMIT_HASH}

#### 参考5-2-4, 5-2-5
.PHONY: pytest
pytest: ## make pytest
	docker exec ${CONTAINER_NAME}-${COMMIT_HASH} /bin/bash -c 'pytest -v --cov=tests --cov-report=html'
```
- 参考5-2-6. [bash - How to use shell commands in Makefile - Stack Overflow](https://stackoverflow.com/questions/10024279/how-to-use-shell-commands-in-makefile)


###  5-3. manifestのイメージタグを更新し、manifestリポジトリへpushするところまでの解説
```
.deploy_dev:
  image: google/cloud-sdk:330.0.0-slim
  services:
    - docker:19.03.13-dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
    DOCKER_TLS_VERIFY: 1
    DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"
  before_script:
    - PROJECT_ID=$(gcloud config list project --format="value(core.project)")
    - COMMIT_HASH=$(git rev-parse --short HEAD)
    - PROJECT_OWNER=gkzz
    - MANIFEST_ROOT_DIR=gke-quickstart-manifest
    - CONTAINER_NAME=hello-python
    - TARGET_BRANCH=demo
  script:
### Runnerがmanifestリポジトリをclone/MRするために必要な鍵認証の設定をする
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - git credential-cache exit
    - ssh-keyscan -H "$CI_SERVER_HOST" >> ~/.ssh/known_hosts
    - 'which ssh-agent || ( apk add --update openssh )'
    - eval "$(ssh-agent -s)"
    - echo "$SSH_PRIVATE_KEY" | ssh-add - > /dev/null
### manifestリポジトリをcloneし、MR元のブランチへcheckout
    - git config --global user.name "dummy@example.com"
    - git config --global user.email "dummy@example.com"
    - git clone git@${CI_SERVER_HOST}:${PROJECT_OWNER}/${MANIFEST_ROOT_DIR}.git && cd ${MANIFEST_ROOT_DIR}/manifests
    - git remote set-url --push origin git@${CI_SERVER_HOST}:${PROJECT_OWNER}/${MANIFEST_ROOT_DIR}.git
    - git checkout -b argocd/${COMMIT_HASH}
#### yqのイメージを使ってmanifestのイメージタグを${COMMIT_HASH}の値に書き換える
#### 参考5-3-1
    - >
      docker run --rm -v "$PWD:$PWD" -w="$PWD"
      --entrypoint yq linuxserver/yq
      -ry '.spec.template.spec.containers[0].image
      |="gcr.io/'${PROJECT_ID}'/'${CONTAINER_NAME}':'${COMMIT_HASH}'"'
      deployment.yml.tmpl > deployment.yml
 #### MR
 #### 参考5-3-2
 https://dev.to/farnabaz/gitlab-create-merge-requests-from-cli-c36
    - git add deployment.yml
    - 'git commit -m "RUNNER: ${COMMIT_HASH}" deployment.yml'
    - >
      git push
      -o merge_request.create
      -o merge_request.title="Runner: ${COMMIT_HASH}"
      -o merge_request.target=${TARGET_BRANCH}
      origin argocd/${COMMIT_HASH}
```
- 参考5-3-1. [linuxserver/yq](https://hub.docker.com/r/linuxserver/yq)
- 参考5-3-2. [Gitlab: Create merge requests from cli - DEV Community 👩‍💻👨‍💻](https://dev.to/farnabaz/gitlab-create-merge-requests-from-cli-c36)

## 6. トラブルシュート
- [failed to adjust OOM score for shim: invalid argument error -- for docker:dind in Kubernetes · Issue #4837 · containerd/containerd](https://github.com/containerd/containerd/issues/4837#issuecomment-794162761)
  - ここで使っているGKE上のRunnerでは、issueのコメントでfixしたとされているubuntu20.04を使用。まもなく今回引いたエラーも解消されるものと考える。

```
使っているvalues.yamlより
  143 runners:
  144   config: |
  145     [[runners]]
  146       [runners.kubernetes]
  147         image = "ubuntu:20.04"
  148         privileged = true
  149       [[runners.kubernetes.volumes.empty_dir]]
  150         name = "docker-certs"
  151         mount_path = "/certs/client"
  152         medium = "Memory"
```
- バージョン未指定のときはサーバー側で20.10.5となり、エラーとなる
```
Step 11/31 : RUN mkdir -p $HOME_DIR
175 ---> Running in xxxxxxxx
176io.containerd.runc.v2: failed to adjust OOM score for shim: set shim OOM score: write /proc/311/oom_score_adj: invalid argument
177: exit status 1: unknown
```

```
$ docker version
  Version:          19.03.13 ## docker:19.03.13-dind
  Version:          20.10.5  ## docker:dind　エラーとなったときに記述していたもの
```

## 7. おわりに
さいごに、Gitlab RunnerとArgo CD使用]GitOpsスタイルなCI/CDパイプラインを構築して学んだこと、今後も考えなければならないこと、そしてCI/CDパイプラインを設計する際に参考にした資料を列挙します。

#### 学び
- アプリケーションのソースコードとKubernetesのmanifest(インフラ)を分離することで、CIツール＝Gitlab Runnerにデプロイ環境へのアクセス権限を渡す必要がなくなった。
- manifestをGitで管理することで、かつCDはArgo CDを通してGitでおこなうことで、デプロイ環境をGitでバージョン管理することができた
  - このような手順書と作業ログファイル大量発生から解放されるはず！？
  - デプロイ手順書_v1、デプロイ手順書2_v1レビュー済、デプロイ手順書_v1デプロイ後のロールバック作業ログ。。

#### 今後も考えなければならないこと
- 環境変数など、デプロイ環境に応じて値が変わる変数の取り扱いや設定ファイルの指定はだれがおこなうか？
  - 従来はアプリケーションのconfig、dotenvやDockerfileのENVなどだったはずだが、デプロイ環境がKubernetesの場合はどうする？
- GKEの場合、セキュアな変数を取り扱う際にはSecretを使うことがあるが、これもコード化する？manifestリポジトリのGit管理下に含める？
  - 出所：[Secret を使用してセンシティブ データを保存  |  Config Connector のドキュメント  |  Google Cloud](https://cloud.google.com/config-connector/docs/how-to/secrets)

#### CI/CDパイプラインを設計する際に参考にした資料
- [マイクロサービスパターン[実践的システムデザインのためのコード解説] (impress top gear) | Chris Richardson, 樽澤広亨, 長尾高弘 |本 | 通販 | Amazon](https://www.amazon.co.jp/dp/4295008583)
  - > オーダーサービスのためのデプロイメントパイプラインの例。パイプラインは一連のステージから構成されている。コミット前テストは、コードをコミットする前に開発者が実行するテストである。その他のステージは、 Jenkins CI サーバーなどの自動化ツールによって実行される（図 9.9）
- [CircleCIおよびArgoCDを使用したKubernetesCI / CD](https://ichi.pro/circleci-oyobi-argocd-o-shiyoshita-kubernetesci-cd-110448048135194)
- [Kubernetes anti-patterns: Let's do GitOps, not CIOps!](https://www.weave.works/blog/kubernetes-anti-patterns-let-s-do-gitops-not-ciops)

## 8. 記事を書く際、参考にした記事のリンク集
- [継続的デリバリーとは](https://www.redhat.com/ja/topics/devops/what-is-continuous-delivery)
- [Guide To GitOps](https://www.weave.works/technologies/gitops/)
- [Gitlab RunnerをGKE上で実行するまでの設定方法[Google Cloud SDKとHelmら使用]](https://zenn.dev/gkz/articles/gke-gitlab-cicd)
- [Getting Started - Argo CD - Declarative GitOps CD for Kubernetes](https://argoproj.github.io/argo-cd/getting_started/)
- [FATA[0000] Argo CD server address unspecified](https://zenn.dev/gkz/scraps/c2b18eaf5ccac3)
- [Multicluster GitOps with ArgoCD](https://www.infracloud.io/blogs/multicluster-gitops-argocd/)
- [yqコマンド(jq wrapper for YAML)使い方備忘録](https://zenn.dev/gkz/articles/yq-beginners-guide)
- [System Design Interview – An insider's guide, Second Edition | Xu, Alex](https://www.amazon.co.jp/dp/B08CMF2CQF)
- [Snowflake形式のIDを採用した場合の苦労ポイント - yoskhdia’s diary](https://yoskhdia.hatenablog.com/entry/2018/01/05/124633)
- https://zenn.dev/gkz/scraps/fe827186c5d8bc#comment-5d8009d756861e
- [includefile | Keyword reference for the .gitlab-ci.yml file | GitLab](https://docs.gitlab.com/ee/ci/yaml/#includefile)
- [extends | Keyword reference for the .gitlab-ci.yml file | GitLab](https://docs.gitlab.com/ee/ci/yaml/#extends)
- [Use Docker to build Docker images | GitLab](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#kubernetes) 
- [gcloud auth activate-service-account  | Cloud SDK Documentation](https://cloud.google.com/sdk/gcloud/reference/auth/activate-service-account)
- [gcloud auth configure-docker  |  Cloud SDK Documentation  |  Google Cloud](https://cloud.google.com/sdk/gcloud/reference/auth/configure-docker)
- [pytestのすぐに使えるカバレッジ計測 - Qiita](https://qiita.com/kg1/items/e2fc65e4189faf50bfe6)
- [Publish code coverage report with GitLab Pages | GitLab](https://about.gitlab.com/blog/2016/11/03/publish-code-coverage-report-with-gitlab-pages/)
- [bash - How to use shell commands in Makefile - Stack Overflow](https://stackoverflow.com/questions/10024279/how-to-use-shell-commands-in-makefile)
- [linuxserver/yq](https://hub.docker.com/r/linuxserver/yq)
- [Gitlab: Create merge requests from cli - DEV Community 👩‍💻👨‍💻](https://dev.to/farnabaz/gitlab-create-merge-requests-from-cli-c36)
- [failed to adjust OOM score for shim: invalid argument error -- for docker:dind in Kubernetes · Issue #4837 · containerd/containerd](https://github.com/containerd/containerd/issues/4837#issuecomment-794162761)
- [マイクロサービスパターン[実践的システムデザインのためのコード解説] (impress top gear) | Chris Richardson, 樽澤広亨, 長尾高弘 |本 | 通販 | Amazon](https://www.amazon.co.jp/dp/4295008583)
- [Secret を使用してセンシティブ データを保存  |  Config Connector のドキュメント  |  Google Cloud](https://cloud.google.com/config-connector/docs/how-to/secrets)

[ref-0-1-1]:https://cloud.google.com/kubernetes-engine/docs/quickstart
[ref-0-1-2]:https://www.redhat.com/ja/topics/devops/what-is-continuous-delivery
[ref-1-0-0]:https://www.weave.works/technologies/gitops/
[ref-3-1-1]:https://zenn.dev/gkz/articles/gke-gitlab-cicd
[ref-3-1-2]:https://argoproj.github.io/argo-cd/getting_started/
[ref-3-1-3]:https://zenn.dev/gkz/scraps/c2b18eaf5ccac3
[ref-3-2-1]:https://www.infracloud.io/blogs/multicluster-gitops-argocd/