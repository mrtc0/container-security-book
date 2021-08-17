# etcd

Kubernetes のコントロールプレーンには、クラスタの情報を格納する KV ストアとして etcd があります。
etcd はデフォルトでは 2379/tcp で API を提供しています。

```shell
root@master1:/# ss -ntlp4 | grep etcd
LISTEN    0         4096             127.0.0.1:2379             0.0.0.0:*        users:(("etcd",pid=11583,fd=6))
LISTEN    0         4096            10.0.1.105:2379             0.0.0.0:*        users:(("etcd",pid=11583,fd=5))
LISTEN    0         4096            10.0.1.105:2380             0.0.0.0:*        users:(("etcd",pid=11583,fd=3))
```

保管する情報には Secret リソースも含まれるため、適切なアクセス制御や暗号化を施す必要があります。  
多くのクラスタ構築ツールでは外部公開せずに、TLS クライアント証明書による接続が必須になるように構成されます。

## etcd の操作

etcd は API を介して操作することができます。

```
root@master1:/# curl -s -k \
    --cert /etc/ssl/etcd/ssl/node-master1.pem \
    --key /etc/ssl/etcd/ssl/node-master1-key.pem \
    --cacert /etc/ssl/etcd/ssl/ca.pem \
    https://127.0.0.1:2379/version | jq
{
  "etcdserver": "3.4.13",
  "etcdcluster": "3.4.0"
}
```

また、etcdctl を使って操作することもできます。etcd のバージョンによって API が異なるため、`ETCDCTL_API` 環境変数で利用するバージョンを指定する必要があります。  
キーの一覧を取得すると Kubernetes クラスタのほとんどのリソースが含まれていることがわかります。

```
root@master1:/# etcdctl \
    --cert /etc/ssl/etcd/ssl/node-master1.pem \
    --key /etc/ssl/etcd/ssl/node-master1-key.pem \
    --cacert /etc/ssl/etcd/ssl/ca.pem \
    --endpoints https://127.0.0.1:2379 \
    get / --prefix --keys-only

/registry/apiextensions.k8s.io/customresourcedefinitions/bgpconfigurations.crd.projectcalico.org
/registry/apiextensions.k8s.io/customresourcedefinitions/bgppeers.crd.projectcalico.org
/registry/apiextensions.k8s.io/customresourcedefinitions/blockaffinities.crd.projectcalico.org
...
/registry/clusterrolebindings/calico-kube-controllers
/registry/clusterrolebindings/calico-node
/registry/clusterrolebindings/cephfs-csi-nodeplugin
...
/registry/clusterroles/admin
/registry/clusterroles/calico-kube-controllers
/registry/clusterroles/calico-node
...
/registry/configmaps/default/kube-root-ca.crt
/registry/configmaps/kube-node-lease/kube-root-ca.crt
/registry/configmaps/kube-public/cluster-info
...
/registry/controllerrevisions/kube-system/calico-node-f68f5dfc5
/registry/controllerrevisions/kube-system/kube-proxy-5bd89cc4b7
/registry/controllerrevisions/kube-system/nodelocaldns-84bfb6b45f
...
/registry/daemonsets/kube-system/calico-node
/registry/daemonsets/kube-system/kube-proxy
/registry/daemonsets/kube-system/nodelocaldns
...
/registry/deployments/kube-system/calico-kube-controllers
/registry/deployments/kube-system/coredns
/registry/deployments/kube-system/dns-autoscaler
...
/registry/namespaces/default
/registry/namespaces/kube-node-lease
/registry/namespaces/kube-public
...
/registry/pods/kube-system/calico-kube-controllers-7c5b64bf96-l54fz
/registry/pods/kube-system/calico-node-g7r9q
/registry/pods/kube-system/calico-node-pxzx4
...
/registry/secrets/kube-node-lease/default-token-jqg65
/registry/secrets/kube-public/default-token-rhw5q
/registry/secrets/kube-system/attachdetach-controller-token-dn6nl
...
```

値はシリアライズされたバイナリ値となっているため、可読性に乏しいですが、リソースのオブジェクトが入っていることがわかります。

```shell
root@master1:/# etcdctl \
    --cert /etc/ssl/etcd/ssl/node-master1.pem \
    --key /etc/ssl/etcd/ssl/node-master1-key.pem \
    --cacert /etc/ssl/etcd/ssl/ca.pem \
    --endpoints https://127.0.0.1:2379 \
    get /registry/namespaces/kube-system -w fields
"ClusterID" : 18321220406064639639
"MemberID" : 7551142479662027965
"Revision" : 1223524
"RaftTerm" : 3
"Key" : "/registry/namespaces/kube-system"
"CreateRevision" : 4
"ModRevision" : 4
"Version" : 1
"Value" : "k8s\x00\n\x0f\n\x02v1\x12\tNamespace\x12\xb6\x01\n\x9b\x01\n\vkube-system\x12\x00\x1a\x00\"\x00*$a65a4862-cb15-4a8d-9115-35cb3dd9a6422\x008\x00B\b\b\xc9\xffو\x06\x10\x00z\x00\x8a\x01O\n\x0ekube-apiserver\x12\x06Update\x1a\x02v1\"\b\b\xc9\xffو\x06\x10\x002\bFieldsV1:\x1d\n\x1b{\"f:status\":{\"f:phase\":{}}}\x12\f\n\nkubernetes\x1a\b\n\x06Active\x1a\x00\"\x00"
"Lease" : 0
"More" : false
"Count" : 1
```

## Secret の暗号化

etcd に含まれるデータはデフォルトでは暗号化されません。そのため、etcd の API にアクセスされた場合、Secret リソースも含めた Kubernetes クラスタに含まれるクレデンシャルが漏洩する可能性があります。

```shell
❯ kubectl create secret generic password --from-literal=pass='p@ssw0rd'
secret/password created

root@master1:/# etcdctl \
    --cert /etc/ssl/etcd/ssl/node-master1.pem \
    --key /etc/ssl/etcd/ssl/node-master1-key.pem \
    --cacert /etc/ssl/etcd/ssl/ca.pem \
    --endpoints https://127.0.0.1:2379 \
    get /registry/secrets/default/password -w fields
"ClusterID" : 18321220406064639639
"MemberID" : 7551142479662027965
"Revision" : 1228694
"RaftTerm" : 3
"Key" : "/registry/secrets/default/password"
"CreateRevision" : 1228577
"ModRevision" : 1228577
"Version" : 1
# `p@ssw0rd` が含まれている
"Value" : "k8s\x00\n\f\n\x02v1\x12\x06Secret\x12\xcc\x01\n\xaf\x01\n\bpassword\x12\x00\x1a\adefault\"\x00*$85ed6ed7-af50-4962-9450-29de336c26132\x008\x00B\b\b\xf9\xde\xed\x88\x06\x10\x00z\x00\x8a\x01_\n\x0ekubectl-create\x12\x06Update\x1a\x02v1\"\b\b\xf9\xde\xed\x88\x06\x10\x002\bFieldsV1:-\n+{\"f:data\":{\".\":{},\"f:pass\":{}},\"f:type\":{}}\x12\x10\n\x04pass\x12\bp@ssw0rd\x1a\x06Opaque\x1a\x00\"\x00"
"Lease" : 0
"More" : false
"Count" : 1
```

適切に暗号化を行うと、値は次のようになり、etcd の API 経由では平文は取得できなくなります。

```yaml
"Value" : "k8s:enc:aesgcm:v1:key1:ZZS(՝Uu\x1e\x81\xc0owc\x83j\x8e\x05\x8b\a(\x85\xb2\x0e\x8dd%\xc9!\xeey7\x..."
```

暗号化方式にはいくつかのパラメータが利用できますが、ローカルで管理している暗号化キーを使用する場合、EncryptionConfiguration のファイルに暗号化キーが保存されているため、ホストが侵害された場合は、復号される可能性があります。そのため、KMS のような Envelope 暗号化を利用する方式が推奨されます。

# Rerefences

- https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- https://etcd.io/
- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#securing-etcd-clusters
