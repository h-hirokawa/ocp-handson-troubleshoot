# OpenShift トラブルシューティング ハンズオン - 実習手順

## 実習 1 - Cluster OperatorのStatus確認

以下のコマンドを実行して、Cluster Operatorに問題が発生していないことを確認。

```
oc get clusteroperator
```

全てが AVAILABLE: True であればOK

## 実習 2 - 検証用アプリケーションのデプロイ

```
# プロジェクトを作成する
oc new-project test-$(oc whoami)

# サンプルのアプリケーションをデプロイする
oc new-app cakephp-mysql-persistent

# BuildConfig が作成されることを確認する
oc get bc cakephp-mysql-persistent

# ビルドが開始されることを確認する
oc get build

# イメージのビルドをビルドログから確認する
oc logs -f bc/cakephp-mysql-persistent
# 「Push successful」が表示されたらCtrl+c でログ表示を終了

# DeploymentConfig が作成されたことを確認する
oc get dc

# cakephp-mysql-persistent pod のステータスが Running になるまで待機(Ctrl+cで終了)
oc get pod -w

# MySQL コンテナのログを確認する
oc logs $(oc get pod -oname -l name=mysql)

# Apache (CakePHP) コンテナのログを確認する
oc logs $(oc get pod -oname -l name=cakephp-mysql-persistent)

# Route のホスト名を確認する
oc get route

# ブラウザに http://<<Route のホスト名>> を入力し、アプリケーションの起動を確認する
```

## 実習 3 - プロセスダウン時挙動確認

```
# 作成したプロジェクトにスイッチする
oc project test-$(oc whoami)

# MySQL Pod の稼働状況を確認する
oc get pod -l name=mysql -o wide

# Projectのイベント発生状況を確認する
oc get event --sort-by='.lastTimestamp'

# コンテナ内のプロセスを Kill する
oc rsh $(oc get pod -oname -l name=mysql) kill 1

# イベントログからコンテナが再起動されたことを確認する
oc get event --sort-by='.lastTimestamp' | grep mysql

# 再起動後の Pod 稼働状態を再確認する
oc get pod -l name=mysql -o wide
## Podが再起動されRunning Statusとなり、Restart Countが増えている
```

## 実習 4 - Pod削除時挙動確認

```
# 作成したプロジェクトにスイッチする
oc project test-$(oc whoami)

# CakePHP Pod の稼働状況を確認する
oc get pod -l name=cakephp-mysql-persistent -o wide

# Pod を削除する
oc delete $(oc get pod -oname -l name=cakephp-mysql-persistent)

# イベントログからコンテナが再作成されたことを確認する
oc get event --sort-by='.lastTimestamp'

# Pod が再作成されていることを確認する
oc get pod -l name=cakephp-mysql-persistent -o wide
## 新規 Pod がデプロイされており、Name, Ageなどが変わっていることを確認
```

## 実習 5 - アプリ・ネットワークの確認

```
# 作成したプロジェクトにスイッチする
oc project test-$(oc whoami)

# CakePHP Route のホスト名を変数に格納する
route_host=$(oc get route cakephp-mysql-persistent -o template={{.spec.host}})
echo $route_host

# Route のホスト名を DNS サーバから名前解決できることを確認する
getent hosts $route_host

# Routeに対するhttpアクセスが成功することを確認する
curl -I $route_host

# CakePHP Service の IP アドレス, Port を確認する
oc get service cakephp-mysql-persistent -o wide

# CakePHP コンテナ内で Shell セッションを開始する
oc rsh -c cakephp-mysql-persistent $(oc get pod -oname -l name=cakephp-mysql-persistent)

# MySQL Serviceのホスト名を OpenShift 内部の DNS で名前解決できることを確認する
getent hosts mysql mysql.${OPENSHIFT_BUILD_NAMESPACE}.svc

# Route のホスト名を DNS サーバから名前解決できることを確認する
getent hosts <ROUTE_HOST>

# Service IPでの HTTP での疎通を確認する
curl -I <Service_IP>:8080 

# Pod から Service 経由での API Server への疎通を確認する
curl -k https://172.30.0.1/
curl -k https://kubernetes.default.svc/
```

## 実習 6 - Pod内でのパケットキャプチャ

```
# 作成したプロジェクトにスイッチする
oc project test-$(oc whoami)

# Pod 名を確認する
oc get pod -l name=cakephp-mysql-persistent -o wide

# Node に debug ログインする
oc debug node/<NODE>

# Pod 名と Namespace を環境変数に指定する
NAME=<Pod NAME>
NAMESPACE=test-<Username>

# 変数に Pod ID を格納する
pod_id=$(chroot /host crictl pods --namespace ${NAMESPACE} --name ${NAME} -q)
container_id=$(chroot /host crictl ps --pod ${pod_id} -q)

# pod の PID を取得する
pid=$(chroot /host bash -c "runc state $container_id | jq .pid")

# パケットダンプを取得する (コマンドはインタフェースの一覧取得）
nsenter -n -t $pid -- tcpdump -i eth0

# 別のターミナルを起動し、アプリケーション (Route のホスト名) に接続する
oc project test-$(oc whoami)
route_host=$(oc get route cakephp-mysql-persistent -o template={{.spec.host}})
curl $route_host

# キャプチャから Apache Pod の IP アドレスと MySQL Service の IP アドレス間での疎通を確認する

# MySQL Pod のレプリカ数を 0 に変更する
oc scale dc/mysql --replicas=0

# 再度アプリケーションへの接続を試みる
curl $route_host

# キャプチャから Apache Pod の IP アドレスと MySQL Service の IP アドレス間での疎通を確認する

# MySQL Pod のレプリカ数を 1 に戻す
oc scale dc/mysql --replicas=1

# 元のターミナルで以下を実施して debug を抜ける
## Ctrl+C でキャプチャを終了
exit
## 「Removing debug pod」のメッセージが表示される
```

## 実習 7 - Quota によるシステムリソースの制限

```
# 作成したプロジェクトにスイッチする
oc project test-$(oc whoami)

# Quota を作成する
oc create quota compute-resources --scopes=NotTerminating \
--hard=limits.cpu=2,limits.memory=2G,requests.cpu=1,requests.memory=1G,pods=4 

# 適用した Quota の設定を確認する
oc describe quota compute-resources

# DeploymentConfig に Requests と Limits を適用する
oc set resources dc/cakephp-mysql-persistent --requests=cpu=100m,memory=256Mi \
--limits=cpu=500m,memory=512Mi
oc set resources dc/mysql --requests=cpu=100m,memory=256Mi --limits=cpu=500m,memory=512Mi

# Pod のレプリカ数を 3 台にする
oc scale dc/cakephp-mysql-persistent --replicas=3

# Pod がスケールアウトされた台数を確認する
oc get pod -l name=cakephp-mysql-persistent

# Quota から要求されたリソースを確認する
oc describe quota compute-resources

# イベントログからスケールアウトがフェイルしたことを確認する
oc get event --sort-by='.lastTimestamp' | grep cakephp-mysql-persistent

# レプリカ数を元に戻す
oc scale dc/cakephp-mysql-persistent --replicas=1

以下, 参考出力
---
# Memory の Request や Limit に起因する場合の出力例
Error creating: pods "cakephp-mysql-persistent-1-mfsfm" is forbidden: exceeded quota: compute-resources, requested: limits.memory=512Mi,requests.memory=256Mi, used: limits.memory=1536Mi,requests.memory=768Mi, limited: limits.memory=2G,requests.memory=1G

# CPU の Request や Limit に起因する場合の出力例
Error creating: pods "mysql-2-srktf" is forbidden: exceeded quota: compute-resources, requested: limits.cpu=500m,pods=1, used: limits.cpu=2,pods=4, limited: limits.cpu=2,pods=4

# Pod レプリカ数 (台数) に起因する場合の出力例                                                                       
Error creating: pods "cakephp-mysql-persistent-1-xsf98" is forbidden: exceeded quota: compute-resources, requested: pods=1, used: pods=3, limited: pods=3
```
