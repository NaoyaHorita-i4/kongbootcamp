# KongBootCamp

# ToDo
[0. 事前準備](#事前準備)<br>
[1. ゴールデンイメージの作成](#ゴールデンイメージの作成)<br>
[2. Dataplaneの起動と設定反映](#Dataplaneの起動と設定反映)<br>
3. BookInfoアプリのデプロイ
4. お試しでService/Routeの作成
7. ゴールデンイメージの準備
8. trivyでscan
9. github container registryにpush
5. WebIDE→KongDP→BookInfoアプリの接続確認
6. プラグイン実装
10. Dataplaneの起動までのIaC化
11. Prometheus/Grafanaのデプロイ
以下に従ってコマンド実行していけばOK。ただし、グローバルIPアドレスを自身で用意したものに変更すること。
https://qiita.com/ipppppei/items/c15acc5c7f7af3e7c289

12. メトリクス、Konnectの監査ログの取得、ngrokへの転送
以下に従ってメトリクスをPrometheusで取得できるようにする。
https://qiita.com/ipppppei/items/04046dc6362b51d98d15
kong_nginx_requests_totalで検索をかけるとメトリクスが見れる。

ngrokは使用せず、AKS上に監査ログ保管用エンドポイント、及び監査ログを標準出力するpythonアプリをpodとして起動することで、監査ログを確認する。
https://miro.com/app/board/uXjVI49P4Us=/?moveToWidget=3458764629947658993&cot=14

13. APIOpsの実装
14. 初期ユースケースの要件を満たしているか確認

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

# command list
```
coder@code-server:~$ pwd
/home/coder
```




DPのIPアドレスを確認

```
coder@code-server:~$ kubectl get service my-kong-kong-proxy -n kong
NAME                 TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)                      AGE
my-kong-kong-proxy   LoadBalancer   10.0.72.22   74.225.152.186   80:30477/TCP,443:30101/TCP   11m
```


Bookinfo.yamlのデプロイ
※yamlにenvを設定することで、productpageからのリクエストをKongに流すことができる。

```
coder@code-server:~$ kubectl apply -f bookinfo.yaml -n kong
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```


bookinfo用のIngressを作る
```
kubectl apply -f ./bookIngress.yaml -n bookinfo
```

ブラウザで以下にアクセスするとbookinfoのWeb画面が見れる。
http://productpage.74-225-133-33.nip.io/productpage

kubectl scale --replicas=0 deployment/details-v1 -n bookinfo





audit-logs関連のリソースを作成
```
kubectl apply -f auditlogs-pod.yaml
kubectl apply -f auditlogs-cm.yaml
kubectl get ing -n audit-logs
kubectl get pod -n audit-logs
```

audit-logs宛にAPIリクエストが発生した場合、以下でログを確認できる。
```
kubectl logs webhook-server -n audit-logs -f
```

Konnect側の監査ログ取得セットアップ
-  Organization　→ Audit Logs Setup →　Konnectタブ
- Endpoint：http://webhook.74-225-133-33.nip.io/
- Authorization Header：Bearer hoge # audit-logsはデモ用で認証かけていないのでダミーの値でOK。
- Log Format：Json
- View Advanced Fields　→　Disable SSL Verification (Do not recommend)を有効にする。
- disableをenableに変更。
- Save
- SatusでActice、200、timeを確認。
