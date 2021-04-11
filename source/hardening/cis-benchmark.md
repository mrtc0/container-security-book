# CIS Benchmarks に準拠して Docker をセキュアにする

CIS Benchmarks[^1] は CIS (Center For Internet Security) というインターネット・セキュリティの標準化に取り組んでいる団体が発行しているベストプラクティスのガイドラインです。  
CIS Benchmarks の対象には OS の他に Apache などのミドルウェアがあり、Docker も含まれています。[^2]

CIS Benchmarks の特徴として、項目の検査内容や対応が明確なため、ツールとして落とし込みやすいことです。既に docker-bench-security[^3] など、ツール化されているものがあるため、気軽に試すことができます。

## docker-bench-security で CIS Benchmark に準拠する

docker-bench-security は CIS Docker Benchmark に準拠しているかを確認するツールです。  
大きく分けて次の7点をチェックします。

1. Host Configuration ... Docker daemon を実行しているホストの構成
2. Docker daemon configuration ... Docker daemon の設定
3. Docker daemon configuration files ... Docker daemon が利用するファイルのパーミッションなどの構成
4. Container Images and Build File ... ビルド済みイメージのセキュリティ
5. Container Runtimes ... コンテナランタイム
6. Docker Security Operations ... コンテナ運用におけるベストプラクティス
7. Docker Swarm Configuration ... Docker Swarm mode の設定

実行方法は次のようなコマンドで、コンテナとして動かすことができます。

```sh
ubuntu@sandbox:~$ docker run -it --net host --pid host --userns host --cap-add audit_control \
     -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
     -v /etc:/etc:ro \
     -v /usr/bin/containerd:/usr/bin/containerd:ro \
     -v /usr/bin/runc:/usr/bin/runc:ro \
     -v /usr/lib/systemd:/usr/lib/systemd:ro \
     -v /var/lib:/var/lib:ro \
     -v /var/run/docker.sock:/var/run/docker.sock:ro \
     --label docker_bench_security \
     docker/docker-bench-security
```

するとレポートが出力されます。問題がなければ「PASS」と表示されますが、準拠していない場合は「WARN」と出力されます。

```
[INFO] 1 - Host Configuration
[WARN] 1.1  - Ensure a separate partition for containers has been created
[NOTE] 1.2  - Ensure the container host has been Hardened
[INFO] 1.3  - Ensure Docker is up to date
[INFO]      * Using 19.03.13, verify is it up to date as deemed necessary
[INFO]      * Your operating system vendor may provide support and security maintenance for Docker
[INFO] 1.4  - Ensure only trusted users are allowed to control Docker daemon
[INFO]      * docker:x:999:ubuntu
[WARN] 1.5  - Ensure auditing is configured for the Docker daemon
[WARN] 1.6  - Ensure auditing is configured for Docker files and directories - /var/lib/docker
[WARN] 1.7  - Ensure auditing is configured for Docker files and directories - /etc/docker
...
```

例えば `1.5 Ensure auditing is configured for the Docker daemon` に対応するとしましょう。  
CIS Benchmark を確認すると、これは Docker daemon に対して audit が有効になっていないことを指摘しています。rootless Docker でない場合、Docker daemon は root 権限で動作するため、audit 対象としてセキュリティログを集めておくのがよいでしょう。

CIS Benchmark の Remediation 項目を確認すると auditd のルールに `-w /usr/bin/dockerd -k docker` を追加するという対応方法が記載されています。  
それに乗っ取り、対応を行い、Audit の項目のコマンドを実行して確認します。

```
ubuntu@sandbox:~$ sudo auditctl -l | grep /usr/bin/dockerd
-w /usr/bin/dockerd -p rwxa -k docker
```

これで問題ないようです。再度 docker-bench-security を実行すると `[PASS]` になることが確認できます。

## イメージをセキュアにする

Docker イメージをセキュアにするには、[Docker イメージのセキュリティ](../security/image/README.md)で紹介した「機密情報を含ませない」「脆弱性のあるパッケージをアップデートする」以外にも、いくつか気をつけるべき点があります。  
これも CIS Benchmark に「4. Container Images and Build File Configuration」として基準が示されており、docker-security-bench でスキャンすることが可能です。  
しかしながら docker-security-bench ではスコアとして計上されない Not Scored な項目がいくつか実装されていません。例えば次のような項目です。

* 機密情報を Dockerfile に含めない
* setuid / setgid されたバイナリの削除

また、CIS Benchmark で示されている基準以外にも `latest` タグを避ける、`sudo` コマンドを使用しないなどの観点もあります。  
これらを検出するために goodwithtech/dockle[^4] というツールがあるので、紹介します。

例えば `nginx:latest` イメージに対して実行すると次のような結果を得ます。

```
$ dockle nginx:latest
ubuntu@sandbox:~$ dockle nginx:latest
WARN    - CIS-DI-0001: Create a user for the container
        * Last user should not be root
WARN    - DKL-DI-0006: Avoid latest tag
        * Avoid 'latest' tag
INFO    - CIS-DI-0005: Enable Content trust for Docker
        * export DOCKER_CONTENT_TRUST=1 before docker pull/build
INFO    - CIS-DI-0006: Add HEALTHCHECK instruction to the container image
        * not found HEALTHCHECK statement
INFO    - CIS-DI-0008: Confirm safety of setuid/setgid files
        * setuid file: bin/mount urwxr-xr-x
        * setuid file: usr/bin/gpasswd urwxr-xr-x
        * setgid file: usr/bin/expiry grwxr-xr-x
        * setuid file: bin/su urwxr-xr-x
        * setgid file: usr/bin/wall grwxr-xr-x
        * setuid file: usr/bin/chfn urwxr-xr-x
        * setuid file: usr/bin/newgrp urwxr-xr-x
        * setuid file: usr/bin/chsh urwxr-xr-x
        * setuid file: bin/umount urwxr-xr-x
        * setgid file: sbin/unix_chkpwd grwxr-xr-x
        * setgid file: usr/bin/chage grwxr-xr-x
        * setuid file: usr/bin/passwd urwxr-xr-x
```

結果からは root ユーザーが使用されていたり、setuid されたバイナリが複数あることを確認できます。  

---

[^1]: https://www.cisecurity.org/cis-benchmarks/
[^2]: https://www.cisecurity.org/benchmark/docker/
[^3]: https://github.com/docker/docker-bench-security
[^4]: https://github.com/goodwithtech/dockle
