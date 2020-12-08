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
hoge@hoge:~/gke-quickstart/app$ kubectl cluster-info | grep 'Kubernetes master' | awk '/http/ {print $NF}'
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
```
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

### 4-3.HelmでGitlabのchartをインストール
- gitlabのnamspaceの作成とGitlabのhelm repositoryを追加
```
$ kubectl create namespace gitlab
namespace/gitlab created
$ helm repo add gitlab https://charts.gitlab.io
"gitlab" has been added to your repositories
```

- 以下のコマンドで追加されたことが確認できる
```
$ helm repo list
NAME    URL
gitlab  https://charts.gitlab.io
```

- Gitlab Runnerのコンフィグファイル(values.yaml)を以下のURLから任意のファイルにコピペ
  - ここでは`values.yaml`という名前としてコピペした
- https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/master/values.yaml


- Gitlabの画面で

- **`Operations`** をクリック
![](https://storage.googleapis.com/zenn-user-upload/h2mezbme12tigjt0314d2fspkgjw =400x)

![](https://storage.googleapis.com/zenn-user-upload/cewj5o4000nelrgfwvf1waqv85du =400x)

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
## 6..gitlab-ci.ymlの工夫

-----

## 7.今後の課題
最後に本記事では解決できなかった課題を2点ほど挙げさせていただきます。
```
- GKEと接続したGitlab Runnerでdocker及びkubectlコマンドをどうやって使うか
- manifestファイルにベタ書きしたくない値をどうやって隠蔽するか
  - manifestファイルには環境変数を置き、Gitlab Runnerのジョブを実行する際、環境変数の値に書き換える
  - あるいは環境変数のkeyを参照して値をRunnerが読み込めるようにする
```

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
