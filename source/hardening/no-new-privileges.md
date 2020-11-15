# No New Privileges

Linux カーネルには No New Privileges[^1] と呼ばれる、子プロセスが新しい特権を取得できないようにする仕組みがあります。

setuid されたバイナリがコンテナの中にある場合、権限昇格につながる可能性がありますが、docker では `security-opt=no-new-privileges:true` というフラグを付与することで、これを防止できます。

次のように setuid された `mybash` を用意します。

```shell
$ cat Dockerfile
FROM ubuntu:18.04
RUN cp /bin/bash /bin/mybash && chmod +s /bin/mybash
RUN useradd -ms /bin/bash newuser
USER newuser
CMD ["/bin/bash"]
```

`no-new-privileges:false` の場合は `euid=0` で root 権限になっていることが確認できます。

```shell
$ docker run --security-opt=no-new-privileges:false -it --rm test:latest
newuser@3ee2685cd961:/$ id
uid=1000(newuser) gid=1000(newuser) groups=1000(newuser)
newuser@3ee2685cd961:/$ /bin/mybash -p
mybash-4.4# id
uid=1000(newuser) gid=1000(newuser) euid=0(root) egid=0(root) groups=0(root)
```

ここで `no-new-privileges:true` とすると新しいプロセスは特権を取得できないため、setuid されていても `euid=0` になることはありません。

```shell
$ docker run --security-opt=no-new-privileges:true -it --rm test:latest
newuser@80f241191a07:/$ /bin/mybash -p
newuser@80f241191a07:/$ id
uid=1000(newuser) gid=1000(newuser) groups=1000(newuser)
```

[^1]: https://www.kernel.org/doc/html/latest/userspace-api/no_new_privs.html "https://www.kernel.org/doc/html/latest/userspace-api/no_new_privs.html"
