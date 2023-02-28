# 第六回ハンズオン・ワークショップ

## 概要

本ハンズオンではKubernetesのストレージやデータ保護に関連して以下のハンズオンを実施します。

## Dynamic Provisioning について

`PersistentVolume`の割り当て方法

- Static Provisioning
  - クラスター管理者が事前に`StoragaClass`の設定と`PersistentVolume`の作成を行う
  - ユーザは`PersistentVolumeClaim`を作成してボリュームの要求を行う
- Dynamic Provisioning
  - クラスタ～管理者は事前に`StorageClass`の設定を行う
  - ユーザは`PersistentVolumrClaim`を作成してボリュームの要求を行う
  - PVCが作成されると要求内容に応じて自動的にPVが作成される

### NFS Provisioner

本環境では、NFSサーバをバックエンドストレージとする Dynamic Provisoning を行うために、`NFS Subdir External Provisioner`をHelmで導入しています。

https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

`NFS Subdir External Provisioner`がNFSサーバと連携して`PersistentVolumeClaim`作成時に自動的に`PersistentVolume`を作成します。

Helmで導入する際に作成される`StorageClass`を使用します。

```shell
kubectl get storageclass
```

## Velero 概要

本ハンズオンのバックアップ・リストアでは、Veleroを使用します。

Velero（旧称：Heptio Ark）は、Heptioが開発したKubernetesリソースと永続ボリュームのバックアップ・リストアを行うツールです。現在VMware社で開発されています。

https://github.com/vmware-tanzu/velero

バックアップ・リストアのほか、災害復旧（DR）やマイグレーションなどもユースケースに挙がっています。

<u>機能と特徴</u>

- クラスタ全体、Namespace単位、ラベル指定単位でのバックアップとリストアが可能
- リソースとPVともにバックアップ対象とすることができる
- スケジュール管理されたバックアップ取得およびバックアップデータ保持が可能
- バックアップ前後のhook処理が可能
- S3、GCS、Azure Blob Storage、Minioなどの複数のオブジェクトストレージに対応
- バックエンドのブロックおよびオブジェクトストレージの操作はバックエンドごとに固有のplugin（プロバイダと呼ばれる）で行うアーキテクチャを採用

## ハンズオン①

###  bastionサーバログイン

1. bastionサーバへログインし、ユーザなどを確認します。

   ```shell
   whoami
   ```

2. 作業用ディレクトリ作成・移動

   本ハンズオンでは、第一回でビルドしたJavaアプリケーションを使用します。

   ```shell
   mkdir ws-6
   cd ws-6
   ```

### PVCマニフェストファイル作成

1. マニフェストファイルの作成

   ボリュームの要求を行う`PersistentVolumeClaim`リソースのマニフェストファイルを作成します。

   `sample-pvc.yaml`

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: nginx-logs
     labels:
       app: nginx-with-pv
   spec:
     storageClassName: nfs-client
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 1Gi
   ```

### PersistentVolumeClaim作成

1. PVCの作成

   作成した`sample-pvc.yaml`を適用してPVCを作成します。

   ```shell
   kubectl apply -f sample-pvc.yaml
   ```

   作成後以下のコマンドで確認します。

   ```shell
   kubectl get pvc
   ```

   実行例

   ```shell
   kubectl get pvc
   NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   nginx-logs   Bound    pvc-27fadc15-759b-442f-8a47-35ce1ed20272   1Gi        RWX            nfs-client     3s
   ```

   STATUSがBoundでPVと紐づいています。紐づいているPVはVOLUME列のPVです。ランダムな文字列でPVが生成されていることがわかります。

### ステートレスサンプルアプリケーション作成

1. マニフェストファイルの作成

   PVCを利用しないサンプルアプリケーションのマニフェストファイルを作成します。

   `nginx-without-pv.yaml`

   ```yaml
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-without-pv
     labels:
       app: nginx-without-pv
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nginx-without-pv
     template:
       metadata:
         labels:
           app: nginx-without-pv
       spec:
         containers:
         - image: nginx:1.17.6
           name: nginx
           ports:
           - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
     labels:
       app: nginx-without-pv
     name: nginx-without-pv
   spec:
     ports:
     - port: 80
       targetPort: 80
     selector:
       app: nginx-without-pv
   ```

2. マニフェストファイルの適用

   作成した`nginx-without-pv.yaml`をクラスタへ適用します。

   ```shell
   kubectl apply -f nginx-without-pv.yaml
   ```

   ステートレスサンプルアプリケーションが作成されたことを確認します。

   ```shell
   kubectl get pods
   ```

3. サンプルファイルの作成

   作成したステートレスサンプルアプリケーションのシェルへ接続し、サンプルファイル`test.log`を作成します。

   ```shell
   kubectl exec -it [nginx-without-pv pod name] -- /bin/bash
   ```

   ***!!Attention!!***

   - [nginx-without-pv pod name]は、作成したPodの名前に置き換えてください。（`kubectl get pod`で確認）

   実行例

   ```shell
   kubectl exec -it nginx-without-pv-7999c56fdc-54lrn -- /bin/bash
   root@nginx-without-pv-7999c56fdc-54lrn:/#
   ```

   以下のコマンドでサンプルファイルを作成します。

   ```shell
   ls -l /var/log/nginx
   echo "Test hogehoge" >> /var/log/nginx/test.log
   ls -l /var/log/nginx
   ```

   実行例

   ```shell
   root@nginx-without-pv-7999c56fdc-54lrn:/# ls -l /var/log/nginx/
   total 0
   lrwxrwxrwx 1 root root 11 Dec 28  2019 access.log -> /dev/stdout
   lrwxrwxrwx 1 root root 11 Dec 28  2019 error.log -> /dev/stderr
   root@nginx-without-pv-7999c56fdc-54lrn:/# echo "Test hogehoge" >> /var/log/nginx/test.log
   root@nginx-without-pv-7999c56fdc-54lrn:/# ls -l /var/log/nginx/
   total 4
   lrwxrwxrwx 1 root root 11 Dec 28  2019 access.log -> /dev/stdout
   lrwxrwxrwx 1 root root 11 Dec 28  2019 error.log -> /dev/stderr
   -rw-r--r-- 1 root root 14 Feb 24 02:32 test.log
   root@nginx-without-pv-7999c56fdc-54lrn:/#
   ```

   `exit`コマンドでコンテナからbastionサーバのプロンプトに戻ります。

   ```shell
   root@nginx-without-pv-7999c56fdc-54lrn:/# exit
   ```

4. ステートレスサンプルアプリケーションの削除・再作成

   永続ボリュームを使用していないので、Pod削除時に作成したサンプルファイルが削除されることを確認します。

   ```shell
   kubectl delete -f nginx-without-pv.yaml
   ```

   再度ステートレスサンプルアプリケーションを作成し、Running状態になっていることを確認します。

   ```shell
   kubectl apply -f nginx-without-pv.yaml
   kubectl get pod
   ```

   再作成したPodのシェルに接続します。

   ```shell
   kubectl exec -it [nginx-without-pv pod name] -- /bin/bash
   ```

   ***!!Attention!!***

   - [nginx-without-pv pod name]は、作成したPodの名前に置き換えてください。（`kubectl get pod`で確認）

   実行例

   ```shell
   kubectl exec -it nginx-without-pv-7999c56fdc-b7vl4 -- /bin/bash
   root@nginx-without-pv-7999c56fdc-b7vl4:/#
   ```

   先の手順で作成したサンプルファイルが存在しないことを確認します。永続ボリュームを使用していないので削除されています。

   ```shell
   ls -l /var/log/nginx
   cat /var/log/nginx/test.log
   ```

   実行例

   ```shell
   root@nginx-without-pv-7999c56fdc-b7vl4:/# ls -l /var/log/nginx/
   total 0
   lrwxrwxrwx 1 root root 11 Dec 28  2019 access.log -> /dev/stdout
   lrwxrwxrwx 1 root root 11 Dec 28  2019 error.log -> /dev/stderr
   root@nginx-without-pv-7999c56fdc-b7vl4:/# cat /var/log/nginx/test.log
   cat: /var/log/nginx/test.log: No such file or directory
   root@nginx-without-pv-7999c56fdc-b7vl4:/#
   ```

   `exit`コマンドでコンテナからbastionサーバのプロンプトに戻ります。
   
   ```shell
   root@nginx-without-pv-7999c56fdc-b7vl4:/# exit
   exit
   ```
   
   次に永続ボリュームを使用してデータが永続化されることを確認します。

### ステートフルサンプルアプリケーション作成

1. マニフェストファイルの作成

   作成したPVCを利用するサンプルアプリケーションのマニフェストファイルを作成します。

   `nginx-with-pv.yaml`

   ```yaml
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-with-pv
     labels:
       app: nginx-with-pv
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nginx-with-pv
     template:
       metadata:
         labels:
           app: nginx-with-pv
         annotations:
           backup.velero.io/backup-volumes: nginx-logs
           pre.hook.backup.velero.io/container: nginx
           pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "echo Before Backup. Pre-Hook on `date +%Y%m%d-%H:%M:%S`."]'
           post.hook.backup.velero.io/container: nginx
           post.hook.backup.velero.io/command: '["/bin/bash", "-c", "echo After Backup. Post-Hook on `date +%Y%m%d-%H:%M:%S`."]'
       spec:
         volumes:
           - name: nginx-logs
             persistentVolumeClaim:
              claimName: nginx-logs
         containers:
         - image: nginx:1.17.6
           name: nginx
           ports:
           - containerPort: 80
           volumeMounts:
             - mountPath: "/var/log/nginx"
               name: nginx-logs
               readOnly: false
   ---
   apiVersion: v1
   kind: Service
   metadata:
     labels:
       app: nginx-with-pv
     name: nginx-with-pv
   spec:
     ports:
     - port: 80
       targetPort: 80
     selector:
       app: nginx-with-pv
   ```

   | フィールド                                     | 説明                                                         |
   | ---------------------------------------------- | ------------------------------------------------------------ |
   | spec.template.spec.volumes[]                   | Podで使用するボリュームの定義（リスト形式）<br />`persistentVolimeClaim`でPVC名を指定 |
   | spec.template.spec.containers[].volumeMounts[] | コンテナで利用する際のボリュームマウント定義（リスト形式）<br />`volumeMounts.name`でPodに定義したボリューム名を指定（volumes.name）<br />`volumeMounts.mountPath`でコンテナ内でマウントするパスを指定 |

2. マニフェストファイルの適用

   前段で作成した`nginx-with-pv.yaml`を適用し、サンプルアプリケーションを作成します。

   ```shell
   kubectl apply -f nginx-with-pv.yaml
   ```

   作成したサンプルアプリケーションがRunning状態になっていることを確認します。

   ```shell
   kubectl get pods
   NAME                                           READY   STATUS    RESTARTS      AGE
   nginx-with-pv-56bf89776d-q4qx9                 1/1     Running   0             28s
   ```

3. ボリュームの確認

   作成したPodのシェルに接続し、マウントしたボリュームにサンプルファイルを作成します。

   ```shell
   kubectl exec -it [nginx-with-pv pod name] -- /bin/bash
   ```

   ***!!Attention!!***

   - [nginx-with-pv pod name]は、作成したPodの名前に置き換えてください。（`kubectl get pod`で確認）

   実行例

   ```shell
   kubectl exec -it nginx-with-pv-56bf89776d-q4qx9 -- /bin/bash
   root@nginx-with-pv-56bf89776d-q4qx9:/#
   ```

   コンテナのファイルシステム`/var/log/nginx`にボリュームをマウントしています。

   ```shell
   ls -l /var/log/nginx
   ```

   実行例

   ```shell
   root@nginx-with-pv-56bf89776d-q4qx9:/# ls -l /var/log/nginx/
   total 0
   -rw-r--r-- 1 root root 0 Feb 24 01:51 access.log
   -rw-r--r-- 1 root root 0 Feb 24 01:51 error.log
   ```

   Nginxのアクセスログとエラーログのファイルがあります。

   以下のコマンドでサンプルファイルを作成します。

   ```shell
   echo "Test hogehoge" >> /var/log/nginx/test.log
   ls -l /var/log/nginx
   ```

   実行例

   ```shell
   root@nginx-with-pv-56bf89776d-q4qx9:/# echo "Test hogehoge" >> /var/log/nginx/test.log
   root@nginx-with-pv-56bf89776d-q4qx9:/# ls -l /var/log/nginx/
   total 4
   -rw-r--r-- 1 root root  0 Feb 24 01:51 access.log
   -rw-r--r-- 1 root root  0 Feb 24 01:51 error.log
   -rw-r--r-- 1 root root 14 Feb 24 01:59 test.log
   ```

   `exit`コマンドでシェルから抜けます。

   ```shell
   exit
   ```

   実行例

   ```shell
   root@nginx-with-pv-56bf89776d-q4qx9:/# exit
   exit
   ```

4. サンプルアプリケーションの削除・再作成

   永続ボリュームによってデータが永続化されていることを確認します。一旦作成したサンプルアプリケーションを削除します。

   ```shell
   kubectl delete -f nginx-with-pv.yaml
   ```

   実行例

   ```shell
   kubectl delete -f nginx-with-pv.yaml
   deployment.apps "nginx-with-pv" deleted
   service "nginx-with-pv" deleted
   ```

   サンプルアプリケーションのPodが削除されたことを確認します。

   ```shell
   kubectl get pod
   ```

   PVCの状態は変わらないことを確認します。

   ```shell
   kubectl get pvc
   ```

   実行例

   ```shell
   kubectl get pvc
   NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   nginx-logs   Bound    pvc-27fadc15-759b-442f-8a47-35ce1ed20272   1Gi        RWX            nfs-client     23m
   ```

   再度、サンプルアプリケーションを作成します。

   ```shell
   kubectl apply -f nginx-with-pv.yaml
   ```

   PodがRunning状態になっていることを確認します。再作成したので異なる名前で作成されています。

   ```shell
   kubectl get pod
   ```

   実行例

   ```shell
   kubectl get pods
   NAME                                           READY   STATUS    RESTARTS      AGE
   nginx-with-pv-56bf89776d-65w87                 1/1     Running   0             19s
   ```

   再びPodのシェルに接続します。

   ```shell
   kubectl exec -it [nginx-with-pv pod name] -- /bin/bash
   ```

   ***!!Attention!!***

   - [nginx-with-pv pod name]は、作成したPodの名前に置き換えてください。（`kubectl get pod`で確認）

   実行例

   ```shell
   kubectl exec -it nginx-with-pv-56bf89776d-65w87 -- /bin/bash
   root@nginx-with-pv-56bf89776d-65w87:/#
   ```

   サンプルアプリケーション削除前に作成した`test.log`ファイルが存在することを確認します。

   ```shell
   ls -l /var/log/nginx/
   cat /var/log/nginx/test.log
   ```

   実行例

   ```shell
   root@nginx-with-pv-56bf89776d-65w87:/# ls -l /var/log/nginx/
   total 4
   -rw-r--r-- 1 root root  0 Feb 24 01:51 access.log
   -rw-r--r-- 1 root root  0 Feb 24 01:51 error.log
   -rw-r--r-- 1 root root 14 Feb 24 01:59 test.log
   root@nginx-with-pv-56bf89776d-65w87:/# cat /var/log/nginx/test.log
   Test hogehoge
   root@nginx-with-pv-56bf89776d-65w87:/#
   ```

   永続ボリュームを使用してデータを永続化しているので、Podを再作成してもデータが削除されないことを確認できました。
   
   `exit`コマンドでシェルから抜けます。
   
   ```shell
   exit
   ```
   
   実行例
   
   ```shell
   root@nginx-with-pv-56bf89776d-65w87:/# exit
   ```

## ハンズオン②

Veleroを使用したKubernetesのバックアップ・リストアを実施します。本ハンズオンを実施するにあたって、各ユーザには`velero`NamespaceでVelero カスタムリソースを扱える権限を付与しています。

### バックアップ対象リソースの確認

1. ステートレスサンプルアプリケーションの確認

   ```shell
   kubectl get deployment,pod,service,pvc -l app=nginx-without-pv
   ```

   ※「-l」オプションはラベルセレクターです。特定のラベルを持つリソースのみを選択します。

   ※PVCを使用していないので結果に表示されません。

   実行例

   ```shell
   kubectl get deployment,pod,service,pvc -l app=nginx-without-pv
   NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/nginx-without-pv   1/1     1            1           21m
   
   NAME                                    READY   STATUS    RESTARTS   AGE
   pod/nginx-without-pv-7999c56fdc-b7vl4   1/1     Running   0          21m
   
   NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
   service/nginx-without-pv   ClusterIP   172.99.72.238   <none>        80/TCP    21m
   ```

2. ステートフルサンプルアプリケーションの確認

   ```shell
   kubectl get deployment,pod,service,pvc -l app=nginx-with-pv
   ```

   ※PVCを使用しているので結果に表示されます。

   実行例

   ```shell
   kubectl get deployment,pod,service,pvc -l app=nginx-with-pv
   NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/nginx-with-pv   1/1     1            1           53m
   
   NAME                                 READY   STATUS    RESTARTS   AGE
   pod/nginx-with-pv-7645b45685-dhr9g   1/1     Running   0          14m
   
   NAME                    TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)   AGE
   service/nginx-with-pv   ClusterIP   172.110.242.224   <none>        80/TCP    53m
   
   NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   persistentvolumeclaim/nginx-logs   Bound    pvc-27fadc15-759b-442f-8a47-35ce1ed20272   1Gi        RWX            nfs-client     78m
   ```

### BackupStorageLocation 設定

Veleroで取得したバックアップデータの保存先オブジェクトストレージの設定を行います。Veleroでは保存先オブジェクトストレージの設定を`BackupStorageLocation`リソースで管理します。

1. `BackupStorageLocation`の確認

   以下のコマンドを実行し、現在設定されている`BackupStorageLocation`を確認します。未設定のため出力されません。

   ```shell
   velero backup-location get -l user=[Your User Name]
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero backup-location get -l user=user20
     ```

2. `BackupStorageLocation`の設定

   ユーザごとに`BackupStorageLocation`の設定を`velero`コマンドで行います。

   ```shell
   velero backup-location create [Your User Name]-minio \
     --provider aws \
     --bucket [Your User Name] \
     --labels user=[Your User Name] \
     --config region=minio,s3ForcePathStyle=true,s3Url=http://192.168.10.189:9000
   ```

   本環境では事前に割り当てられたユーザ名と同名で`Minio`にバケットを作成しています。

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero backup-location create user20-minio \
       --provider aws \
       --bucket user20 \
       --labels user=user20 \
       --config region=minio,s3ForcePathStyle=true,s3Url=http://192.168.10.189:9000
     ```

   以下のコマンドを実行し、`BackupStorageLocation`を確認します。

   ```shell
   velero backup-location get -l user=[Your User Name]
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero backup-location get -l user=user20
     ```

### Veleroバックアップ・リストア実行

サンプルアプリケーションを対象にバックアップを取得します。

1. バックアップの取得

   ①以下のコマンドを実行し、ステートフルサンプルアプリケーションのバックアップを取得します。

   ```shell
   velero backup create [Your User Name]-backup-with-pv \
     --storage-location [Your User Name]-minio \
     --selector app=nginx-with-pv \
     --labels user=[Your User Name] \
     --include-namespaces [Your User Name]
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero backup create user20-backup-with-pv \
       --storage-location user20-minio \
       --selector app=nginx-with-pv \
       --labels user=user20 \
       --include-namespaces user20
     ```

   ②以下のコマンドを実行し、ステートレスサンプルアプリケーションのバックアップを取得します。

   ```shell
   velero backup create [Your User Name]-backup-without-pv \
     --storage-location [Your User Name]-minio \
     --selector app=nginx-without-pv \
     --labels user=[Your User Name] \
     --include-namespaces [Your User Name]
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero backup create user20-backup-without-pv \
       --storage-location user20-minio \
       --selector app=nginx-without-pv \
       --labels user=user20 \
       --include-namespaces user20
     ```

   ①、②で実行しているように、バックアップ対象がステートレスかステートフルかに関わらず同じような使用感でバックアップを取得することが可能です。

   #### Velero 関連の annotations について

   ハンズオン①でステートフルサンプルアプリケーションをデプロイする際、以下のannotationsを設定しています。これらはVeleroの挙動を調整するために使用します。

   ```yaml
     template:
       metadata:
         labels:
           app: nginx-with-pv
         annotations:
           backup.velero.io/backup-volumes: nginx-logs
           pre.hook.backup.velero.io/container: nginx
           pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "echo Before Backup. Pre-Hook on `date +%Y%m%d-%H:%M:%S`."]'
           post.hook.backup.velero.io/container: nginx
           post.hook.backup.velero.io/command: '["/bin/bash", "-c", "echo After Backup. Post-Hook on `date +%Y%m%d-%H:%M:%S`."]'
   ```

   | フィールド                             | 説明                                                         |
   | -------------------------------------- | ------------------------------------------------------------ |
   | `backup.velero.io/backup-volumes`      | PersistentVolumeのバックアップを取得する際の取得対象ボリューム名 |
   | `pre.hook.backup.velero.io/container`  | バックアップ実行前にHookを実行するコンテナ名                 |
   | `pre.hook.backup.velero.io/command`    | バックアップ実行前に実行するHookの内容                       |
   | `post.hook.backup.velero.io/container` | バックアップ実行後にHookを実行するコンテナ名                 |
   | `post.hook.backup.velero.io/command`   | バックアップ実行後に実行するHookの内容                       |

2. バックアップの確認

   以下のコマンドでバックアップ取得状況の確認を行います。STATUSが「InProgress」の場合バックアップ取得中で、「Completed」の場合バックアップ取得完了です。Completedになっていることを確認します。

   ```shell
   velero backup get -l user=[Your User Name]
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero backup get -l user=user20
     NAME                       STATUS       ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
     user20-backup-with-pv      Completed    0        0          2023-02-25 06:07:41 +0000 UTC   29d       user20-minio       app=nginx-with-pv
     user20-backup-without-pv   InProgress   0        0          2023-02-25 06:10:17 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     ```

   ステートフルサンプルアプリケーションにはPodのAnnotationでHook処理を定義しています。バックアップのログで定義した内容が実行されていることを確認します。

   ```shell
   velero backup logs [Your User Name]-backup-with-pv
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero backup logs user20-backup-with-pv
     ```

   ログに以下のようなHookに関するエントリを確認することができます。（以下はuser20の場合）

   ```shell
   time="2023-02-26T12:26:59Z" level=info msg="stdout: Before Backup. Pre-Hook on 20230226-12:26:59.\n" backup=velero/user20-backup-with-pv hookCommand="[/bin/bash -c echo Before Backup. Pre-Hook on `date +%Y%m%d-%H:%M:%S`.]" hookContainer=nginx hookName="<from-annotation>" hookOnError=Fail hookPhase=pre hookSource=annotation hookTimeout="{30s}" hookType=exec logSource="pkg/podexec/pod_command_executor.go:173" name=nginx-with-pv-8c96bd4f-xbssh namespace=user20 resource=pods
   
   （中略）
   
   time="2023-02-26T12:27:01Z" level=info msg="stdout: After Backup. Post-Hook on 20230226-12:27:01.\n" backup=velero/user20-backup-with-pv hookCommand="[/bin/bash -c echo After Backup. Post-Hook on `date +%Y%m%d-%H:%M:%S`.]" hookContainer=nginx hookName="<from-annotation>" hookOnError=Fail hookPhase=post hookSource=annotation hookTimeout="{30s}" hookType=exec logSource="pkg/podexec/pod_command_executor.go:173" name=nginx-with-pv-8c96bd4f-xbssh namespace=user20 resource=pods
   ```

   バックアップ取得が完了しました。ステートフルサンプルアプリケーションを一旦削除してリストアします。

3. サンプルファイルの削除

   ステートフルサンプルアプリケーションを削除する前にハンズオン①で作成した`test.log`ファイルを削除します。リストア時にこの`test.log`ファイルが戻っているか確認します。

   ```shell
   kubectl exec -it [nginx-with-pv pod name] -- /bin/bash
   ```

   ***!!Attention!!***

   - [nginx-with-pv pod name]は、作成したPodの名前に置き換えてください。（`kubectl get pod`で確認）

   実行例

   ```shell
   kubectl exec -it nginx-with-pv-7645b45685-dhr9g -- /bin/bash
   root@nginx-with-pv-7645b45685-dhr9g:/#
   ```

   `test.log`ファイルを削除します。

   ```shell
   rm /var/log/nginx/test.log
   ```

   `exit`コマンドでbastionサーバのプロンプトに戻ります。

   ```shell
   root@nginx-with-pv-7645b45685-dhr9g:/# exit
   exit
   ```

4. サンプルステートフルアプリケーションの削除

   以下のコマンドを実行してサンプルステートフルアプリケーションを削除します。

   ```shell
   kubectl delete deployment,service,pvc -l app=nginx-with-pv
   ```

   実行例

   ```shell
   kubectl delete deployment,service,pvc -l app=nginx-with-pv
   deployment.apps "nginx-with-pv" deleted
   service "nginx-with-pv" deleted
   persistentvolumeclaim "nginx-logs" deleted
   ```

   Deployment、Service、PVCリソースが削除されます。

   ```shell
   kubectl get deployment,service,pvc -l app=nginx-with-pv
   ```

5. リストアの実行

   以下のコマンドでリストアする際のもととなるバックアップを確認します。

   ```shell
   velero backup get -l user=[Your User Name]
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero backup get -l user=user20
     NAME                       STATUS       ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
     user20-backup-with-pv      Completed    0        0          2023-02-25 06:07:41 +0000 UTC   29d       user20-minio       app=nginx-with-pv
     user20-backup-without-pv   InProgress   0        0          2023-02-25 06:10:17 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     ```

   ステートフルアプリケーションをリストアするので、**[Your User Name]-backup-with-pv**を使用します。

   以下のコマンドでリストアを実行します。

   ```shell
   velero restore create [Your User Name]-restore-with-pv \
     --from-backup [Your User Name]-backup-with-pv \
     --include-namespaces [Your User Name] \
     --labels user=[Your User Name] \
     --selector app=nginx-with-pv
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero restore create user20-restore-with-pv \
       --from-backup user20-backup-with-pv \
       --include-namespaces user20 \
       --labels user=user20 \
       --selector app=nginx-with-pv
     ```

6. リストアの確認

   以下のコマンドでリストア状況を確認します。STATUSが「Completed」になっていたらリストア完了です。

   ```shell
   velero restore get -l user=[Your User Name]
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero restore get -l user=user20
     NAME                               BACKUP                  STATUS      STARTED                         COMPLETED                       ERRORS   WARNINGS   CREATED
                     SELECTOR
     user20-restore-with-pv             user20-backup-with-pv   Completed   2023-02-25 06:29:37 +0000 UTC   2023-02-25 06:29:42 +0000 UTC   0        0          2023-02-25 06:29:37 +0000 UTC   app=nginx-with-pv
     ```

   サンプルステートフルアプリケーションがリストアされていることを確認します。

   ```shell
   kubectl get deployment,pod,service,pvc -l app=nginx-with-pv
   ```

   以下のコマンドでPodのシェルに接続します。

   ```shell
   kubectl exec -it [nginx-with-pv pod name] -- /bin/bash
   ```

   ***!!Attention!!***

   - [nginx-with-pv pod name]は、作成したPodの名前に置き換えてください。（`kubectl get pod`で確認）

   実行例

   ```shell
   kubectl exec -it nginx-with-pv-7645b45685-dhr9g -- /bin/bash
   Defaulted container "nginx" out of: nginx, restore-wait (init)
   root@nginx-with-pv-7645b45685-dhr9g:/#
   ```

   サンプルファイル`test.log`がリストアされていることを確認します。

   ```shell
   ls -l /var/log/nginx
   cat /var/log/nginx/test.log
   ```

   実行例

   ```shell
   root@nginx-with-pv-7645b45685-dhr9g:/# ls -l /var/log/nginx/
   total 4
   -rw-r--r-- 1 root root  0 Feb 24 01:51 access.log
   -rw-r--r-- 1 root root  0 Feb 24 01:51 error.log
   -rw-r--r-- 1 root root 14 Feb 24 01:59 test.log
   root@nginx-with-pv-7645b45685-dhr9g:/# cat /var/log/nginx/test.log
   Test hogehoge
   ```

   バックアップ取得時点の状態にリストアできたことを確認できました。
   
   exitコマンドでbastionサーバのプロンプトに戻ります。
   
   ```shell
   root@nginx-with-pv-7645b45685-dhr9g:/# exit
   exit
   ```

### Velero スケジュールバックアップ実行

これまでは手動によるバックアップ実行を行ってきましたが、Veleroでスケジュールバックアップを実行できることを確認します。

1. Velero スケジュールバックアップ実行

   以下のコマンドを実行しスケジュールバックアップを実行します。

   ```shell
   velero schedule create [Your User Name]-schedule-without-pv \
     --storage-location [Your User Name]-minio \
     --selector app=nginx-without-pv \
     --labels user=[Your User Name] \
     --include-namespaces [Your User Name] \
     --schedule="*/1 * * * *"
   ```

   | 引数・オプション | 説明                                   |
   | ---------------- | -------------------------------------- |
   | --schedule       | スケジュール間隔の定義。cron形式で指定 |

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero schedule create user20-schedule-without-pv \
       --storage-location user20-minio \
       --selector app=nginx-without-pv \
       --labels user=user20 \
       --include-namespaces user20 \
       --schedule="*/1 * * * *"
     ```

   以下のコマンドでスケジュールバックアップが作成されたことを確認します。

   ```shell
   velero schedule get -l user=[Your User Name]
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero schedule get -l user=user20
     ```

2. スケジュールバックアップの確認

   以下のコマンドでスケジュールバックアップが取得できていることを確認します。

   ```shell
   velero backup get -l user=[Your User Name]
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero backup get -l user=user20
     NAME                                        STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
     user20-schedule-without-pv-20230226124300   Completed   0        0          2023-02-26 12:43:00 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     user20-schedule-without-pv-20230226124200   Completed   0        0          2023-02-26 12:42:00 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     user20-schedule-without-pv-20230226124100   Completed   0        0          2023-02-26 12:41:00 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     user20-schedule-without-pv-20230226124000   Completed   0        0          2023-02-26 12:40:00 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     user20-schedule-without-pv-20230226123900   Completed   0        0          2023-02-26 12:39:00 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     user20-schedule-without-pv-20230226123800   Completed   0        0          2023-02-26 12:38:00 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     user20-schedule-without-pv-20230226123700   Completed   0        0          2023-02-26 12:37:00 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     user20-schedule-without-pv-20230226123600   Completed   0        0          2023-02-26 12:36:00 +0000 UTC   29d       user20-minio       app=nginx-without-pv
     ```

   1分毎にバックアップを取得していることを確認します。各バックアップの接尾辞に取得した時間を表す文字列が付加されています。

3. スケジュールバックアップの削除

   ```shell
   velero schedule delete [Your User Name]-schedule-without-pv
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero schedule delete user20-schedule-without-pv
     ```

### リソースの削除

ハンズオンで使用したリソースを削除します。

1. Velero バックアップ・リストアの削除

   ```shell
   velero restore delete [Your User Name]-restore-with-pv
   velero backup delete -l user=[Your User Name]
   
   # 「velero backup delete -l user=[Your User Name]」でバックアップリソースが削除されない場合は以下のコマンドを実行
   kubectl -n velero delete backup [Your User Name]-backup-with-pv [Your User Name]-backup-without-pv
   ```

   ***!!Attention!!***

   - [Your User Name]は、割り当てられたbastionサーバのユーザ名に置き換えてください。

     （実行例）user20の場合

     ```shell
     velero restore delete user20-restore-with-pv
     velero backup delete -l user=user20
     
     # 「velero backup delete -l user=user20」でバックアップリソースが削除されない場合は以下のコマンドを実行
     kubectl -n velero delete backup user20-backup-with-pv user20-backup-without-pv
     ```

2. サンプルアプリケーションの削除

   ```shell
   kubectl delete -f nginx-with-pv.yaml -f nginx-without-pv.yaml
   kubectl delete -f sample-pvc.yaml
   ```

以上で、本ハンズオンは終了です。



