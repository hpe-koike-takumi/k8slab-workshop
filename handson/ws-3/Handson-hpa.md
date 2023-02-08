# 第三回ハンズオン・ワークショップ

## 概要

本ハンズオンは、Kubernetesの機能によるアプリケーションのオートスケーリングのハンズオンです。

以下のトピックを実施します。

- `HorizontalPodAutoScaler`によるPodのオートスケーリング

### 前提

- Docker Hubアカウントが作成済みであること
- ハンズオン実施マシンに`java`が導入済みであること
- ハンズオン実施マシンに`git`コマンドが導入済みであること
- ハンズオン実施マシンに`podman`コマンドが導入済みであること
- Kubernetesクラスタが導入済みであること

### 留意事項

- 本資料では Kubernetesの導入手順については説明しません。
- 本資料では AWSのリソース導入手順については説明しません。

本ハンズオンの実施内容はこちら

- サンプルアプリケーションデプロイ
- サンプルアプリケーションへの負荷掛けによるオートスケーリング

## HorizontalPodAutoscaler

HorizontalPodAutoscaler(HPA)は、CPU使用率などの負荷に応じて、DeploymentなどのPod数を自動的に増減させるKubernetesリソースです。ユーザトラフィックなどの急激な増加への対応や、休暇期間などリソースに余裕がある時期に、自動的にPod数を調整することで効率的かつ低負荷で運用することが可能です。

ハンズオン第一回で、手動でスケーリングを行いましたが、本ハンズオンではその作業を自動化します。

HPAがスケールアウト・インを判断する際のメトリクス

- Metics Server

  クラスタ内のリソース使用状況を集約するコンポーネント

  https://github.com/kubernetes-sigs/metrics-server

- 外部メトリクス

  Prometheusなどに集約されたメトリクス

## ハンズオン手順

### bastionサーバログイン

1. bastionサーバへログインし、ユーザなどを確認します。

   ```shell
   whoami
   podman version
   ```

2. 作業用ディレクトリを作成します。

   ```shell
   mkdir ws-3
   cd ws-3
   ```

### サンプルアプリケーションビルド・プッシュ

1. サンプルアプリケーションとして以下の`index.php`を作成します。

   `index.php`

   ```php
   <?php
     $x = 0.0001;
     for ($i = 0; $i <= 1000000; $i++) {
       $x += sqrt($x);
     }
     echo "OK!";
   ?>
   ```

2. `Dockerfile`を作成します。

   `Dockerfile`

   ```dockerfile
   FROM php:5-apache
   COPY index.php /var/www/html/index.php
   RUN chmod a+rx index.php
   ```

3. コンテナイメージをビルドします。

   ```shell
   podman image build -t [Your Dockerhub ID]/php-apache:v1 .
   ```

   **[Your Dockerhub ID]は作成いただいたご自身のDockerHubアカウントに置き換えてください。**

   ※ベースイメージの取得元選択では、「docker.io」を選択してください。

   ```shell
   STEP 1/3: FROM php:5-apache
   ? Please select an image:
       registry.fedoraproject.org/php:5-apache
       registry.access.redhat.com/php:5-apache
     ▸ docker.io/library/php:5-apache
       quay.io/php:5-apache
   ```

   ビルド実行後、ビルドしたイメージがローカルにあることを確認します。

   ```shell
   podman image ls
   ```

4. コンテナイメージをDockerHubへプッシュします。

   DpckerHubへご自身のアカウントでログインします。

   ```shell
   podman login docker.io
   ```

   イメージをプッシュします。

   ```shell
   podman image push [Your Dockerhub ID]/php-apache:v1
   ```

   **[Your Dockerhub ID]は作成いただいたご自身のDockerHubアカウントに置き換えてください。**

### サンプルアプリケーションデプロイ

DockerHubへプッシュしたコンテナイメージを使用してKubernetesクラスタへデプロイします。

1. マニフェストファイル作成

   `php-apache.yaml`

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: php-apache
   spec:
     selector:
       matchLabels:
         run: php-apache
     replicas: 1
     template:
       metadata:
         labels:
           run: php-apache
       spec:
         containers:
         - name: php-apache
           image: [Your Dockerhub ID]/php-apache:v1
           ports:
           - containerPort: 80
           resources:
             limits:
               cpu: 500m
             requests:
               cpu: 200m
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: php-apache
     labels:
       run: php-apache
   spec:
     ports:
     - port: 80
     selector:
       run: php-apache
   ```

   サンプルアプリケーション用DeploymentとServiceを定義します。

   **[Your Dockerhub ID]は作成いただいたご自身のDockerHubアカウントに置き換えてください。**

2. アプリケーションデプロイ

   ```shell
   kubectl apply -f php-apache.yaml
   ```

   デプロイ後、リソースを確認します。

   ```shell
   kubectl get deployment,service
   ```

### HorizontalPodAutoscaler作成

1. マニフェストを作成します。
   
   `php-apache-hpa.yaml`

   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: php-apache
   spec:
     minReplicas: 1
     maxReplicas: 10
     metrics:
     - resource:
         name: cpu
         target:
           averageUtilization: 50
           type: Utilization
       type: Resource
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: php-apache
   ```
   
   HPAは負荷に応じて、1から10台のあいだで（`minReplica`/`maxReplica`）Pod数を調整し、メトリクス平均CPU使用率を50%に保とうとします。（`metrics[].resource`）
   
2. HPA作成

   ```yaml
   kubectl apply -f php-apache-hpa.yaml
   ```

   作成後確認します。

   ```shell
   kubectl get hpa
   ```

   実行結果

   ```shell
   kubectl get hpa
   NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
   php-apache   Deployment/php-apache   0%/50%    1         10        1          22d
   ```

   HPA作成直後は、TARGETS列の分子が「unknown」と表示されるかもしれません。その場合はしばらく経ってから再度確認すると、数値が表示されます。

   ※HPAは以下のようにコマンドで作成することも可能です。
   
   ```shell
   kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
   ```

### 負荷掛け

1. 以下のコマンドでサンプルアプリケーションに負荷をかけます

   ```shell
   kubectl run load-generator --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
   ```

2. 状況確認

   しばらくして以下のコマンドを実行すると負荷が高まっていることを確認できます。

   ```shell
   kubectl get hpa
   ```

   DeploymentとPodの状況を確認します。Pod数が増えていることを確認します。

   ```shell
   kubectl get deployment,pod
   ```

3. 負荷の停止

   負荷掛け用Podを削除します。

   ```shell
   kubectl delete pod load-generator
   ```

   数分後、Deploymentの状態が元に戻ったことを確認します。

   ```shell
   kubectrl get deployment,pod
   ```
   
   最後にサンプルアプリケーションを削除します。
   
   ```shell
   kubectl delete -f php-apache.yaml
   ```

以上で、本ハンズオンは終了です。

