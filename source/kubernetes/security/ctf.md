# CTF を通して学ぶ Kubernetes セキュリティ

ここまでで Linux コンテナと Kubernetes のセキュリティを学んできました。ここからは CTF(Capture The Flag) 形式のゲームを通してより実践的に学んでみましょう。  

## kubectf の概要

CTF とは与えられた問題を解き、特定のフォーマットを持つ文字列を探し当てるゲームです。[^1]  
今回は演習のために [kubectf](https://github.com/mrtc0/kubectf) を用意しました。  
kubectf は minikube で演習クラスタを手元の環境で作成します。namespace ごとに問題を用意しており、Kubernetes ノードあるいは同一 namespace のリソースのどこかに存在する Flag を見つけ出します。  
kubectf では、ある Pod に侵入できたケースを想定し、 `victim` と名前がついたコンテナから問題に取り組みます。  
ですので、クラスタ管理者として Pod 外から manifest を変更したり、ノードに SSH してはいけません。あくまで攻撃者として脆弱な Pod に侵入し、そこから実行できる権限の範囲で取り組んでください。

例えば次のように `treasure-hunt` という問題では `treasure-hunt` namespace に切り替え、 `victim` と名前がついた Pod に入り、その中で問題に取り組みます。

```shell
❯ kubens treasure-hunt

❯ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
docker-registry-6b5b55b44-zgvxf   1/1     Running   0          42m
victim-68bcd4465f-q4pkb           1/1     Running   0          42m

# victim コンテナの中から問題に取り組む
❯ kubectl exec -it victim-68bcd4465f-q4pkb sh
/ #
```

## kubectf のセットアップ

kubectf のセットアップに際し、以下のソフトウェアが必要です。

* minikube
* kubectl
* Docker Engine

これらのソフトウェアのインストールはそれぞれのマニュアルをご参照ください。[^2]

インストールが終わったら kubectf のリポジトリを clone し、インストールスクリプトを実行します。

```shell
$ git@git.pepabo.com:mrtc0/kubectf.git
$ cd kubectf && ./setup.sh
```

インストールスクリプトの実行が終わり、全ての namespace で Pod が動いていることを確認してください。

```shell
❯ kubectl get ns
NAME                    STATUS   AGE
can-you-keep-a-secret   Active   4d17h
default                 Active   4d17h
gatekeeper-system       Active   4d17h
kube-node-lease         Active   4d17h
kube-public             Active   4d17h
kube-system             Active   4d17h
mountme                 Active   4d17h
mountme2                Active   4d17h
sniff                   Active   41h
treasure-hunt           Active   4d17h

❯ kubectl get pods --all-namespace
NAMESPACE               NAME                                            READY   STATUS    RESTARTS   AGE
can-you-keep-a-secret   victim-56958dffc6-ddfhk                         1/1     Running   1          4d17h
gatekeeper-system       gatekeeper-audit-7d99d9d87d-xqsnf               1/1     Running   2          4d17h
gatekeeper-system       gatekeeper-controller-manager-f94cc7dfc-56f4v   1/1     Running   2          4d17h
gatekeeper-system       gatekeeper-controller-manager-f94cc7dfc-5qfm4   1/1     Running   2          4d17h
gatekeeper-system       gatekeeper-controller-manager-f94cc7dfc-b77rm   1/1     Running   3          4d17h
kube-system             coredns-f9fd979d6-8tgk8                         1/1     Running   2          4d17h
kube-system             etcd-minikube                                   1/1     Running   2          4d17h
kube-system             kube-apiserver-minikube                         1/1     Running   2          4d17h
kube-system             kube-controller-manager-minikube                1/1     Running   2          4d17h
kube-system             kube-proxy-ljjkx                                1/1     Running   2          4d17h
kube-system             kube-scheduler-minikube                         1/1     Running   2          4d17h
kube-system             storage-provisioner                             1/1     Running   5          4d17h
mountme                 victim-7c5745b4dc-t4hdw                         1/1     Running   2          4d17h
mountme2                victim-7c5745b4dc-b2cmf                         1/1     Running   2          4d17h
sniff                   client-66b6f5cdcf-v5n9j                         1/1     Running   0          15h
sniff                   server-79b88f567-v4xsp                          1/1     Running   1          42h
sniff                   victim-7f4dc947d5-b9npv                         1/1     Running   0          15h
treasure-hunt           docker-registry-6b5b55b44-2mm42                 1/1     Running   2          4d17h
treasure-hunt           victim-68bcd4465f-n7s2r                         1/1     Running   2          4d17h
```

これで準備は OK です。問題の詳細等はリポジトリの各問題のディレクトリにある README を参照してください。

---

[^1] : Jeopardy 形式で多く利用されるルールです。他の形式の CTF もあり、それぞれルールが異なります。
