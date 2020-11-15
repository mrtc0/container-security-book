# AppArmor

AppArmor は Linux Security Module (LSM) の一つで、Mandatory access control (MAC) を実現しています。  
アプリケーションごとにプロファイルを適用することができ、特定のファイルへのアクセスやシステムコールの呼び出しの制限を行うことができます。

例えば次のようなプロファイルを作成し、有効化することで `/home/ubuntu/mybash` は `/etc/hosts` の読み込みだけができ、他のファイルへの読み書きができなくなります。

```sh
$ cat /etc/apparmor.d/test
#include <tunables/global>

profile test /home/ubuntu/mybash {
    #include <abstractions/base>

    /etc/hosts r,
    /usr/bin/cat ix,
}

$ sudo apparmor_parser -r -W /etc/apparmor.d/test

$ ./mybash
mybash-5.0$ cat /etc/passwd
cat: /etc/passwd: Permission denied
mybash-5.0$ cat /etc/hosts
# Your system has configured 'manage_etc_hosts' as True.
...
127.0.0.1 localhost
...

mybash-5.0$ echo test >> /etc/hosts
mybash: /etc/hosts: Permission denied
```

Docker コンテナにも `default-docker` というプロファイル名で適用されており、多層防御の一つとして機能します。[^1]

```sh
$ sudo aa-status | grep docker
   docker-default
```

例えば、 `CAP_SYS_ADMIN` を付与した場合でも `mount` コマンドは AppArmor によって実行が防止されますが、AppArmor を外すことで実行することができ、AppArmor が最後の砦として機能していることが確認できます。

```sh
$ docker container run --rm -it --cap-add SYS_ADMIN --security-opt seccomp=unconfined ubuntu:latest bash
root@85c7ea124688:/# mkdir a; mkdir b; mount --bind a b
mount: /b: bind /a failed.

$ docker container run --rm -it --cap-add SYS_ADMIN --security-opt seccomp=unconfined --security-opt apparmor=unconfined ubuntu:latest bash
root@110e911e07bc:/# mkdir a; mkdir b; mount --bind a b
root@110e911e07bc:/#
```

コンテナ上で動くアプリケーションに対応したカスタムプロファイルを作成することで、コンテナをより強固にすることができます。

---

[^1]: https://github.com/moby/moby/blob/master/contrib/apparmor/template.go
