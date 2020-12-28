# 1. はじめに

この文書では、`UPI(User Provisioned Infrastructure)`を使って `OCP(OpenShift Container Platform)`の環境を構築していきます。省略系の`OCP`だとわかりにくいので、ここでは`OpenShift`と呼ぶ事にします。

本文書は本章も含めて、合計4章で構成します。

### 1. はじめに(本章)
本文書のテーマであるOpenShiftのクラスターインストールと、本文書内の手順で行う内容を簡単に紹介します。
 
### 2. [OpenShiftの基礎](../2_Overview/)
クラスターインストールを行ううえで知っておくべき、OpenShift(およびベースとなるKubernetes)の基礎知識を説明します。

### 3. [クラスターインストールの計画](../3_Planning/)
インストールするクラスターの設計や、インストールに必要な周辺サービスの設定を説明します。

### 4. [クラスターインストールの実行](../4_Installation/)
クラスターインストールを実行する手順を順番に説明します。

<br>

---

## 1-1. OpenShiftとは

`OpenShift`とは、`Red Hat`が提供する商用版の `Kubernetes` です。

オープンソースの`Kubernetes`に、運用に必要な独自機能を追加したり、オープンソースの運用管理ツールや、開発ツールをバンドルした製品です。もちろんサポートが付いているのも大きな違いです。

例えば、監視用のOSSである`Prometheus`や、ログ収集用の`EFK`(`ElasticSearch`/`Fluented`/`Kafka`)等の他に、`CI/CD`ツールである`Jenkis`、`Tekton` もバンドルされています。

また、`Red Hat`の Middleware群も`OpenShift`上での使用に限り、提供されており、とりあえず `OpenShift`を持っていればコンテナ環境を構築・運用するためのツール類が一通り揃っている。という形になっています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/99425/24e2fe39-71ae-2c20-b9e3-90be616c5b79.png)

## 1-2. インストール方法
`OpenShift`のインストール方法は2つに大別されます。

### 1-2-1. UPIインストール

`UPI(User Provisioned Infrastructure)`インストールとは、ユーザーが自分で `OpenShift`の環境構築に必要な、OSをインストールしたノードや、`DNS`の設定、ロードバランサーなどの環境をまず整え、その上にソフトウェアとしての`OpenShift`(`Kubernetes`と、運用に必要な各種ツール)をインストールする方法です。

### 1-2-2. IPIインストール

`UPI`インストールの対義語として`IPI(Installer Provisioned Infrastracture)`という言葉があります。こちらは、`AWS`、`Azure`、`GCP`のような Public Cloud Providerが持っているインフラ構築用の APIを使用して `UPI`で手作業で行っていた、ノードの作成、ロードバランサーの設定、`DNS`の設定等を`OpenShift`のインストーラーが自動で行う方法です。

正確を期すと、`Public Cloud Provider`のアカウントを準備しておく事と、`OpenShift`で使用するドメインの取得と取得したドメインを、その`Public Cloud Provider`の`DNS`サービスに登録しておく事は事前にする必要があります。

`IPI`は、インストーラーを実行すると指定した数の`ノード`をデプロイし、`DNSサーバー`も自動で構成されるなど非常に楽なので、私もテストなどではこの方法を使用しています。

が、構成に細かなカスタマイズが必要な場合や、`OpenShift`のインストーラーが`IPI`に対応してない環境（例えばオンプレミス)に環境を作成したい場合は、`UPI`による`OpenShift`のインストールが必要になります。

`IPI`インストールができる環境は、`DNSサーバー`や`Load Balancer`や、ネットワークの構成をインストーラーからAPIを使用して構築・構成する必要があります。そのためIPIがサポートしている環境は`AWS`や`Azure`、`GCP`等の環境に限られています。が、後述しますが`IPI`の意味する所は最近、少し広がってきたようで、`Baremetal IPI` という言葉も出てきました。

### 1-2-3. IPIとUPI

物理環境では厳密な意味での完全な`IPI`インストールというのはあり得ません。例えば物理環境では、少なくてもルーターやサーバーを設置してケーブルを配線するという環境準備が必要です。

しかし、`OpenShift 4.6`では、ネットワークの環境が整っていれば、`OpenStack`の`Ironic`等で利用されている`Bare Metal Provisioning`の技術を利用する事でインストーラーから`ノード`の`OS`のデプロイを実行できるようになりました。これも`Bare Metal IPI`と呼ぶようになりました。ただこの場合も、DNSやルーティングなどは、事前に手動でセットアップする必要があるので、`Public Cloud Provider`上の`IPI`とはできる範囲が違います。

参考：<a href="https://openshift-kni.github.io/baremetal-deploy/4.6/Deployment.html">Deploying Installer Provisioned Infrastructure (IPI) of OpenShift on Bare Metal - 4.6</a>


## 1-3. この文書で実行する内容

この文書では、物理環境であればどこにでも適用できるインストール方法である`Baremetal`環境への `UPI(User Provisioned Infrastructure)`インストールを`OpenShift 4.5`を使用して実行してみます。

`UPI(User Provisioned Infrastructure)`ですので、自分で、`OpenShiftt`を稼働させるための`DNS`サーバー、`Load Balancer`、`Kubernetes`の`Masterノード`、`Workerノード`等のインストールなどの周辺環境を`Provision`してあげる必要があります。

`BareMetal`と書いていますが、仮想マシンを物理マシンと同等とみなせる仮想化環境でも手順は同じです。

※`VMware`環境の場合は、`OpenShift` 4.5から、`VMWare vSphere IPI`インストールという方式が出てきましたが、ここでは触れません。