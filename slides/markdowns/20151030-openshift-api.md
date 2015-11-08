class: center, middle

# OpenShift API 概要

---
# OpenShift/K8S REST API について

- OpenShift のコマンド `oc` や `oadm` は、API を叩いて操作 (kubectl も同様)

- OpenShift/kubernetes の API は Swagger という API のリファレンスを自動生成するためのフレームワークを利用

- Swagger を利用して、API のリファレンスを確認することが可能

---
# Swagger API について

- API のリファレンス

- 一覧情報の取得方法

~~~
    # Root doc
    $ curl -k https://ose3-master.example.com:8443/swaggerapi/

    # Kubernetes
    $ curl -k https://ose3-master.example.com:8443/swaggerapi/api/v1

    # OpenShift
    $ curl -k https://ose3-master.example.com:8443/swaggerapi/oapi/v1
~~~

- apis と models

  - apis ... REST API のエンドポイント

  - models ... 定義ファイル(e.g. hello-pod.yaml)のスキーマ

---
# CLIを使ったAPIの確認

```area
   $ oc get pod --loglevel=9

   $ kubectl get pod --loglevel=9
```

- 実行結果


```area
...
I1030 00:49:57.298397    6503 debugging.go:101] curl -k -v -XGET  -H "User-Agent: oc/v1.1.0 (linux/amd64) kubernetes/44c91b1" -H "Authorization: Bearer NeDHlfiw6YVGVOsZuUp4Y-yj7IxVsguxpin_aE0GnGk" https://ose3-master.example.com:8443/api/v1/namespaces/demo/pods
...
```

---
# 具体的な API 操作

- 事前準備

  - OpenShift の場合、認証 token を取得必要あり
  - K8Sはデフォルトで認証機能はないが、実用環境ではユーザーが何らかの認証方法を利用しているはず - 参考: [K8S doc](https://github.com/kubernetes/kubernetes/blob/master/docs/admin/authentication.md)

```area
   $ oc login -u <YOURSER_NAME> -p <PASSWORD>

   $ oc whoami -t
   NeDHlfiw6YVGVOsZuUp4Y-yj7IxVsguxpin_aE0GnGk
```

---
# 実行例

- pod 一覧

```area
$ curl -k  -H "Authorization: Bearer `oc whoami -t`" \
                       https://ose3-master.example.com:8443/api/v1/namespaces/demo/pods
```

- コンテナログ

```area
curl -k -XGET  -H "User-Agent: oc/v1.1.0 (linux/amd64) kubernetes/44c91b1" -H "Authorization: Bearer `oc whoami -t`" \
                   https://ose3-master.example.com:8443/api/v1/namespaces/demo/pods/mysql-55-rhel7-1-gzpsn/log?container=mysql-55-rhel7
```

- コンテナ作成

```area
curl -k -XPOST -H "Authorization: Bearer `oc whoami -t`" \
    https://ose3-master.example.com:8443/api/v1/namespaces/demo/pods \
     -d '{"kind":"Pod","apiVersion":"v1","metadata":{"name":"hello-openshift","namespace":"demo","creationTimestamp":null,"labels":{"name":"hello-openshift"}},"spec":{"containers":[{"name":"hello-openshift","image":"openshift/hello-openshift:v1.0.6","ports":[{"containerPort":8080,"protocol":"TCP"}],"resources":{},"terminationMessagePath":"/dev/termination-log","imagePullPolicy":"IfNotPresent","securityContext":{"capabilities":{},"privileged":false}}],"restartPolicy":"Always","dnsPolicy":"ClusterFirst"},"status":{}}'
```

---
# 関連 API ドキュメント

- https://docs.openshift.com/enterprise/3.0/rest_api/openshift_v1.html
- https://docs.openshift.com/enterprise/3.0/rest_api/kubernetes_v1.html
- https://blog.openshift.com/visualize-openshift-api-with-swagger/ (情報がやや古い)
- http://kubernetes.io/v1.0/docs/api.html


---
class: center middle
# END
