# KongBootCamp

# ToDo
[0. 事前準備](#事前準備)<br>
[1. ゴールデンイメージの作成](#ゴールデンイメージの作成)<br>
[2. Dataplaneの起動と設定反映](#Dataplaneの起動と設定反映)<br>
[3. BookInfoアプリのデプロイ](#BookInfoアプリのデプロイ)<br>
[4. Prometheus/Grafanaのデプロイ](#Prometheus/Grafanaのデプロイ)<br>
[5. 監査ログの取得](#監査ログの取得)<br>
[6. APIOpsの実装](#APIOpsの実装)<br>

# 事前準備
1. Kubernetesクラスタの用意
2. k8sクラスタ、Konnectへ接続可能な端末で各種ツールのインストール
- kubectl
- helm
- deck
3. Konnectのアカウント登録


# ゴールデンイメージの作成
1. GitHubActionsのworkflow作成<br>
./.github/workflow/imagepull.yaml
2. ワークフロー発火

# Dataplaneの起動と設定反映
## Dataplaneの起動
1. Konnectに接続　→　GatewayManagerで新規にCP作成<br>
2. Data Plane Nodesで Configure data plane<br>
3. 画面の指示に従う
- kongのnamespace作成
```
kubectl create namespace kong
```
- helmのrepo追加
```
helm repo add kong https://charts.konghq.com
```
- Update Helm
```
helm repo update
```
- 証明書作成
```
kubectl create secret tls kong-cluster-cert -n kong --cert=/{PATH_TO_FILE}/tls.crt --key=/{PATH_TO_FILE}/tls.key
```
- values.yamlの作成<br>
以下を修正、追記する。（./values.yaml 参照）
  - repository
  - tag
  - imagePullSecrets
  - serviceMonitor
  - status
- Apply the values.yaml
```
helm install my-kong kong/kong -n kong --values ./values.yaml
```
## 設定反映
1. deckのインストール
deckで接続先の設定、yamlファイルの反映。（./deck-bookinfo.yaml 参照）
```
deck --konnect-addr https://us.api.konghq.com \
  --konnect-control-plane-name ${CONTROLPLANE_NAME} \
  --konnect-token ${KONNECT_PAT} \
  gateway sync kong-config.yaml
```
※.deck.yamlに設定を保存しておくと便利。

# BookInfoアプリのデプロイ
1. https://github.com/imurata/bookinfo/platform/kube/bookinfo.yaml
をベースにして、details,ratings,reviews宛のリクエストをKong Gatewayに転送するようにproductpageのDeploymentを修正する。
（./bookinfo.yaml 参照）

2. yamlをapply
```
kubectl apply -f bookinfo.yaml -n kong
```

3. Bookinfoアプリをインターネット公開
Ingressを作成してグローバルIPアドレスで公開する。（./bookIngress.yaml 参照）
```
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install contour bitnami/contour --namespace projectcontour --create-namespace

kubectl apply -f ./bookIngress.yaml -n bookinfo
```

4. ブラウザで接続確認<br>
http://productpage.74-225-133-33.nip.io/productpage

# Prometheus/Grafanaのデプロイ
以下のリンクを参考にcert-managerとPromethes Operatorをインストールする。<br>
https://qiita.com/ipppppei/items/c15acc5c7f7af3e7c289

1. cert-managerのインストール
```
helm repo add jetstack https://charts.jetstack.io --force-update

helm repo update

helm install cert-manager jetstack/cert-manager --namespace cert-manager  --create-namespace --set installCRDs=true

# 確認
kubectl get all -n cert-manager

kubectl api-resources |grep cert-manager
```

2. Promethes Operatorのインストール
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update

helm show values prometheus-community/kube-prometheus-stack > prom-values.yaml
```
AlertManager、Prometheus、Grafanaについてvalues.yamlに変更を加える。<br>
※./prom-values.yamlと./prom-values_original.yamlで差分確認可能。<br>
prom-values.yaml中の公開グローバルIPアドレスは、Bookinfoアプリ公開時に作成したIngressのIPアドレスに変更すること。<br>

変更したvalues.yamlを用いてPrometheus/Grafanaをデプロイ。
```
helm upgrade -i -f prom-values.yaml prometheus-stack prometheus-community/kube-prometheus-stack -n prometheus-stack --create-namespace

# 確認
kubectl get ing -n prometheus-stack

kubectl get issuer,certificate -n prometheus-stack

kubectl get secret -n prometheus-stack |grep general

kubectl get secret -n prometheus-stack prometheus-general-tls -o jsonpath={.data.'ca\.crt'}  | base64 -d | openssl x509 -noout -text | grep -A 1 "Subject Alternative Name"
```

3. ブラウザで確認<br>
http://prometheus.74-225-133-33.nip.io/ <br>
https://grafana.74-225-133-33.nip.io/login <br>
※Grafanaはadmin/prom-operatorでログイン可能。

4. productpageにアクセスを繰り返すと、メトリクスが取得できていることを確認できる。

# 監査ログの取得
1. audit-logsアプリケーション関連のリソースを作成
```
kubectl apply -f auditlogs-pod.yaml

kubectl apply -f auditlogs-cm.yaml

kubectl get ing -n audit-logs

kubectl get pod -n audit-logs

kubectl logs webhook-server -n audit-logs -f
```


2. Konnectで監査ログのセットアップ
-  Organization　→ Audit Logs Setup →　Konnectタブ
- Endpoint：http://webhook.74-225-133-33.nip.io/
  - ※グローバルIPアドレスは各自の設定に変更すること。
- Authorization Header：Bearer hoge
  - ※audit-logsはデモ用で認証かけていないのでダミーの値でOK。
- Log Format：Json
- View Advanced Fields　→　Disable SSL Verification (Do not recommend)を有効にする。
- disableをenableに変更。
- Save
- SatusでActice、200、timeを確認。

# APIOpsの実装
以下のリポジトリのREADME及び./github/workflowsを参照。<br>
https://github.com/NaoyaHorita-i4/konnect-apiops-template/tree/main