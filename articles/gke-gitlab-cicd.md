---
title: "Gitlab RunnerをGKE上で実行するまでの設定方法[Google Cloud SDKとHelmら使用]"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "gitlab"]
published: false
---

これは [エーピーコミュニケーションズ Advent Calendar 2020](https://qiita.com/advent-calendar/2020/ap-com) の9日目の記事です。

## 0.はじめに
こんにちは。都内でエンジニアをしている、[@gkzvoice](https://twitter.com/gkzvoice)です。

本記事は、Gitlab RunnerをGKE上で実行するまでの設定方法についての備忘録です。

## 1.本記事における問題点の共有

Gitlabが公開している、[GitLab CI/CD on Google Kubernetes Engine in 15 minutes or less](https://about.gitlab.com/blog/2020/03/27/gitlab-ci-on-google-kubernetes-engine)のとおりにいかず、試行錯誤する点がいくつかありました。

具体的には以下のとおりです。

```
- Gitlabの設定画面にてHelm Tillerが見当たらない
- Gitlab Runnerはあったが、こちらはインストールに失敗する
```

結論としては、無事Gitlab RunnerをGKE上で実行することができるようになりました。

そこで本記事ではどのようにおこなったか、書いていきます。

## 2.環境/バージョン情報

### 2-1.超簡易構成図

```
<GCE(検証環境)> # git push
↓
<gitlab.com> # 後続する設定に該当するGitlab Runnerを実行
↓ 
<GKE上のGitlab Runner> # .gitlab-ci.ymlに書かれたジョブを実行
↓
echo "success" 
```

### 2-2. GCPで有効化しなければいけないAPI

上記のGitlabのブログ、[GitLab CI/CD on Google Kubernetes Engine in 15 minutes or less](https://about.gitlab.com/blog/2020/03/27/gitlab-ci-on-google-kubernetes-engine)に以下のAPIを有効化するようにと書かれています。

> A Google Cloud project with the following APIs enabled:
>
> **`Google Kubernetes Engine API`**
>
> **`Cloud Resource Manager API`**
>
> **`Cloud Billing API`**

### 2-3.バージョン情報
- GCE(検証環境)
```
$ grep "^VERSION=" /etc/os-release 
VERSION="20.04.1 LTS (Focal Fossa)"
```
- Google Cloud SDK（以下、GCloud SDKと記載）
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
- GCloud SDKのconfig情報
```
$ gcloud config list
[compute]
region = asia-northeast1
zone = asia-northeast1-a
[container]
cluster = hello-cluster
[core]
account = yourmail@gmail.com
disable_usage_reporting = False
project = your-project
```

### 2-4.GCloud SDKのconfigの追加設定
- regionとzoneとclusterは設定されていなかった
- そこで以下のように設定した
```
$ gcloud config set compute/region asia-northeast1
$ gcloud config set compute/zone asia-northeast1-a
$ gcloud config set container/cluster hello-cluster
```

### 2-5.GCPプロジェクトのIDを環境変数として使えるようにする
※本記事では説明の便宜上、 **`GCPプロジェクトのIDを${PROJECT_ID}`** と記載します。[^1]

```
$ export PROJECT_ID = your-gcp-project-id
```

これでGitlab Runnerを動かすGKEクラスターを作成する準備が終わりました。

続いてGCloud SDKでGKEクラスターを作成していきます。

-----

## 3. GKEクラスターの作成
- クラスターの名前をhello-cluterとして作成
```
$ gcloud container clusters create hello-cluter --num-nodes=1
```
- 作成したクラスターの確認
```
$ gcloud container clusters list | grep -v "NAME" | awk '{print $1}'
hello-cluster
```
- kubectl config current-contextで作成したGKEクラスタが表示されるようにする
```
$ gcloud container clusters get-credentials \
> $(gcloud container clusters list | grep -v "NAME" | awk '{print $1}')
```
- 無事kubectlからGKEクラスターを確認することが出来た
  - 下記のコマンドを実行して表示されるものはクラスター名ではない？[^2]
```
$ kubectl config current-context
gke_${PROJECT_ID}_asia-northeast1-a_hello-cluster
```

## 4.Gcloud SDKで作成したGKEクラスターとGitlabとを連携
### 4-1.GitlabのGUIでもろもろ設定するページに遷移する

- [https://gitlab.com](https://gitlab.com)にログインし、任意のプロジェクトをクリック
  - ここでは **`gke-quickstart`** を選択
  - このプロジェクトとは **`.gitlab-ci.yml`** がpushされるリポジトリ

![](https://storage.googleapis.com/zenn-user-upload/os7ueulapbtsey1tu5o11g1nhl4b =400x)

- **`Operations`** をクリック

![](https://storage.googleapis.com/zenn-user-upload/5eccpal33jm3v41wsxj6j6lsgior =400x)
- **`Kubernetes`** をクリック
- **`Connect cluster with certificate`** をクリック

![](https://storage.googleapis.com/zenn-user-upload/85z0jsgycjv72klux8495scuez7e =400x)

### 4-2.GitlabのGUIでもろもろ設定する
- 下記の画像のような画面にて、以下の項目を入力
  - Kubernetes cluster name
  - API URL
  - CA Certificate
  - Enter new Service Token

![](https://storage.googleapis.com/zenn-user-upload/c9nbaapjcxxtnz69kime5ubdxk3i =400x)

各項目の値の取得方法をこれから記載していきます。

- Kubernetes cluster nameの取得方法
```
$ gcloud container clusters list | grep -v "NAME" | awk '{print $1}'
hello-cluster
```
- API URLの取得方法
```
$ kubectl cluster-info | grep 'Kubernetes master' | awk '/http/ {print $NF}'
https://xx.xx.xx.xx
```
- CA Certificateの取得方法
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
- new Service Tokenの取得にあたり以下のようなyamlファイルを作成
:::details gitlab-admin-service-account.yaml

```yaml:
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: gitlab
    namespace: kube-system
```
:::

- gitlab-admin-service-account.yamlをkubectl apply
```
$ kubectl apply -f gitlab-admin-service-account.yml
serviceaccount/gitlab created
clusterrolebinding.rbac.authorization.k8s.io/gitlab-admin created
```

- サービストークンを取得
```
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
token: <サービストークン>
```
- 以下の項目はチェックがついたまま
  - **`RBAC-enabled cluster`**
  - **`GitLab-managed cluster`**
  - **`Namespace per environment`**
- 以下の項目は空白のまま
  - **`Project namespace prefix (optional, unique)`**

![](https://storage.googleapis.com/zenn-user-upload/ujmia7r0glr75o6z5s1z82s9y09i =400x)

- **`Add Kubernetes cluster`** をクリック(ここまでの設定を保存)

### 4-3.Gitlab RunnerのHelm Chartをインストール
- gitlabのnamspaceの作成とGitlabのHelm Repositoryを追加
```
$ kubectl create namespace gitlab
namespace/gitlab created
$ helm repo add gitlab https://charts.gitlab.io
"gitlab" has been added to your repositories

# Helm Repositoryが追加されたことかどうかは、以下のコマンドで確認できる
$ helm repo list
NAME    URL
gitlab  https://charts.gitlab.io
```

- Gitlab Runnerのコンフィグファイル(values.yaml)を以下のURLから任意のファイルにコピペ
  - ここでは **`values.yaml`** という名前としてコピペした
  - https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/master/values.yaml


- Gitlabの画面で **`Settings`** をクリック
- **`CI/CD`** をクリック

![](https://storage.googleapis.com/zenn-user-upload/h2mezbme12tigjt0314d2fspkgjw =400x)
- Runnesrの右隣の **`Expand`** をクリック

![](https://storage.googleapis.com/zenn-user-upload/lvfa4tkv9y5sq99ksiod3fgki4p8 =400x)
- **`Set up a specific Runner manually`** 付近の以下の太字の値をコピーし、先ほど作成した **`values.yaml`** にペースト
  - 2.Specify the following URL during the Runner setup: **`http://gitlab.your-domain.com/`**
  - 3.Use the following registration token during setup: **`xxxxxxxxxxxxxxxxx`**

![](https://storage.googleapis.com/zenn-user-upload/cewj5o4000nelrgfwvf1waqv85du =400x)

- **`values.yaml`** は先ほどコピペした **`gitlabUrl`** と **`runnerRegistrationToken`** 以外に2箇所修正する

:::details values.yaml修正箇所抜粋(行番号付き)

```yaml:
27 ## The GitLab Server URL (with protocol) that want to register the runner against
28 ## ref: https://docs.gitlab.com/runner/commands/README.html#gitlab-runner-register
29 ##
30 # gitlabUrl: http://gitlab.your-domain.com/
31 gitlabUrl: http://gitlab.your-domain.com/   #### 1箇所目 ####
32 
33 ## The Registration Token for adding new Runners to the GitLab Server. This must
34 ## be retrieved from your GitLab Instance.
35 ## ref: https://docs.gitlab.com/ce/ci/runners/README.html
36 ##
37 # runnerRegistrationToken: ""
38 runnerRegistrationToken: "xxxxxxxxxxxxxxxxx"   #### 2箇所目 ####

(略)

101 ## For RBAC support:
102 rbac:
103 #  create: false
104   create: true  #### 3箇所目 ####
105   ## Define specific rbac permissions.
106   # resources: ["pods", "pods/exec", "secrets"]
107   # verbs: ["get", "list", "watch", "create", "patch", "delete"]

(略)

176   ## Specify the tags associated with the runner. Comma-separated list of tags.
177   ##
178   ## ref: https://docs.gitlab.com/ce/ci/runners#use-tags-to-limit-the-number-of-jobs-using-the-runner
179   ##
180   # tags: ""
181   tags: "aaa,bbb,ccc,ddd"  #### 4箇所目 ####
```
:::

#### ポイントは **`values.yaml`** のtagを記載するところ。
- 複数のtagを設定する場合、tag同士を **`,(カンマ)`** で区切ること
- tagとtagとのあいだにスペースは不要

```yaml:values.yaml(再掲)
180   # tags: ""
181   tags: "aaa,bbb,ccc,ddd"
```

- **`values.yaml`** を使ってGitlab Runnerをインストール

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

# deployedとインストールできている
$ helm list -n gitlab
NAME         	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
gitlab-runner	gitlab   	1       	2020-11-26 00:28:14.721212245 +0000 UTC	deployed	gitlab-runner-0.23.0	13.6.0

# podなども作られている
$ kubectl get all -n gitlab
NAME                                               READY   STATUS    RESTARTS   AGE
pod/gitlab-runner-gitlab-runner-846756c97d-jxhgk   1/1     Running   0          30h

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gitlab-runner-gitlab-runner   1/1     1            1           30h

NAME                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/gitlab-runner-gitlab-runner-846756c97d   1         1         1       30h

```
## 6.ジョブが実行された様子

無事、ジョブが実行されることも確認できました！

![](https://storage.googleapis.com/zenn-user-upload/wj5paajani3zt714xp4rztec0d5p =400x)


[参考]使った.gitlab-ci.yml

:::details gitlab-ci.yml
```yaml
include:
  - project: gkzz/gitlab-ci-common
    ref: master
    file: .failure.yml

before_script:
  - whoami
#  - docker --version
#  - kubectl version

stages:
  - test
  - failure

test_k8s:
  stage: test
  script:
    - ls
  tags:
    - always

# 上記のtest_k8sが失敗した場合のみon_failureは実行される
# 実行される内容は外部プロジェクトの.failure.ymlを読み込む
# https://gitlab.com/gkzz/gitlab-ci-common/-/blob/master/.failure.yml
on_failure:
  stage: failure
  extends: .failure
  tags:
    - always
  when: on_failure
```
:::

-----

## 7.今後の課題
最後に本記事では解決できなかった課題を2点ほど挙げさせていただきます。

```
- GKEと接続したGitlab Runnerでdocker及びkubectlコマンドをどうやって使うか
  - Runnerが使うのはGCPのサービスアカウントなるものっぽくて、それの権限が足りないような気が。。
- manifestファイルにベタ書きしたくない値をどうやって隠蔽するか
  - 環境変数？？
```

### 7-1.GKEと接続したGitlab Runnerでdocker及びkubectlコマンドをどうやって使うか

:::details [参考]dockerコマンドをRunner上で実行したときのジョブのエラーメッセージ

```.log
Running with gitlab-runner 13.6.0 (8fa89735)
  on gitlab-runner-gitlab-runner-846756c97d-jxhgk z3hFb2PB
Resolving secrets
00:00
Preparing the "kubernetes" executor
00:00
Using Kubernetes namespace: gitlab
Using Kubernetes executor with image ubuntu:16.04 ...
Preparing environment
00:03
Waiting for pod gitlab/runner-z3hfb2pb-project-22922816-concurrent-0tm9ff to be running, status is Pending
Running on runner-z3hfb2pb-project-22922816-concurrent-0tm9ff via gitlab-runner-gitlab-runner-846756c97d-jxhgk...
Getting source from Git repository
00:01
Fetching changes with git depth set to 50...
Initialized empty Git repository in /builds/gkzz/gke-quickstart/.git/
Created fresh repository.
Checking out c1d6cabd as master...
Skipping Git submodules setup
Executing "step_script" stage of the job script
00:00
$ whoami
root
$ docker --version
/bin/bash: line 103: docker: command not found
Cleaning up file based variables
00:00
ERROR: Job failed: command terminated with exit code 1
```
:::

:::details [参考]kubectlコマンドをRunner上で実行したときのジョブのエラーメッセージ

```.log
Running with gitlab-runner 13.6.0 (8fa89735)
  on gitlab-runner-gitlab-runner-846756c97d-jxhgk z3hFb2PB
Resolving secrets
00:00
Preparing the "kubernetes" executor
00:00
Using Kubernetes namespace: gitlab
Using Kubernetes executor with image ubuntu:16.04 ...
Preparing environment
00:03
Waiting for pod gitlab/runner-z3hfb2pb-project-22922816-concurrent-09qkjk to be running, status is Pending
Running on runner-z3hfb2pb-project-22922816-concurrent-09qkjk via gitlab-runner-gitlab-runner-846756c97d-jxhgk...
Getting source from Git repository
00:02
Fetching changes with git depth set to 50...
Initialized empty Git repository in /builds/gkzz/gke-quickstart/.git/
Created fresh repository.
Checking out 8f0eb238 as master...
Skipping Git submodules setup
Executing "step_script" stage of the job script
00:00
$ whoami
root
$ kubectl version
/bin/bash: line 103: kubectl: command not found
Cleaning up file based variables
00:00
ERROR: Job failed: command terminated with exit code 1
```
:::

### 7-2.manifestファイルにベタ書きしたくない値をどうやって隠蔽するか

案としては以下の2案が考えられそうです。

```
- manifestファイルには環境変数を置き、Gitlab Runnerのジョブを実行する際、環境変数の値に書き換える
- あるいは環境変数のkeyを参照して値をRunnerが読み込めるようにする
```
.gitlab-ci.ymlにsedコマンドを書き、環境変数が書かれたmanifestファイルを書き換えようとしましたが、Syntax Errorを引いてしまいました。。

:::details sedのところでSyntax Errorを引く.gitlab-ci.yml抜粋

```.yaml:manifests/deployment.yaml.tmpl
  script:
    - pwd
    - ls
    - cp $PWD/manifests/deployment.yml.tmpl $PWD/manifests/deployment.yml 
    - sed -i -e 's|port: <SERVICE_PORT>|port: "${SERVICE_PORT}"|' \
          -i -e 's|targetPort: <FORWARD_PORT>|targetPort: "${FORWARD_PORT}"|' \
          -i -e 's|image: <CONTAINER_IMAGE>:<IMAGE_VERSION>|image: "${CONTAINER_IMAGE}":"${IMAGE_VERSION}"|' \
          -i -e 's|- containerPort: <FORWARD_PORT>|- containerPort: "${FORWARD_PORT}"|' \ 
          $PWD/manifests/deployment.yml
```
:::

:::details sedの対象のmanifests/deployment.yaml.tmpl

```yaml:manifests/deployment.yaml.tmpl
apiVersion: v1
kind: Service
metadata:
  name: hello-python-service
spec:
  selector:
    app: hello-python
  ports:
  - protocol: "TCP"
    port: <SERVICE_PORT>
    targetPort: <FORWARD_PORT>
  type: LoadBalancer

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-python
spec:
  selector:
    matchLabels:
      app: hello-python
  replicas: 4
  template:
    metadata:
      labels:
        app: hello-python
    spec:
      containers:
      - name: hello-python
        image: <CONTAINER_IMAGE>:<IMAGE_VERSION>
        #imagePullPolicy: Never
        ports:
        - containerPort: <FORWARD_PORT>
```
:::

以上が残された課題です。

## 8.GitLabの日本コミュニティのみなさまへ
本記事を執筆するにあたり、GitLabの日本コミュニティのみなさまのお力添えがなければ、公開まで到達できなかったと思います。ありがとうございました。本記事がGitlabのよりよい発展の一助となればうれしいです。Gitlab万歳！

## 9.参考
- 本記事の執筆にあたり参考にした記事
  - [GitLab CI/CD on Google Kubernetes Engine in 15 minutes or less](https://about.gitlab.com/blog/2020/03/27/gitlab-ci-on-google-kubernetes-engine/)
  - [Adding and removing Kubernetes clusters | GitLab](https://docs.gitlab.com/ee/user/project/clusters/add_remove_clusters.html)
  - [Quickstart | Kubernetes Engine Documentation | Google Cloud](https://cloud.google.com/kubernetes-engine/docs/quickstart)
  - [GitLab Runnerをkubernetes上でラフに動かしてみよう #gitlab #kubernetes #gitlab-jp #developer #devops](https://www.creationline.com/lab/gitlab/33461)
  - [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
  - [GitLabの継続的インテグレーション(CI)と継続的デリバリー(CD)](https://www.gitlab.jp/stages-devops-lifecycle/continuous-integration)
  - [Deploying a containerized web application](https://cloud.google.com/kubernetes-engigcloud%20container%20clusters%20get-credentialsne/docs/tutorials/hello-app)
- Gitlabの日本コミュニティ
  - [GitLab connpass](https://gitlab-jp.connpass.com/)
  - [GitLab-JP Slack](https://gitlab-jp.herokuapp.com)


## P.S. Twitterもやってるのでフォローしていただけると泣いて喜びます:)
[@gkzvoice](https://twitter.com/gkzvoice)

[^1]: GCPプロジェクトのIDの確認方法について。GCPコンソール画面からダッシュボードを開くと右記のようなURLとなっている。このURLの${PROJECT_ID}がGCPプロジェクトのID。（https://console.cloud.google.com/home/dashboard?project=${PROJECT_ID}）
[^2]: 注釈を付けたコマンドを実行して表示されるものはクラスターの名前の頭に${PROJECT_ID}などがついていて、なんなのかよくわかっていない。。
