# 2. OpenShiftの基礎

## 2-1. Kubernetes と OpenShiftの違い
`OpenShift` は `Red Hat` が提供する商用の `Kubernetes` ディストリビューションの一つです。
オープンソースである `Kubernetes` の商用サポートの他、監視をはじめとする `Kubernetes` クラスターの定常運用を実現する機能群を備えています。
また `CI/CD` や `ServiceMesh`、`Serverless` などの機能も容易に追加できます。

`OpenShift` では、`Kubernetes` を制御するためのサービス以外にもさまざまな内部コンポーネントが稼働しています。
たとえば、コンテナレジストリや、コンテナアプリケーションへのトラフィックを管理する `Router` などです。
また、充実した `Graphical User Interface` が標準機能として用意されているため、運用者が別のツールを選択したり、導入に時間を割いたりする必要はありません。
`OpenShift` はアップストリームの `Kubernetes` を包含しており、`etcd` の `Quorum` を担保するため、`Master Node` が 3台、`Worker Node` が 2台以上の構成が推奨されています。
また、`Operator` というテクノロジーを活用することによって、`OpenShift` クラスターのインストール、構成変更、アップデート、機能追加など容易にしています。
このように、企業が `Kubernetes` を利用するにあたって、`Kubernetes` だけでは不足しているであろう運用機能や開発生産性を高められる機能、組織など複数人での使い勝手を便利にする権限設定などを追加しています。

サポート期間については、`Kubernetes v1.19` から一年間となり、`OpenShift 4.6` では、`EUS(Extended Update Support)` が利用可能になるなど、長期サポートを要するユーザーには嬉しい変更がなされています。

## 2-2. 利用形態
ユーザーは2種類の利用形態を選ぶことができます。一つはクラウドプロバイダーが提供する `マネージドOpenShift` を利用する形態(`Hosted OpenShift`)です。
もう一つはオンプレミスなどに自身で構築して利用する `セルフマネージドOpenShift` の形態です。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/69447/e47f8cdd-e984-44e3-c5e6-b6b02a1524f8.png)

### 2-2-1. マネージド OpenShift
様々なクラウドプロバイダーにとって `マネージドOpenShift` が提供されています。各サービスのプランを契約して使用します。

- OSD
  - OpenShift Dedicated
  - `AWS` または `GCP` 上で利用できます。`Red Hat`社の `SRE` が運用し、サポートを提供します。
- ROSA
  - Red Hat OpenShift Service on AWS
  - `AWS` 上で利用できます。`AWS`社がサポートを提供します。
- ARO
  - Azure Red Hat OpenShift
  - `Azure` 上で利用できます。`Microsoft`社がサポートを提供します。
- ROKS
 - Red Hat OpenShift on IBM Cloud (Kubernetes Service)
 - `IBM Cloud` 上で利用できます。`IBM`社がサポートを提供します。

※各種マネージドサービスにおいて、`Red Hat` のサポートチームも各社と連携したサポートを提供します。

### 2-2-2. セルフマネージド OpenShift
クラスター構成に合わせたサブスクリプションを購入し、自身でオンプレミスやクラウドリソースを使用して `OpenShift` を構築できます。

## 2-3. インストール
前述の通り、ユーザーはクラウドプロバイダーが提供する `マネージドサービスOpenShift` を利用できますが、自身でオンプレミス環境やクラウドリソースを使用して `OpenShift` を構築することができます。`IPI` と `UPI` の2つの方法があります。

それぞれのインストール方法が対応するクラウドプロバイダーやプラットフォームは以下です。（2020年11月11日時点）

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/69447/302bee42-26b6-b36f-9a73-a90ca8a1cc0b.png)


### 2-3-1. IPI (Installer-Provisioned Infrastructure)
`IPI` は、インストーラを用いて動的に仮想マシンやロードバランサ、ストレージなどを準備し、`OpenShift` クラスターをインストールする方法です。

### 2-3-2. UPI (User-Provisioned Infrastructure)
`UPI` は、事前に仮想マシンなどのリソースを準備し、`OpenShift` クラスターをインストールする方法です。オンプレ環境やインターネット接続に制限のある環境や、既存のIT資産を活用する場合に使われます。

## 2-4. OpenShift基本構成
`OpenShift` が備える基本構成について一部紹介します。

### 2-4-1. クラスター構成
`OpenShift` では、`Kubernetes` と同様に、`Master Node` と `Worker Node` によってクラスターが組まれます。`OpenShift` では `Infra Node` を追加し、モニタリングサービスやコンテナイメージレジストリやルーティングをサポートする機能などを `Master Node` や `Worker Node` ではなく、`Infra Node` に委譲させることもできます。今回紹介する `Baremetal` 環境への `UPI`インストール方法では、`Infra Node` もクラスター構成に含みます。

### 2-4-2. 拡張コンポーネント
`OpenShift` の `Master Node` には、`Kubernetes` のクラスタ制御サービスが起動しています。`Master Node` で稼働する `Kubernetes` のサービスは「`Kubernetes API` (`kube-apiserver`)」、「`etcd` (`etcd-member`)」、「`Kubernetes Controller Manager` (`kube-controller-manager`)」、「`Kubernetes Scheduler` (`kube-scheduler`)」があります。さらに `OpenShift` では、`Kubernetes` コンポーネントを補った拡張コンポーネントが提供されています。たとえば、`Kubernetes` の中核となる `Kubernetes API` (`openshift-kube-apiserver`) に対して、機能を拡張した `OpenShift API` (`openshift-apiserver`) が `Operator` 経由で展開されています。`OpenShift` では、「`oc`」コマンドという `OpenShift` 専用の管理コマンドを利用することで、コンテナアプリのビルドやクラスタ管理操作を要求できます。ただし、`OpenShift API` はあくまで `Kubernetes API` を拡張しているだけにすぎず、アップストリーム版の `Kubernetes` 同様に「`kubectl`」コマンドを利用できるように構成されています。

`OpenShift` の `Worker Node` では、`Kubernetes` のコアサービス以外にもノード監視を行うための `Prometheus` のエージェント（`Node Exporter`） やクラスタ間のネットワークを経由する `CNIプラグイン`（`Multus`）などが存在します。これらは `OpenShift` を構成するうえで欠かせない機能であるため、`Master Node` 上でも同様のコンテナが稼働しています。他にも `OpenShift` では、`Master Node` を含め「`MachineSet`」というオブジェクトがノードを制御しています。これは、`Kubernetes`における `Pod` と `ReplicaSet` の関係と似た機能です。つまり、アプリケーションのワークロードに応じて `MachineSet` の`replicasフィールド`を変更することで、動的なノードの追加や縮退が制御できます。このように `Worker Node` はアプリケーションワークロードに応じて動的に管理されるものであり、ユーザーが意識して `Worker Node` 上のサービスを制御することは望まれません。逆にアプリケーション以外は常にイミュータブル な存在であることが、`OpenShift` が目指す `Kubernetes` クラスタの運用スタイルです。

### 2-4-3. クラスタモニタリング
クラスタの稼働メトリクス情報は「`openshift-monitoring`」というプロジェクト内の `Operator` が管理する、`Prometheus` によって取得されています。また、取得したメトリクス情報は `Grafana` にて可視化されます。可視化に関しては、 `Grafana` 以外にも `OpenShift` ポータル上から確認することも可能です。このコンテナのメトリクス取得には「`kube-state-metrics`」が利用されています。`kube-state-metrics` とは、`Kubernetes API` を監視し、`OpenShift` が 提供する `Pod` や `Deployment`、`DaemonSet` などのオブジェクトに対して、メトリクス情報を取得します。したがって、`Pod` の `CPU` や`メモリ使用率` などは、特別な設定を行わずとも `OpenShift` 上で監視できます。ただし「`kube-state-metrics`」が取得する値以外の固有アプリケーションのメトリクスは、自分で設定しなければいけません。`Prometheus` の運用そのものは `Prometheus Operator` が行いますが、取得するメトリクスの指定はユーザー自身が決めて対応します。また、`Prometheus Operator` では `AlertManager` も提供できます。`AlertManager` を使うことによって、`Prometheus` が取得したメトリクスをトリガーに、`Webhook` を呼び出してアラートをあげることも可能です。

### 2-4-4. クラスタロギング
クラスタのログ基盤には `EFK`(`Elasticsearch`、`Fluentd`、`Kibana`)スタックを活用します。ログコレクターとして `Fluentd` が利用され、 各`Worker Node` 上に `DaemonSet` として展開されます。これによってすべてのノードのログを収集し、`Elasticsearch` に蓄積して分析を行います。また、開発者やクラスタ管理者が集計されたデータを分析するために `Kibana` が可視化しています。ただし、事前に `Elasticsearch` 用のクラウドリソースを確保しないといけないため、クラスタインストール時には `EFKスタック` は稼働していません。必要に応じて管理者が、`Cluster Logging Operator` と `Elasticsearch Operator` の 双方を準備し、`EFKスタック` を構築します。`Cluster Logging Operator` は、ログの収集(`Collection`)、保存(`LogStore`)、視覚化(`Visualization`)に関する、`EFK` の設定を行うインターフェースです。また、`Cluster Logging Operator` によって、`Elasticsearch` と `Kibana` は `OpenShift` との認証連携が行われています。したがって、一般の開発者は所属するプロジェクトのログのみ参照できるといった制限も可能です。

### 2-4-5. クラスタネットワーク
`Kubernetes` におけるコンテナ間のネットワークは、`Container Network Interface` (`CNI`)を実装するネットワークプラグインに依存します。ネットワークプラグインがオーバーレイネットワークを作ることで、各 `Worker Node` に配置されるコンテナ同士が疎通できます。`OpenShift` もこの `CNI` に準拠しており、さまざまなネットワークプラグインをサポートして います。ただし、デフォルトでは「`OpenShift SDN`」という `Open vSwitch`(`OVS`)ベース の `CNI プラグイン`を使用し、オーバーレイネットワークを構築します。`OpenShift SDN` は、`Pod` ネットワークを設定するため 3つのモードを提供しています。

- NetworkPolicy Mode
プロジェクト管理者が`NetworkPolicy` オブジェクトを使用して、独自の分離ポリシーを設定できるモード。`OpenShift` のデフォルトモード

- MultiTenant Mode
`Pod` およびサービスのプロジェクトを分離するモード。異なるプロジェクトの `Pod` および `Service` は、パケットの送受信ができない。疎通する場合は、管理者によって個別に分離を解除する

- Subnet Mode
すべての `Pod` がほかのすべての `Pod` および `Service` と通信できるモード。`NetworkPolicy Mode` も同様に通信が許可されているが、 `NetworkPolicy` の利用ができるかどうかが異なる

こうした、ネットワークプラグインも `Cluster Network Operator`(`CNO`) によって管理されています。`CNO` は、`DaemonSet`を使って `OpenShift SDN`(または指定したネットワークプラグイン)を管理します。また `OpenShift 4` では、`Multus CNI` というマルチネットワークプラグインを利用しています。`Multus CNI` は、`Pod` に対して複数の `NIC`(`Network Interface Card`)を提供します。こうした複数 `NIC` によるトラフィック分離は、仮想マシンを利用していたときのように、コンテナのデータプレーンとコントロールプレーンを切り離し、パフォーマンスを改善します。さらに、特定のプライベートネットワークを構築することで、セキュリティ面の強化も図れます。エンタープライズでは、専用の `VLAN` ネットワークを稼働している場合やトラフィック制限を管理しなければいけないことがあります。 `Multus CNI` は、こうした要件にも耐えられるように日々開発が行われています。

## 2-5. OpenShift 参考情報
- [OpenShift ウェビナー](https://www.redhat.com/ja/explore/openshift/webinar)
    - OpenShift スペシャリストが基本のキからマニアックなことまで紹介しているウェビナー
    - 動画アーカイブ有り
    - これから OpenShift 始めたい方にオススメ
- [OpenShift Docs Portal](https://docs.openshift.com/)
    - Self-managed, Managed, 他, ドキュメントリンク
    - 様々な提供形態の OpenShift に関して Docs を確認したい方向け
- [赤帽エンジニアブログ-OpenShift](https://rheb.hatenablog.com/archive/category/Red%20Hat%20OpenShift)
    - 社員有志によるブログ
    - 楽しく読みつつ、実践的ノウハウを学びたい方向け
- [OpenShift Community Japan](https://openshift.connpass.com/)
    - テックコミュニティ
    - 一部動画アーカイブ有り
    - Kubernetes が好きなヒトにオススメ

---

## → Next: [3. クラスターインストールの計画](../3_Planning/README.md)
