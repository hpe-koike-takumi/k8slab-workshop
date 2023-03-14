# 第三回ハンズオン・ワークショップ

## 概要

本ハンズオンは、Kubernetesクラスタの拡張とアップグレードのハンズオンです。

以下のトピックを実施します。

- ノード追加によるKubernetesクラスタの拡張
- Kubernetesクラスタのアップグレード
- Kubernetesクラスタからノード削除

### 前提

- `kubeadm`によるKubernetesクラスタが導入済みであること
- 参加者ユーザに`cluster-admin`権限を割り当てていること（講師側で実施します）

### 留意事項

- 本資料では Kubernetesの導入手順については説明しません。
- 本資料では AWSのリソース導入手順については説明しません。
- ノード追加ハンズオンでは、参加者各自に1台インスタンスを割り当てます。各参加者は手順に従い、インスタンスをKubernetesクラスタのWorker Nodeとして追加します。
- アップグレードハンズオンでは、各参加者はノード追加ハンズオンで追加したWorker Nodeのアップグレードを実施します。（追加したWorker Nodeは1マイナーバージョン古いバージョンです。Control Planeと同じバージョンにするためにバージョンアップを行います）

本ハンズオンの実施内容はこちら

- ノード追加ハンズオン
- ノードアップグレードハンズオン

## クラスタの拡張

Kubernetesクラスタは、Control Plane Node と Worker Node から構成されており、ユーザのコンテナアプリケーションは、Worker Node 上で稼働します。コンテナアプリケーションのリソース要求に対してクラスタのキャパシティが不足したり、Worker Node に不具合があったりした場合は、クラスタに Worker Node を追加する必要があります。

本ハンズオンでは、上記のような状況での手動によるWorker Nodeの追加を行います。

### Cluster Autoscaler

手動によるPodのスケールに対して、`HorizontalPodAutoscaler`が自動でPod数の増減を調整を行うように、クラスタのノードに対しても同様にノード数調整を自動で行う`Cluster Autoscaler`と呼ばれるコンポーネントが存在します。

https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler

パブリッククラウドのマネージドサービスなどのKubernetesディストリビューションでは、Cluster AutoscalerがIaaSと連携してノードの自動増減を行います。

特徴としては、クラスタへPodをデプロイしたさいにPending状態のPodがあれば、Cluster Autoscalerがオートスケーリングを行うことが挙げられます。

## ハンズオン手順（ノード追加ハンズオン）

ハンズオン開始前に講師が以下の情報をお伝えします。

- 追加用インスタンス情報（アカウント情報一覧でお伝えします）
- インスタンスSSH接続用秘密鍵
- ノード追加用コマンド

1. 講師より割り当てられたインスタンスを確認します。

2. bastionサーバより割り当てられたインスタンスへログインします。

   ```shell
   ssh -i [private-key] ubuntu@[your instance ip]
   ```

   ログイン後パッケージ情報を更新します。

   ```shell
   sudo apt -y update
   ```

3. ノード追加コマンドを実行します。

   割り当てられたノード上でクラスタへのノード追加コマンドを実行します。（コマンドは講師よりお伝えします）

   ```shell
   sudo kubeadm join hpe-k8slab-teacher-cls-nlb-823832ea89b82cce.elb.ap-northeast-3.amazonaws.com:6443 --token 9mq2db.scyy6tzi8b24d9br --discovery-token-ca-cert-hash sha256:d82a21bad4d13481b909671c60bc37c836a5bfa7b65b26f76a70aa83431a8961
   ```

   ノード追加時に使用するTokenの有効期限は24時間です。

   ノードからログアウトします。

   ```shell
   exit
   ```

4. bastionサーバでノードの確認を行います。

   ```shell
   kubectl get nodes
   ```

   割り当てられたノードがWorker Nodeとして追加されていることを確認します。

## Kubernetes アップグレード

Kubernetesは更新スピードが早く、サポート対象も最新3マイナーバージョンで、機能追加やバグ修正、セキュリティパッチという観点で塩漬けではなく、アップグレードを運用の一環として継続的に実施していくことが推奨されます。

実際にアップグレードを実施するにあたっては、事前に更新内容の調査を行い、変更の影響ないかテストをして、アップグレード戦略・計画をたててアップグレード作業に臨みますが、本ハンズオンでは、クラスタ構築に使用している`kubeadm`によるWorker Nodeのアップグレードハンズオンを行います。

Kubernetesクラスタのコンポーネントのアップグレード順序

1. Control Plane
2. Worker Node
3. クラスタ上で利用する運用管理系ツール（Operatorなど）

## ハンズオン手順（アップグレードハンズオン）

本ハンズオンでは、クラスタを参加者で共有しているので、参加者はノード追加ハンズオンで追加した1マイナーバージョン古いWorker Nodeのアップグレードを実施してControl Planeバージョンと同一バージョンにします。

本資料では、参考として、Control Planeのアップグレード手順をAppendixとして掲載しております。

### Worker Nodeのupgrade

1. kubeadmのupgrade

   割り当てられたWorker nodeへsshでログインします。

   以下のコマンドを実行しkubeadmをupgradeします。

   本環境では、v1.25.xのWorker Nodeをv1.26.0にアップグレードします。

   ```shell
   sudo apt-mark unhold kubeadm && \
   sudo apt-get update && sudo apt-get install -y kubeadm=1.26.0-00 && \
   sudo apt-mark hold kubeadm
   ```

   実行例

   ```shell
   $ sudo apt-mark unhold kubeadm && \
   sudo apt-get update && sudo apt-get install -y kubeadm=1.26.0-00 && \
   sudo apt-mark hold kubeadm
   kubeadm の保留を解除しました。
   ヒット:1 https://download.docker.com/linux/ubuntu jammy InRelease
   ヒット:2 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu jammy InRelease
   取得:3 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu jammy-updates InRelease [114 kB]
   取得:4 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu jammy-backports InRelease [99.8 kB]
   取得:6 http://security.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
   ヒット:5 https://packages.cloud.google.com/apt kubernetes-xenial InRelease
   324 kB を 1秒 で取得しました (307 kB/s)
   パッケージリストを読み込んでいます... 完了
   W: https://download.docker.com/linux/ubuntu/dists/jammy/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
   W: https://apt.kubernetes.io/dists/kubernetes-xenial/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
   パッケージリストを読み込んでいます... 完了
   依存関係ツリーを作成しています... 完了
   状態情報を読み取っています... 完了
   以下のパッケージはアップグレードされます:
     kubeadm
   アップグレード: 1 個、新規インストール: 0 個、削除: 0 個、保留: 14 個。
   9,730 kB のアーカイブを取得する必要があります。
   この操作後に追加で 2,961 kB のディスク容量が消費されます。
   取得:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubeadm amd64 1.26.0-00 [9,730 kB]
   9,730 kB を 1秒 で取得しました (16.3 MB/s)
   (データベースを読み込んでいます ... 現在 64036 個のファイルとディレクトリがインストールされています。)
   .../kubeadm_1.26.0-00_amd64.deb を展開する準備をしています ...
   kubeadm (1.26.0-00) で (1.25.4-00 に) 上書き展開しています ...
   kubeadm (1.26.0-00) を設定しています ...
   Scanning processes...
   Scanning linux images...
   
   Running kernel seems to be up-to-date.
   
   No services need to be restarted.
   
   No containers need to be restarted.
   
   No user sessions are running outdated binaries.
   
   No VM guests are running outdated hypervisor (qemu) binaries on this host.
   kubeadm は保留に設定されました。
   ```

2. kubeadm upgradeの実行

   以下のコマンドでworker nodeのコンポーネントのupgradeを実行します。

   ```shell
   sudo kubeadm upgrade node
   ```

   実行例

   ```shell
   $ sudo kubeadm upgrade node
   [upgrade] Reading configuration from the cluster...
   [upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
   [preflight] Running pre-flight checks
   [preflight] Skipping prepull. Not a control plane node.
   [upgrade] Skipping phase. Not a control plane node.
   [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
   [upgrade] The configuration for this node was successfully updated!
   [upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
   ```

   一旦worker nodeからログアウトします。

3. Worker nodeのdrain

   以下のコマンドでWorker nodeをdrainします。

   \<node\>は割り当てられたWorker Node名に置き換えてください。

   ```shell
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
   ```

   実行例

   ```shell
   $ kubectl drain ip-192-168-20-211 --ignore-daemonsets --delete-emptydir-data --force
   node/ip-192-168-20-211 cordoned
   Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-jmbk7, ingress-nginx/ingress-nginx-controller-b52z5, kube-system/kube-multus-ds-5jrr8, kube-system/kube-proxy-7p8pb, monitoring/monitoring-prometheus-node-exporter-86867; deleting Pods that declare no controller: default/dnsutils
   evicting pod calico-system/calico-typha-5597444667-jzj4g
   evicting pod sample-app/sample-nad-app-7d857498c8-r9xs6
   evicting pod calico-apiserver/calico-apiserver-7b46f8789b-sdtr6
   evicting pod default/dnsutils
   evicting pod monitoring/alertmanager-monitoring-kube-prometheus-alertmanager-0
   evicting pod monitoring/monitoring-kube-state-metrics-5d8cb49c5d-8nqsw
   evicting pod sample-app/sample-http-app-6f88d79b5-m5h96
   pod/calico-typha-5597444667-jzj4g evicted
   pod/calico-apiserver-7b46f8789b-sdtr6 evicted
   pod/monitoring-kube-state-metrics-5d8cb49c5d-8nqsw evicted
   pod/sample-nad-app-7d857498c8-r9xs6 evicted
   pod/dnsutils evicted
   pod/sample-http-app-6f88d79b5-m5h96 evicted
   pod/alertmanager-monitoring-kube-prometheus-alertmanager-0 evicted
   node/ip-192-168-20-211 drained
   ```

   以下のコマンドでノードがSchedulingDisabled状態であることを確認します。

   ```shell
   kubectl get nodes
   ```

4. kubeletとkubectlのupgrade

   再度worker nodeへsshログインします。ログイン後、以下のコマンドを実行してkubeletをアップグレードします。

   ```shell
   sudo apt-mark unhold kubelet && \
   sudo apt-get update && sudo apt-get install -y kubelet=1.26.0-00 && \
   sudo apt-mark hold kubelet
   ```

   実行例

   ```shell
   $ sudo apt-mark unhold kubelet kubectl && \
   sudo apt-get update && sudo apt-get install -y kubelet=1.26.0-00 && \
   sudo apt-mark hold kubelet kubectl
   kubelet の保留を解除しました。
   ヒット:1 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu jammy InRelease
   ヒット:2 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu jammy-updates InRelease
   ヒット:3 https://download.docker.com/linux/ubuntu jammy InRelease
   ヒット:4 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu jammy-backports InRelease
   ヒット:5 https://packages.cloud.google.com/apt kubernetes-xenial InRelease
   ヒット:6 http://security.ubuntu.com/ubuntu jammy-security InRelease
   パッケージリストを読み込んでいます... 完了
   W: https://download.docker.com/linux/ubuntu/dists/jammy/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
   W: https://apt.kubernetes.io/dists/kubernetes-xenial/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
   パッケージリストを読み込んでいます... 完了
   依存関係ツリーを作成しています... 完了
   状態情報を読み取っています... 完了
   以下のパッケージはアップグレードされます:
     kubelet
   アップグレード: 1 個、新規インストール: 0 個、削除: 0 個、保留: 104 個。
   20.5 MB のアーカイブを取得する必要があります。
   この操作後に追加で 7,006 kB のディスク容量が消費されます。
   取得:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubelet amd64 1.26.0-00 [20.5 MB]
   20.5 MB を 3秒 で取得しました (6,014 kB/s)
   (データベースを読み込んでいます ... 現在 64034 個のファイルとディレクトリがインストールされています。)
   .../kubelet_1.26.0-00_amd64.deb を展開する準備をしています ...
   kubelet (1.26.0-00) で (1.25.5-00 に) 上書き展開しています ...
   kubelet (1.26.0-00) を設定しています ...
   Scanning processes...
   Scanning linux images...
   
   Running kernel seems to be up-to-date.
   
   No services need to be restarted.
   
   No containers need to be restarted.
   
   No user sessions are running outdated binaries.
   
   No VM guests are running outdated hypervisor (qemu) binaries on this host.
   kubelet は保留に設定されました。
   ```
   
5. 設定再読み込み&kubeletの再起動

   ```shell
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet
   ```

   再度、worker nodeからログアウトします。

6. Nodeのuncordon

   kubelet/kubectlをupgradeしたNodeをuncordonして、podをスケジューリングできるように戻します。

   \<node\>は割り当てられたWorker Node名に置き換えてください。

   ```shell
   kubectl uncordon <node>
   ```

   （通常は、以上の手順を各worker nodeで繰り返します。）

   すべてのnodeで処理が完了するとアップグレードは終了です。

   ```shell
   $ kubectl get nodes
   NAME                STATUS   ROLES           AGE   VERSION
   ip-192-168-20-137   Ready    control-plane   14d   v1.26.0
   ip-192-168-20-156   Ready    compute         14d   v1.26.0
   ip-192-168-20-184   Ready    control-plane   14d   v1.26.0
   ip-192-168-20-211   Ready    compute         14d   v1.26.0
   ip-192-168-20-217   Ready    compute         14d   v1.26.0
   ip-192-168-20-64    Ready    control-plane   14d   v1.26.0
   ```

## Kubernetes ノード削除

追加したWorker Nodeをクラスタから削除します。

1. ワークロードの退避

   削除する前に、該当ノード上で稼働しているPodを退避します。

   ```shell
   kubectl drain [Your Node Name] --ignore-daemonsets --delete-emptydir-data --force
   ```

2. ノードの削除

   以下のコマンドでノードを削除します。

   ```shell
   kubectl delete node [Your Node Name]
   ```

   Kubernetes クラスタとしてのノード削除は以上です。その後、プラットフォームに合わせて仮想マシンインスタンスの削除を行います。

## Appendix

### Control Plane のアップグレード

本資料では、ハンズオン環境の特性上Worker Nodeのアップグレードを実施いただきましたが、通常Kubernetesクラスタ運用でのアップグレードでは、まずControl Planeのアップグレードを行います。当Appendixは、参考として、`kubeadm`によるControl Planeのアップグレード手順を紹介いたします。

1. 1台目のControl Plane Nodeへログインします。

   ログイン後、パッケージ情報を更新します。

   ```shell
   sudo apt update
   ```

2. Upgrade先バージョンを決定します。

   Upgrade先kubeadmのバージョンを下記コマンドで確認します。

   ```shell
   $ sudo apt-cache madison kubeadm
      kubeadm |  1.26.0-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.25.6-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.25.5-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.25.4-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.25.3-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.25.2-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.25.1-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.25.0-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm | 1.24.10-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.9-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.8-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.7-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.6-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.5-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.4-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.3-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.2-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.1-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm |  1.24.0-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm | 1.23.16-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm | 1.23.15-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm | 1.23.14-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm | 1.23.13-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm | 1.23.12-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm | 1.23.11-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      kubeadm | 1.23.10-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
      （以下略）
   ```

   本手順では、「1.26.0-00」へUpgradeするとします。

3. kubeadmのupgrade

   ```shell
   sudo apt-mark unhold kubeadm && \
   sudo apt-get update && sudo apt-get install -y kubeadm=1.26.0-00 && \
   sudo apt-mark hold kubeadm
   ```

   実行例

   ```shell
   $ sudo apt-mark unhold kubeadm && \
   > sudo apt-get update && sudo apt-get install -y kubeadm=1.26.0-00 && \
   > sudo apt-mark hold kubeadm
   Canceled hold on kubeadm.
   Hit:2 https://download.docker.com/linux/ubuntu focal InRelease
   Hit:3 http://in.archive.ubuntu.com/ubuntu focal InRelease
   Get:4 http://in.archive.ubuntu.com/ubuntu focal-updates InRelease [114 kB]
   Hit:1 https://packages.cloud.google.com/apt kubernetes-xenial InRelease
   Get:5 http://in.archive.ubuntu.com/ubuntu focal-backports InRelease [108 kB]
   Get:6 http://in.archive.ubuntu.com/ubuntu focal-security InRelease [114 kB]
   Fetched 336 kB in 2s (211 kB/s)
   Reading package lists... Done
   Reading package lists... Done
   Building dependency tree
   Reading state information... Done
   The following packages will be upgraded:
     kubeadm
   1 upgraded, 0 newly installed, 0 to remove and 62 not upgraded.
   Need to get 8580 kB of archives.
   After this operation, 643 kB disk space will be freed.
   Get:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubeadm amd64 1.23.3-00 [8580 kB]
   Fetched 8580 kB in 3s (3130 kB/s)
   (Reading database ... 144554 files and directories currently installed.)
   Preparing to unpack .../kubeadm_1.23.3-00_amd64.deb ...
   Unpacking kubeadm (1.23.3-00) over (1.22.6-00) ...
   Setting up kubeadm (1.23.3-00) ...
   kubeadm set on hold.
   ```

4. kubeadmのバージョンを確認します。

   ```shell
   kubeadm version
   ```

   実行例

   ```shell
   kubeadm version: &version.Info{Major:"1", Minor:"26", GitVersion:"v1.26.0", GitCommit:"b46a3f887ca979b1a5d14fd39cb1af43e7e5d12d", GitTreeState:"clean", BuildDate:"2022-12-08T19:57:06Z", GoVersion:"go1.19.4", Compiler:"gc", Platform:"linux/amd64"}
   ```

5. Kubernetesクラスタがアップグレード可能なVersionを確認します。

   ```shell
   sudo kubeadm upgrade plan
   ```

   実行例

   ```shell
   $ sudo kubeadm upgrade plan
   [upgrade/config] Making sure the configuration is correct:
   [upgrade/config] Reading configuration from the cluster...
   [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
   [preflight] Running pre-flight checks.
   [upgrade] Running cluster health checks
   [upgrade] Fetching available versions to upgrade to
   [upgrade/versions] Cluster version: v1.25.5
   [upgrade/versions] kubeadm version: v1.26.0
   [upgrade/versions] Target version: v1.26.0
   [upgrade/versions] Latest version in the v1.25 series: v1.25.5
   
   W1228 10:53:09.589092   39341 configset.go:177] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeproxy.config.k8s.io", Version:"v1alpha1", Kind:"KubeProxyConfiguration"}: strict decoding error: unknown field "udpIdleTimeout"
   Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
   COMPONENT   CURRENT       TARGET
   kubelet     5 x v1.25.4   v1.26.0
               1 x v1.26.0   v1.26.0
   
   Upgrade to the latest stable version:
   
   COMPONENT                 CURRENT   TARGET
   kube-apiserver            v1.25.5   v1.26.0
   kube-controller-manager   v1.25.5   v1.26.0
   kube-scheduler            v1.25.5   v1.26.0
   kube-proxy                v1.25.5   v1.26.0
   CoreDNS                   v1.9.3    v1.9.3
   etcd                      3.5.5-0   3.5.6-0
   
   You can now apply the upgrade by executing the following command:
   
           kubeadm upgrade apply v1.26.0
   
   _____________________________________________________________________
   
   
   The table below shows the current state of component configs as understood by this version of kubeadm.
   Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
   resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
   upgrade to is denoted in the "PREFERRED VERSION" column.
   
   API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
   kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
   kubelet.config.k8s.io     v1beta1           v1beta1             no
   _____________________________________________________________________
   
   ```

6. Static PodのManifestファイルの変更差分確認

   ```shell
   sudo kubeadm upgrade diff
   ```

   実行例

   ```shell
   $ sudo kubeadm upgrade diff
   [upgrade/diff] Reading configuration from the cluster...
   [upgrade/diff] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
   W1228 10:54:22.074337   40250 configset.go:177] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeproxy.config.k8s.io", Version:"v1alpha1", Kind:"KubeProxyConfiguration"}: strict decoding error: unknown field "udpIdleTimeout"
   --- /etc/kubernetes/manifests/kube-apiserver.yaml
   +++ new manifest
   @@ -19,7 +19,6 @@
        - --client-ca-file=/etc/kubernetes/pki/ca.crt
        - --enable-admission-plugins=NodeRestriction
        - --enable-bootstrap-token-auth=true
   -    - --enable-aggregator-routing=true
        - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
        - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
   ```

7. Control Planeのupgrade

   以下のコマンドでcontrol planeをupgradeします。

   ```shell
   sudo kubeadm upgrade apply v1.26.0
   ```

   実行例

   ```shell
   $ sudo kubeadm upgrade apply v1.26.0
   [upgrade/config] Making sure the configuration is correct:
   [upgrade/config] Reading configuration from the cluster...
   [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
   W1228 10:55:01.382813   40692 configset.go:177] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeproxy.config.k8s.io", Version:"v1alpha1", Kind:"KubeProxyConfiguration"}: strict decoding error: unknown field "udpIdleTimeout"
   [preflight] Running pre-flight checks.
   [upgrade] Running cluster health checks
   [upgrade/version] You have chosen to change the cluster version to "v1.26.0"
   [upgrade/versions] Cluster version: v1.25.5
   [upgrade/versions] kubeadm version: v1.26.0
   [upgrade] Are you sure you want to proceed? [y/N]: y
   [upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
   [upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
   [upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
   [upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.26.0" (timeout: 5m0s)...
   [upgrade/etcd] Upgrading to TLS for etcd
   [upgrade/staticpods] Preparing for "etcd" upgrade
   [upgrade/staticpods] Renewing etcd-server certificate
   [upgrade/staticpods] Renewing etcd-peer certificate
   [upgrade/staticpods] Renewing etcd-healthcheck-client certificate
   [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2022-12-28-10-55-27/etcd.yaml"
   [upgrade/staticpods] Waiting for the kubelet to restart the component
   [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
   [apiclient] Found 3 Pods for label selector component=etcd
   [upgrade/staticpods] Component "etcd" upgraded successfully!
   [upgrade/etcd] Waiting for etcd to become available
   [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests1888900047"
   [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
   [upgrade/staticpods] Renewing apiserver certificate
   [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
   [upgrade/staticpods] Renewing front-proxy-client certificate
   [upgrade/staticpods] Renewing apiserver-etcd-client certificate
   [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2022-12-28-10-55-27/kube-apiserver.yaml"
   [upgrade/staticpods] Waiting for the kubelet to restart the component
   [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
   [apiclient] Found 3 Pods for label selector component=kube-apiserver
   [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
   [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
   [upgrade/staticpods] Renewing controller-manager.conf certificate
   [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2022-12-28-10-55-27/kube-controller-manager.yaml"
   [upgrade/staticpods] Waiting for the kubelet to restart the component
   [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
   [apiclient] Found 3 Pods for label selector component=kube-controller-manager
   [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
   [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
   [upgrade/staticpods] Renewing scheduler.conf certificate
   [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2022-12-28-10-55-27/kube-scheduler.yaml"
   [upgrade/staticpods] Waiting for the kubelet to restart the component
   [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
   [apiclient] Found 3 Pods for label selector component=kube-scheduler
   [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
   [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
   [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
   [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
   [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
   [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
   [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
   [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
   [addons] Applied essential addon: CoreDNS
   [addons] Applied essential addon: kube-proxy
   
   [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.26.0". Enjoy!
   
   [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
   ```

   ControlPlane Nodeのマニフェストファイルを確認します。

   ```shell
   $ sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests/kube-controller-manager.yaml /etc/kubernetes/manifests/kube-scheduler.yaml /etc/kubernetes/manifests/etcd.yaml | grep image | grep -v imagePullPolicy
       image: k8s.gcr.io/kube-apiserver:v1.23.3
       image: k8s.gcr.io/kube-controller-manager:v1.23.3
       image: k8s.gcr.io/kube-scheduler:v1.23.3
       image: k8s.gcr.io/etcd:3.5.1-0
   ```

8. 残りのcontrol planeのupgrade

   2台目のcontrol planeにsshでログインします。

   kubeadmのupgrade

   ```shell
   sudo apt-mark unhold kubeadm && \
   sudo apt-get update && sudo apt-get install -y kubeadm=1.26.0-00 && \
   sudo apt-mark hold kubeadm
   ```

   control planeのupgrade（2台目以降）

   ```shell
   sudo kubeadm upgrade node
   ```

   実行結果

   ```shell
   $ sudo kubeadm upgrade node
   [upgrade] Reading configuration from the cluster...
   [upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
   [preflight] Running pre-flight checks
   [preflight] Pulling images required for setting up a Kubernetes cluster
   [preflight] This might take a minute or two, depending on the speed of your internet connection
   [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
   [upgrade] Upgrading your Static Pod-hosted control plane instance to version "v1.26.0"...
   [upgrade/etcd] Upgrading to TLS for etcd
   [upgrade/staticpods] Preparing for "etcd" upgrade
   [upgrade/staticpods] Renewing etcd-server certificate
   [upgrade/staticpods] Renewing etcd-peer certificate
   [upgrade/staticpods] Renewing etcd-healthcheck-client certificate
   [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2022-12-28-10-59-40/etcd.yaml"
   [upgrade/staticpods] Waiting for the kubelet to restart the component
   [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
   [apiclient] Found 3 Pods for label selector component=etcd
   [upgrade/staticpods] Component "etcd" upgraded successfully!
   [upgrade/etcd] Waiting for etcd to become available
   [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests3127600206"
   [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
   [upgrade/staticpods] Renewing apiserver certificate
   [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
   [upgrade/staticpods] Renewing front-proxy-client certificate
   [upgrade/staticpods] Renewing apiserver-etcd-client certificate
   [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2022-12-28-10-59-40/kube-apiserver.yaml"
   [upgrade/staticpods] Waiting for the kubelet to restart the component
   [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
   [apiclient] Found 3 Pods for label selector component=kube-apiserver
   [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
   [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
   [upgrade/staticpods] Renewing controller-manager.conf certificate
   [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2022-12-28-10-59-40/kube-controller-manager.yaml"
   [upgrade/staticpods] Waiting for the kubelet to restart the component
   [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
   [apiclient] Found 3 Pods for label selector component=kube-controller-manager
   [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
   [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
   [upgrade/staticpods] Renewing scheduler.conf certificate
   [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2022-12-28-10-59-40/kube-scheduler.yaml"
   [upgrade/staticpods] Waiting for the kubelet to restart the component
   [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
   [apiclient] Found 3 Pods for label selector component=kube-scheduler
   [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
   [upgrade] The control plane instance for this node was successfully updated!
   [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
   [upgrade] The configuration for this node was successfully updated!
   [upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
   ```

   ControlPlane Nodeのマニフェストファイルを確認します。

   ```shell
   $ sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests/kube-controller-manager.yaml /etc/kubernetes/manifests/kube-scheduler.yaml /etc/kubernetes/manifests/etcd.yaml | grep image | grep -v imagePullPolicy
       image: k8s.gcr.io/kube-apiserver:v1.23.3
       image: k8s.gcr.io/kube-controller-manager:v1.23.3
       image: k8s.gcr.io/kube-scheduler:v1.23.3
       image: k8s.gcr.io/etcd:3.5.1-0
   ```

   同様に、3台目のcontrol planeも同様にupgradeします。

   control plane nodeすべてのupgrade終了後、kubeletとkubectlのupgradeを実行します。

9. kubeletとkubectlのupgrade

   nodeのdrain

   kubeletとkubectlをupgradeする前に、nodeをdrainします。

   ```shell
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
   ```

   実行結果

   ```shell
   $ kubectl drain ip-192-168-20-137 --ignore-daemonsets --delete-emptydir-data --force
   node/ip-192-168-20-137 cordoned
   Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-5wngt, kube-system/kube-multus-ds-5mf9b, kube-system/kube-proxy-vd5l6, monitoring/monitoring-prometheus-node-exporter-9w2q6
   evicting pod tigera-operator/tigera-operator-6bb5985474-2j4hf
   evicting pod calico-apiserver/calico-apiserver-7b46f8789b-vspdx
   evicting pod calico-system/calico-kube-controllers-5d8f4d9f89-l2gf6
   evicting pod calico-apiserver/calico-apiserver-7b46f8789b-kjl5p
   evicting pod calico-system/calico-typha-5597444667-l2xbc
   pod/calico-typha-5597444667-l2xbc evicted
   pod/tigera-operator-6bb5985474-2j4hf evicted
   pod/calico-kube-controllers-5d8f4d9f89-l2gf6 evicted
   pod/calico-apiserver-7b46f8789b-kjl5p evicted
   pod/calico-apiserver-7b46f8789b-vspdx evicted
   node/ip-192-168-20-137 drained
   $ kubectl get nodes
   NAME                STATUS                     ROLES           AGE   VERSION
   ip-192-168-20-137   Ready,SchedulingDisabled   control-plane   14d   v1.25.4
   ip-192-168-20-156   Ready                      compute         14d   v1.25.4
   ip-192-168-20-184   Ready                      control-plane   14d   v1.25.4
   ip-192-168-20-211   Ready                      compute         14d   v1.25.4
   ip-192-168-20-217   Ready                      compute         14d   v1.25.4
   ip-192-168-20-64    Ready                      control-plane   14d   v1.25.4
   ```

   kubeletとkubectlのupgrade

   1台目のcontrol planeへsshでログインします。

   ```shell
   sudo apt-mark unhold kubelet kubectl && \
   sudo apt-get update && sudo apt-get install -y kubelet=1.26.0-00 kubectl=1.26.0-00 && \
   sudo apt-mark hold kubelet kubectl
   ```

   実行結果

   ```shell
   $ sudo apt-mark unhold kubelet kubectl && \
   sudo apt-get update && sudo apt-get install -y kubelet=1.26.0-00 kubectl=1.26.0-00 && \
   sudo apt-mark hold kubelet kubectl
   kubelet の保留を解除しました。
   kubectl の保留を解除しました。
   ヒット:1 https://download.docker.com/linux/ubuntu jammy InRelease
   ヒット:3 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu jammy InRelease
   ヒット:4 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu jammy-updates InRelease
   ヒット:5 http://ap-northeast-3.ec2.archive.ubuntu.com/ubuntu jammy-backports InRelease
   ヒット:2 https://packages.cloud.google.com/apt kubernetes-xenial InRelease
   ヒット:6 http://security.ubuntu.com/ubuntu jammy-security InRelease
   パッケージリストを読み込んでいます... 完了
   W: https://download.docker.com/linux/ubuntu/dists/jammy/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
   W: https://apt.kubernetes.io/dists/kubernetes-xenial/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
   パッケージリストを読み込んでいます... 完了
   依存関係ツリーを作成しています... 完了
   状態情報を読み取っています... 完了
   以下のパッケージはアップグレードされます:
     kubectl kubelet
   アップグレード: 2 個、新規インストール: 0 個、削除: 0 個、保留: 12 個。
   30.5 MB のアーカイブを取得する必要があります。
   この操作後に追加で 10.0 MB のディスク容量が消費されます。
   取得:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubectl amd64 1.26.0-00 [10.1 MB]
   取得:2 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubelet amd64 1.26.0-00 [20.5 MB]
   30.5 MB を 2秒 で取得しました (13.5 MB/s)
   (データベースを読み込んでいます ... 現在 64036 個のファイルとディレクトリがインストールされています。)
   .../kubectl_1.26.0-00_amd64.deb を展開する準備をしています ...
   kubectl (1.26.0-00) で (1.25.4-00 に) 上書き展開しています ...
   .../kubelet_1.26.0-00_amd64.deb を展開する準備をしています ...
   kubelet (1.26.0-00) で (1.25.4-00 に) 上書き展開しています ...
   kubectl (1.26.0-00) を設定しています ...
   kubelet (1.26.0-00) を設定しています ...
   Scanning processes...
   Scanning linux images...
   
   Running kernel seems to be up-to-date.
   
   No services need to be restarted.
   
   No containers need to be restarted.
   
   No user sessions are running outdated binaries.
   
   No VM guests are running outdated hypervisor (qemu) binaries on this host.
   kubelet は保留に設定されました。
   kubectl は保留に設定されました。
   ```

10. 設定再読み込み&kubeletの再起動

    ```shell
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
    ```

    control planeからログアウトします。

11. nodeのuncordon

    kubelet/kubectlをupgradeしたnodeをuncordonして、podをスケジューリングできるように戻します。

    ```shell
    kubectl uncordon <node>
    ```

    実行結果

    ```shell
    $ kubectl uncordon ip-192-168-20-137
    node/ip-192-168-20-137 uncordoned
    $ kubectl get nodes
    NAME                STATUS   ROLES           AGE   VERSION
    ip-192-168-20-137   Ready    control-plane   14d   v1.26.0
    ip-192-168-20-156   Ready    compute         14d   v1.25.4
    ip-192-168-20-184   Ready    control-plane   14d   v1.25.4
    ip-192-168-20-211   Ready    compute         14d   v1.25.4
    ip-192-168-20-217   Ready    compute         14d   v1.25.4
    ip-192-168-20-64    Ready    control-plane   14d   v1.25.4
    ```

12. 残りのcontrol planeのkubeletとkubectlのupgrade

    上記手順を繰り返して残りのcontrol planeのupgradeを実行します。

    3台目のcontrol planeをupgradeを実行し、すべてのcontrol planeのupgradeが終了しました。

    ```shell
    $ kubectl get nodes
    NAME                STATUS   ROLES           AGE   VERSION
    ip-192-168-20-137   Ready    control-plane   14d   v1.26.0
    ip-192-168-20-156   Ready    compute         14d   v1.26.0
    ip-192-168-20-184   Ready    control-plane   14d   v1.26.0
    ip-192-168-20-211   Ready    compute         14d   v1.26.0
    ip-192-168-20-217   Ready    compute         14d   v1.26.0
    ip-192-168-20-64    Ready    control-plane   14d   v1.26.0
    ```

    次にworker nodeのupgradeを実行します。

以上で、本ハンズオンは終了です。

