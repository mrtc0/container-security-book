# AppArmor

AppArmor はコンテナが侵害されて各保護レイヤを突破された場合に最後の砦となります。そのため、アプリケーションに適したプロファイルを適用することでコンテナをセキュアに運用することができます。  
一方で、AppArmor はシンタックスが複雑であったり、アプリケーションがどのファイルにアクセスしているか把握する必要があったりなど、プロファイルを生成するコストがかかるという面もあります。

ここでは AppArmor プロファイルを比較的簡単に生成する方法を紹介します。

## `docker diff` で変更されたファイルを一覧する

`docker diff` を使うことで変更されたファイルやディレクトリを取得することができます。例えば nginx コンテナを起動し、 `docker diff` を実行することで次のようなリストを取得することができます。

```sh
ubuntu@sandbox:~$ docker run --rm -it nginx:latest
...

ubuntu@sandbox:~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
ae9a66bf223e        nginx:latest        "/docker-entrypoint.…"   3 minutes ago       Up 3 minutes        80/tcp              sweet_archimedes
ubuntu@sandbox:~$ docker diff ae9a
C /run
A /run/nginx.pid
C /var
C /var/cache
C /var/cache/nginx
A /var/cache/nginx/proxy_temp
A /var/cache/nginx/scgi_temp
A /var/cache/nginx/uwsgi_temp
A /var/cache/nginx/client_temp
A /var/cache/nginx/fastcgi_temp
C /etc
C /etc/nginx
C /etc/nginx/conf.d
C /etc/nginx/conf.d/default.conf
```

ただし、これは **変更があったファイル** を表示するだけなので `open` や `read` されたファイルは取得することができません。  
そのため、それらのファイル一覧を取得するには eBPF などを用いてイベントトレースするなどの方法があります。  

例えば [opensnoop.py][1] 相当の機能を持たせた上で、コンテナのイベントのみ表示するようにすると、次のように開いたファイル一覧を取得することができます。

```sh
$ sudo ./cxray
{"data":{"container_id":"4604f9e03","event":{"name":"open","data":{"comm":"bash","fname":"/","pid":"1800","ret":"3","uid":"0"}}},"level":"info","msg":"","time":"2020-11-15T15:31:40+09:00"}
{"data":{"container_id":"4604f9e03","event":{"name":"open","data":{"comm":"cat","fname":"/etc/ld.so.cache","pid":"2371","ret":"3","uid":"0"}}},"level":"info","msg":"","time":"2020-11-15T15:31:41+09:00"}
{"data":{"container_id":"4604f9e03","event":{"name":"open","data":{"comm":"cat","fname":"/lib/x86_64-linux-gnu/libc.so.6","pid":"2371","ret":"3","uid":"0"}}},"level":"info","msg":"","time":"2020-11-15T15:31:41+09:00"}
{"data":{"container_id":"4604f9e03","event":{"name":"open","data":{"comm":"cat","fname":"/etc/passwd","pid":"2371","ret":"3","uid":"0"}}},"level":"info","msg":"","time":"2020-11-15T15:31:41+09:00"}
```

## AppArmor プロファイルを簡単に生成する

AppArmor プロファイルのシンタックスが難解なため、意図しない設定をしてしまうことがあります。  
そこで [genuinetolls/bane][2] というツールを紹介します。

これは TOML ファイルに禁止するコマンドやファイルパスを記述するだけで AppArmor プロファイルを生成するツールです。

例えば次のような TOML ファイルを用意します。

```toml
Name = "sample"

[Filesystem]
# read only paths for the container
ReadOnlyPaths = [
	"/bin/**",
	"/boot/**",
	"/dev/**",
	"/etc/**",
	"/home/**",
	"/lib/**",
	"/lib64/**",
	"/media/**",
	"/mnt/**",
	"/opt/**",
	"/proc/**",
	"/root/**",
	"/sbin/**",
	"/srv/**",
	"/tmp/**",
	"/sys/**",
	"/usr/**",
]

# paths where you want to log on write
LogOnWritePaths = [
	"/**"
]

# paths where you can write
WritablePaths = [
	"/var/run/nginx.pid"
]

# allowed executable files for the container
AllowExec = [
	"/usr/sbin/nginx"
]

# denied executable files
DenyExec = [
	"/bin/dash",
	"/bin/sh",
	"/usr/bin/top"
]
```

`bane` を使用すると AppArmor プロファイルを生成し、 `apparmor_parser` を実行してくれます。

```sh
ubuntu@bpf:~$ sudo ./bane sample.toml
Profile installed successfully you can now run the profile with
`docker run --security-opt="apparmor:docker-sample"`
ubuntu@bpf:~$ docker run --rm -it --security-opt="apparmor:docker-sample" nginx:latest bash
root@03b91bd97550:/# /bin/dash
bash: /bin/dash: Permission denied
root@03b91bd97550:/# sh
bash: /bin/sh: Permission denied
```

また、`docker-slim` を使って生成することも可能です。`docker-slim` のインストールは [ドキュメント][3] を参照してください。  

`docker-slim build` を実行し適当にコンテナの中でコマンドを実行します。今回は HTTP ポートを Expose しないため `--http-probe=false` を与えています。

```sh
ubuntu@vm:~/$ ./docker-slim build ubuntu:latest --http-probe=false
docker-slim: message='join the Gitter channel to ask questions or to share your feedback' info='https://gitter.im/docker-slim/community'
docker-slim: message='join the Discord server to ask questions or to share your feedback' info='https://discord.gg/9tDyxYS'
docker-slim[build]: info=probe message='changing continue-after from probe to enter because http-probe is disabled'
docker-slim[build]: state=started
docker-slim[build]: info=params target=ubuntu:latest continue.mode=enter rt.as.user=true keep.perms=true
docker-slim[build]: state=image.inspection.start
docker-slim[build]: info=image id=sha256:9140108b62dc87d9b278bb0d4fd6a3e44c2959646eb966b86531306faa81b09b size.bytes=72875723 size.human=73 MB
docker-slim[build]: info=image.stack index=0 name='ubuntu:latest' id='sha256:9140108b62dc87d9b278bb0d4fd6a3e44c2959646eb966b86531306faa81b09b'
docker-slim[build]: state=image.inspection.done
docker-slim[build]: state=container.inspection.start
docker-slim[build]: info=container status=created name=dockerslimk_6887_20201115065250 id=efa234652587e1367e9572b72abf534eb4e8190c603c5a46e77177a0a78094bd
docker-slim[build]: info=cmd.startmonitor status=sent
docker-slim[build]: info=event.startmonitor.done status=received
docker-slim[build]: info=container name=dockerslimk_6887_20201115065250 id=efa234652587e1367e9572b72abf534eb4e8190c603c5a46e77177a0a78094bd target.port.list=[] target.port.info=[] message='YOU CAN USE THESE PORTS TO INTERACT WITH THE CONTAINER'
docker-slim[build]: info=continue.after mode=enter message='provide the expected input to allow the container inspector to continue its execution'
docker-slim[build]: info=prompt message='USER INPUT REQUIRED, PRESS <ENTER> WHEN YOU ARE DONE USING THE CONTAINER'

docker-slim[build]: state=container.inspection.finishing
docker-slim[build]: state=container.inspection.artifact.processing
docker-slim[build]: state=container.inspection.done
docker-slim[build]: state=building message='building optimized image'
docker-slim[build]: state=completed
docker-slim[build]: info=results status='MINIFIED BY 9.95X [72875723 (73 MB) => 7322511 (7.3 MB)]'
docker-slim[build]: info=results  image.name=ubuntu.slim image.size='7.3 MB' data=true
docker-slim[build]: info=results  artifacts.location='/home/ubuntu/dist_linux/.docker-slim-state/images/9140108b62dc87d9b278bb0d4fd6a3e44c2959646eb966b86531306faa81b09b/artifacts'
docker-slim[build]: info=results  artifacts.report=creport.json
docker-slim[build]: info=results  artifacts.dockerfile.original=Dockerfile.fat
docker-slim[build]: info=results  artifacts.dockerfile.new=Dockerfile
docker-slim[build]: info=results  artifacts.seccomp=ubuntu-seccomp.json
docker-slim[build]: info=results  artifacts.apparmor=ubuntu-apparmor-profile
docker-slim[build]: state=done
docker-slim[build]: info=report file='slim.report.json'
docker-slim: message='join the Gitter channel to ask questions or to share your feedback' info='https://gitter.im/docker-slim/community'
docker-slim: message='join the Discord server to ask questions or to share your feedback' info='https://discord.gg/9tDyxYS'
```

実行を終了すると、上記メッセージにあるように `/home/ubuntu/dist_linux/.docker-slim-state/images/9140108b62dc87d9b278bb0d4fd6a3e44c2959646eb966b86531306faa81b09b/artifacts/ubuntu-apparmor-profile` に AppArmor プロファイルが生成されます。  

このようにツールの手助けを借りると簡単にプロファイルを生成することができます。

[1]: https://github.com/iovisor/bcc/blob/master/tools/opensnoop.py "https://github.com/iovisor/bcc/blob/master/tools/opensnoop.py"
[2]: https://github.com/genuinetools/bane "https://github.com/genuinetools/bane"
[3]: https://github.com/docker-slim/docker-slim#installation "https://github.com/docker-slim/docker-slim#installation"
