# seccomp

seccomp はシステムコールとその引数を制限する仕組みです。  
例えば Docker では次のような seccomp プロファイルを与えることで `mkdir` を禁止するコンテナを作成できます。

```sh
$ cat seccomp.json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "name": "mkdir",
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}

$ docker run --rm -it --security-opt seccomp=seccomp.json ubuntu:20.04 bash
root@ab9ad7d57f7f:/# mkdir /tmp/test
mkdir: cannot create directory '/tmp/test': Operation not permitted
```

Capability と同様に seccomp も Docker にはデフォルトプロファイルが存在します。[^1]  
Capability と併用することで、もし Capability を破られて特権が必要なシステムコールが呼び出されても seccomp で防ぐことができます。

---

[^1] https://docs.docker.com/engine/security/seccomp/ "Significant syscalls blocked by the default profile / docker docs"
