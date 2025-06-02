# KongBootCamp

# ToDo
1. KonnectからDataplaneの起動
2. DataPlaneへ接続確認
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

# command list
```
coder@code-server:~$ pwd
/home/coder
```

Konnectの画面に従いsecrets作成

```
coder@code-server:~$ kubectl create secret tls kong-cluster-cert -n kong --cert=/home/coder/tls.crt --key=/home/coder/tls.key
coder@code-server:~$ kubectl get secrets -n kong -o yaml
kubectl delete secret kong-cluster-cert -n kong
```

Konnectの画面に従いvalues.yaml作成＆helm install

```
coder@code-server:~$ helm install my-kong kong/kong -n kong --values ./values.yaml
NAME: my-kong
LAST DEPLOYED: Fri May 23 01:53:34 2025
NAMESPACE: kong
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To connect to Kong, please execute the following commands:

HOST=$(kubectl get svc --namespace kong my-kong-kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
PORT=$(kubectl get svc --namespace kong my-kong-kong-proxy -o jsonpath='{.spec.ports[0].port}')
export PROXY_IP=${HOST}:${PORT}
curl $PROXY_IP

Once installed, please follow along the getting started guide to start using
Kong: https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/getting-started/

WARNING: Kong Manager will not be functional because the Admin API is not
enabled. Setting both .admin.enabled and .admin.http.enabled and/or
.admin.tls.enabled to true to enable the Admin API over HTTP/TLS.
```

DPのIPアドレスを確認

```
coder@code-server:~$ kubectl get service my-kong-kong-proxy -n kong
NAME                 TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)                      AGE
my-kong-kong-proxy   LoadBalancer   10.0.72.22   74.225.152.186   80:30477/TCP,443:30101/TCP   11m
```

WebIDEからcurlでEXTERNAL-IPに投げてみる。

```
coder@code-server:~$ curl http://74.225.152.186
{
  "message":"no Route matched with those values",
  "request_id":"e7cf865b819435475476e51042d88e66"
}
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


Deckはインストール済みなので、設定入れる
```
deck --konnect-addr https://us.api.konghq.com \
  --konnect-control-plane-name kongbc \
  --konnect-token ${KONNECT_PAT} \
  gateway sync deck-bookinfo.yaml
```


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
