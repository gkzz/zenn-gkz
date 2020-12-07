---
title: "Gitlab RunnerをGKE上で実行するまでの設定方法[Gcloud SDKとHelmら使用]"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "gitlab"]
published: false
---


## 0.はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。

本記事は、Gitlab RunnerをGKE上で実行するまでの設定方法についての備忘録です。

## 1.本記事で扱うサンプルコードのリポジトリ
- 作成中

## 2.本記事における問題点の共有

Gitlabが公開している、[GitLab CI/CD on Google Kubernetes Engine in 15 minutes or less](https://about.gitlab.com/blog/2020/03/27/gitlab-ci-on-google-kubernetes-engine)のとおりにいかず、試行錯誤する点がいくつかありました。

具体的には以下のとおりです。

```
- Gitlabの設定画面にてHelm Tillerが見当たらない
- Gitlab Runnerはあったが、こちらはインストールに失敗する
```

結論としては、試行錯誤した結果、無事できるようになりました。

そこで本記事ではどのようにおこなったか、書いていこうと思います。
なお、本記事には現時点での説明方法の備忘録としも活用したいので、クラスターを作成することから書いていきます。

## 3.環境/バージョン情報

### 3-1.構成図
- 作成中

### 3-2.バージョン情報

- GCE(検証環境)
```
$ grep "^VERSION=" /etc/`os-release 
VERSION="20.04.1 LTS (Focal Fossa)"
```
- GCloud SDK
```
$ gcloud version
Google Cloud SDK 318.0.0
alpha 2020.11.06
beta 2020.11.06
bq 2.0.62
core 2020.11.06
gsutil 4.54
kubectl 1.16.13
```
- kubectl
```
$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.4", GitCommit:"d360454c9bcd1634cf4cc52d1867af5491dc9c5f", GitTreeState:"clean", BuildDate:"2020-11-11T13:17:17Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}
```
- helm
```
$ helm version
version.BuildInfo{Version:"v3.4.1", GitCommit:"c4e74854886b2efe3321e185578e6db9be0a6e29", GitTreeState:"clean", GoVersion:"go1.14.11"}
```
- Gcloud config情報
```
$ gcloud config list
[compute]
region = asia-northeast1
zone = asia-northeast1-a
[core]
account = yourmail@gmail.com
disable_usage_reporting = False
project = your-project


## regionとzoneは設定されていなかった
## そこで以下のように設定した
$ gcloud config set compute/region asia-northeast1
Updated property [compute/region].
$ gcloud config set compute/zone asia-northeast1-a
Updated property [compute/zone].
```

## 4. クラスターの作成

- クラスターの作成
  - 名前はhello-cluterとした
```
$ gcloud container clusters create hello-cluter --num-nodes=1
```

```
$ gcloud container clusters get-credentials \
> $(gcloud container clusters list | grep -v "NAME" | awk '{print $1}')
Fetching cluster endpoint and auth data.
kubeconfig entry generated for quickstart.
```

## 5. DockerコンテナイメージのビルドからGCRにpushまで

- GCPプロジェクトのIDを環境変数として使えるようにする
```
$ export PROJECT_ID = your-gcp-project-id
```

- サンプルリポジトリのclone
  - 
```

```

- Dockerfileがあるディレクトリまで移動
```
$ cd gke-quickstart/app
```

- Dockerコンテナイメージのビルド
  - `hello-python`は任意のDockerコンテナの名前。後々使うmanifests/deployment.ymlでも使用
```
hoge@hoge:~/gke-quickstart/app$ docker build -f Dockerfile -t gcr.io/${PROJECT_ID}/hello-python:v1 .
```

- runする際に使うイメージIDを取得
```
hoge@hoge:~/gke-quickstart/app$ docker image ls gcr.io/${PROJECT_ID}/hello-python
REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
gcr.io/bamboo-storm-296515/hello-python   v1 
```

```
docker run -d -p 5001:5000 --name hello-python 420c84c84845
30c7fa4a48b4a06206023012de36b304a165750d785f942448d3fcc778f8c577
jump@jump:~/gke-quickstart/app$
```


```
docker push gcr.io/bamboo-storm-296515/hello-python:v1
The push refers to repository [gcr.io/bamboo-storm-296515/hello-python]
7f591ddc4dbf: Layer already exists 
f032673305e2: Layer already exists 
7157cf060c69: Layer already exists 
b60b9aee4946: Layer already exists 
eaed50855d41: Layer already exists 
026c477245c5: Layer already exists 
ee78bcfefc78: Layer already exists 
c4a6d8ca5d2c: Layer already exists 
059ed1793a98: Layer already exists 
712264374d24: Layer already exists 
475b4eb79695: Layer already exists 
f3be340a54b9: Layer already exists 
114ca5b7280f: Layer already exists 
v1: digest: sha256:01fcc05220a9edad25b091149aa9480142e16985cacc5bc33255b7c68afb7450 size: 3050
jump@jump:~/gke-quickstart/app$
```


```
$ kubectl config current-context
gke_bamboo-storm-296515_asia-northeast1-a_gke-quickstart
```

## 5.GitlabでGcloud SDKで作成したGKEと紐付け

- [https://gitlab.com](https://gitlab.com)でログインし、任意のプロジェクトをクリック
  - ここでは **`gke-quickstart`**
![](https://storage.googleapis.com/zenn-user-upload/os7ueulapbtsey1tu5o11g1nhl4b)

- **`Operations`** -> **`Kubernetes`** の順番にクリック
  - **`Operations`** をクリックすると、**`Kubernetes`** のタブが表示される
![](https://storage.googleapis.com/zenn-user-upload/5eccpal33jm3v41wsxj6j6lsgior)
![](https://storage.googleapis.com/zenn-user-upload/o4q3xdxiitktl6x1d1ng2a6kqvgx)
- **`Connect cluster with certificate`** をクリック
![](https://storage.googleapis.com/zenn-user-upload/x0qx5oznrf2nf62ah4i20snxn3xh)



```
$ kubectl get secret $(kubectl get secret | grep -v "NAME" | awk '{print $1}') -o jsonpath="{['data']['ca\.crt']}" | base64 --decode

   -----BEGIN MY CERTIFICATE-----
   -----END MY CERTIFICATE-----
   -----BEGIN INTERMEDIATE CERTIFICATE-----
   -----END INTERMEDIATE CERTIFICATE-----
   -----BEGIN INTERMEDIATE CERTIFICATE-----
   -----END INTERMEDIATE CERTIFICATE-----
   -----BEGIN ROOT CERTIFICATE-----
   -----END ROOT CERTIFICATE-----
```

```
$ kubectl apply -f gitlab/gitlab-admin-service-account.yaml
serviceaccount/gitlab created
clusterrolebinding.rbac.authorization.k8s.io/gitlab-admin created

$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep gitlab | awk '{print $1}')
Name:         gitlab-token-2d59g
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: gitlab
              kubernetes.io/service-account.uid: ad6cdff8-5228-4ad6-8f4f-910e29254547

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1159 bytes
namespace:  11 bytes
token: <authentication_token>
```

```
$ kubectl create namespace gitlab
namespace/gitlab created
$ helm repo add gitlab https://charts.gitlab.io
"gitlab" has been added to your repositories
```

![](https://storage.googleapis.com/zenn-user-upload/h2mezbme12tigjt0314d2fspkgjw)

![](https://storage.googleapis.com/zenn-user-upload/cewj5o4000nelrgfwvf1waqv85du)

```
$ helm install -n gitlab gitlab-runner -f values.yaml gitlab/gitlab-runner
NAME: gitlab-runner
LAST DEPLOYED: Thu Nov 26 00:28:14 2020
NAMESPACE: gitlab
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Your GitLab Runner should now be registered against the GitLab instance reachable at: "https://gitlab.com/"
$ helm list -n gitlab
NAME         	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
gitlab-runner	gitlab   	1       	2020-11-26 00:28:14.721212245 +0000 UTC	deployed	gitlab-runner-0.23.0	13.6.0     
```


```
# .gitlab-ci.yml

include: 
  - .gitlab-ci.d/.dev-test.yml
  - project: gkzz/gitlab-ci-common
    ref: master
    file: .failure.yml

before_script:
  - docker info

stages:
  - build
  - test
  - deploy
  - failure

build_always:
  stage: build
  script:
    - echo "Hello, $GITLAB_USER_LOGIN!"
  tags:
    - always

test_always:
  stage: test
  extends: .dev-test
  tags:
    - always

deploy_always:
  stage: deploy
  script:
    - echo "This job deploys something from the $CI_COMMIT_BRANCH branch."
  tags:
    - always

on_failure:
  stage: failure
  extends: .failure
  tags:
    - always
  when: on_failure

```

```
.failure:
  script: 
    - echo "Something wrong"
```


sedでmanifestファイル書き換え


参考

- [GitLabの継続的インテグレーション(CI)と継続的デリバリー(CD)](https://www.gitlab.jp/stages-devops-lifecycle/continuous-integration)

- [kelseyhightower/kubernetes-the-hard-way: Bootstrap Kubernetes the hard way on Google Cloud Platform. No scripts.](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [GitLab CI/CD on Google Kubernetes Engine in 15 minutes or less](https://about.gitlab.com/blog/2020/03/27/gitlab-ci-on-google-kubernetes-engine/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

- [Deploying a containerized web application](https://cloud.google.com/kubernetes-engigcloud%20container%20clusters%20get-credentialsne/docs/tutorials/hello-app)
