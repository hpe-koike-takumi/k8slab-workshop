# 第二回ハンズオン・ワークショップ

## 概要

本ハンズオンは、Multus CNI を使用した閉域データベースへのアクセスのハンズオンです。

以下のトピックを実施します。

- Multus CNI による Pod へ複数vnic(veth)のアタッチ

### 前提

- Kubernetesクラスタが導入済みであること

### 留意事項

- 本資料では Kubernetesの導入手順については説明しません。
- 本資料では AWSのリソース導入手順については説明しません。

本ハンズオンの実施内容はこちら

- Multus CNIによるPodへのvnic追加
- サービスとデータベーストラフィックの分離

## Multus CNI

Multus CNI は、複数のCNIを呼び出すメタCNIです。

通常、Podには1つのvnicが割り当てられます。これはKubernetesクラスタ構築時にインストールしたCNIにより実行されます。Multus観点ではこれはデフォルトネットワークとも呼ばれ、Podのすべての通信をこのvnic経由で行います。

システムの要件で、サービス用とデータベース用のトラフィックを分離したり、ハードウェアに処理をオフロードさせたりする場合に、Multus CNIを使用します。

## ハンズオン手順

以下の手順で行います。

- Multus を使用しないクライアントによる接続
- `NetworkAttachmentDefinition`の作成
- Multusを使用するクライアントによる接続

本ハンズオンでは、閉域サブネットに配置したデータベースにMultus CNIを使用して接続します。最初に、Multus CNIを使用しないクライアントPodをデプロイしてデータベースにアクセスできないことを確認します。これは閉域データベースのセキュリティグループで、Multus CNIで使用するIPアドレスのみ許可しているため接続できないからです。

次に、Multus CNI を使用する準備として、`NetworkAttachmentDefinition`カスタムリソースを作成します。こちらのカスタムリソースで使用するCNIの種類や設定などを定義します。

最後に、Multusを使用したクライアントPodから閉域データベースへの接続を確認します。クライアントPodで作成した`NetworkAttachmentDefinition`を指定することで、Podに2つ目のvnicが追加され、指定した範囲からIPアドレスが自動的に払い出されます。払い出されたIPアドレスは閉域セキュリティグループで許可しているのでアクセスできます。

### bastionサーバログイン

1. bastionサーバへログインし、ユーザなどを確認します。

   ```shell
   whoami
   ```

2. 作業用ディレクトリを作成します。

   ```shell
   mkdir ws-2
   cd ws-2
   ```

### Multus を使用しないクライアントによる接続

1. マニフェストの作成

   `sample-client-not-nat.yaml`

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: sample-client-not-nad
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: sample-client-not-nad
     template:
       metadata:
         labels:
           app: sample-client-not-nad
       spec:
         nodeSelector:
           kubernetes.io/hostname: ip-192-168-20-140
         containers:
         - name: client-container
           image: jbergknoff/postgresql-client
           command: ["sleep"]
           args: ["infinity"]
   ```

2. マニフェスト適用

   ```shell
   kubectl apply -f sample-client-not-nat.yaml
   ```

   以下のコマンドで「sample-client-not-nad」から始まるPod名を確認します。

   ```shell
   kubectl get pods
   ```

   実行結果

   ```shell
   kubectl get pods
   NAME                                        READY   STATUS    RESTARTS        AGE
   sample-client-not-nad-6ff4554cc4-ggjwh      1/1     Running   2 (4h46m ago)   43h`
   ```

3. 閉域データベースへのアクセス確認

   クライアントPodへアクセスします。

   ```shell
   kubectl exec -it [sample-client-not-nad pod名] -- /bin/sh
   ```

   `ip`コマンドでPodにアタッチしているnicを確認します。Multusを使用していないので、1つのnicのみです。

   ```shell
   / # ip a
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
       inet6 ::1/128 scope host
          valid_lft forever preferred_lft forever
   3: eth0@if36: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9001 qdisc noqueue state UP
       link/ether d6:7c:fe:3f:b9:0f brd ff:ff:ff:ff:ff:ff
       inet 172.20.212.37/32 scope global eth0
          valid_lft forever preferred_lft forever
       inet6 fe80::d47c:feff:fe3f:b90f/64 scope link
          valid_lft forever preferred_lft forever
   ```

   ipコマンド確認後、`psql`コマンドで接続します。(`192.168.100.235`は本環境のdbインスタンスのIPアドレスです)

   ```shell
   psql -U admin -h 192.168.100.235 -p 5432 -d db
   ```

   パスワード入力プロンプトが表示されず接続できないことを確認します。ctrl + c でプロンプトに戻ります。

   Podからの踏み台サーバのシェルに戻ります。

   ```shell
   / # exit
   ```

### `NetworkAttachmentDefinition`の作成

MultusによるPodへのvnic追加のために、Multusのカスタムリソースである`NetworkAttachDefinition`を作成します。

1. マニフェストの作成

   `db-ipvlan-hostlocal-conf-worker3.yaml`

   ```shell
   apiVersion: "k8s.cni.cncf.io/v1"
   kind: NetworkAttachmentDefinition
   metadata:
     name: db-ipvlan-hostlocal-conf-worker3
   spec:
     config: '{
               "cniVersion": "0.4.0",
               "name": "db-ipvlan-hostlocal-conf-worker3",
               "plugins": [
                   {
                       "type": "ipvlan",
                       "master": "ens6",
                       "ipam": {
                           "type": "host-local",
                           "subnet": "192.168.100.0/24",
                           "rangeStart": "192.168.100.132",
                           "rangeEnd": "192.168.100.145",
                           "dataDir": "/var/myipam"
                       }
                   }
               ]
           }'
   ```

   `NetworkAttachmentDefinition`では、使用するCNIの設定を定義します。configをキーにJSON形式で定義します。

   `plugins`内の`type`で使用するCNIの種類を指定します。今回は`ipvlan`を使用しますが、他に`macvlan`プラグインなどがあります。（ https://www.cni.dev/plugins/current/main/ ）
   
   `ipam`ではIPアドレス管理方法を指定します。今回は、`host-local`というノード毎に特定範囲を管理し、その範囲からIPアドレスを払い出す方式ですが、`static`でIPアドレスを指定することも、`dhcp`でDHCPサーバを使用してIPアドレスを自動的に払い出すことも可能です。（ https://www.cni.dev/plugins/current/ipam/ ）
   
2. マニフェストの適用

   ```shell
   kubectl apply -f db-ipvlan-hostlocal-conf-worker3.yaml
   ```

   ファイル名は作成したものに合わせてください。

   以下のコマンドで作成されていることを確認します。

   ```shell
   kubectl get net-attach-def
   ```

### Multusを使用するクライアントによる接続

vnicを追加したPodを作成するために、Podのannotationsで`NetworkAttachDefinition`を指定します。

1. マニフェストの作成

   `sample-client-hostlocal.yaml`

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: sample-client-hostlocal
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: sample-client-hostlocal
     template:
       metadata:
         labels:
           app: sample-client-hostlocal
         annotations:
           k8s.v1.cni.cncf.io/networks: '[
           {
             "name": "db-ipvlan-hostlocal-conf-worker3"
           }
           ]'
       spec:
         nodeSelector:
           kubernetes.io/hostname: ip-192-168-20-140
         containers:
         - name: client-container
           image: jbergknoff/postgresql-client
           command: ["sleep"]
           args: ["infinity"]
   ```

   Multus CNI を Pod から使用する際は、`annotation`の`k8s.v1.cni.cncf.io/networks`で指定します。

   `host-local`ipamでノード毎に割り当てるIPアドレス範囲を指定しているので、クライアントPodをデプロイする際には、作成した`NetworkAttachmentDefinition`に合わせてNodeSelectorでノードを指定します。

2. マニフェストの適用

   ```shell
   kubectl apply -f sample-client-hostlocal.yaml
   ```

   以下のコマンドで「sample-client-hostlocal」から始まるPod名を確認します。

   ```shell
   kubectl get pods
   ```

   実行結果

   ```shell
   kubectl get pods
   NAME                                        READY   STATUS    RESTARTS        AGE
   sample-client-hostlocal-68469d59f-zckmh   1/1     Running   2 (4h46m ago)   43h
   ```

3. 閉域データベースへのアクセス確認

   クライアントPodへアクセスします。

   ```shell
   kubectl exec -it [sample-client-hostlocal pod名] -- /bin/sh
   ```

   `ip`コマンドでPodにアタッチしているnicを確認します。MultusによりPodに2つ目のvnicが追加されていることを確認します。

   ```shell
   / # ip a
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
       inet6 ::1/128 scope host
          valid_lft forever preferred_lft forever
   3: eth0@if37: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9001 qdisc noqueue state UP
       link/ether ca:12:a6:b8:49:2f brd ff:ff:ff:ff:ff:ff
       inet 172.20.212.36/32 scope global eth0
          valid_lft forever preferred_lft forever
       inet6 fe80::c812:a6ff:feb8:492f/64 scope link
          valid_lft forever preferred_lft forever
   4: net1@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UNKNOWN
       link/ether 0e:77:99:0e:40:0e brd ff:ff:ff:ff:ff:ff
       inet 192.168.100.37/24 brd 192.168.100.255 scope global net1
          valid_lft forever preferred_lft forever
       inet6 fe80::e77:9900:30e:400e/64 scope link
          valid_lft forever preferred_lft forever
   ```

   ipコマンド確認後、`psql`コマンドで接続します。(`192.168.100.235`は本環境のdbインスタンスのIPアドレスです)

   ```shell
   psql -U admin -h 192.168.100.235 -p 5432 -d db
   ```

   パスワード入力プロンプトが表示されます。パスワードを入力してアクセスします。

   ```shell
   Password for user admin:
   ```

   確認用クエリを実行します。

   ```sql
   select * from users;
   ```

   クエリ実行結果確認後、データベースから戻ります。

   ```shell
   db=# exit
   ```

   Podからの踏み台サーバのシェルに戻ります。

   ```shell
   / # exit
   ```

4. 最後にリソース削除します。

   Deploymentの削除

   ```shell
   kubectl delete -f sample-client-not-nat.yaml -f sample-client-hostlocal.yaml
   ```

   NetworkAttachmentDefinitionの削除

   ```shell
   kubectl delete -f db-ipvlan-hostlocal-conf-worker3.yaml
   ```

以上で、本ハンズオンは終了です。

