# Namespace

Linux Namespace はホストとの Isolation の要の一つです。  
ここでは Linux namespace を単に Namespace あるいは名前空間と呼ぶこととします。

Namespace は Linux カーネルの機能で、ホストと Namespace 内のプロセスとでリソースを分離することができます。  
コンテナごとに Namespace を持つことで、ホストや他のコンテナとの分離を実現しています。

Namespace には次の7つがあります。

| Namespace | 概要 |
|:---------:|:----:|
| Cgroup | Namespace ごとに cgroup を作成する |
| IPC | IPC や POSIX message queues などを分離 |
| Network | ネットワークデバイスやアドレスなどを分離 |
| Mount | ファイルシステムを分離 |
| PID | プロセスID を分離する |
| User | UID / GID を分離する |
| UTS | hostname を分離する |

例えばコンテナを作成したときにホスト側のプロセスは確認できませんし、ホスト名もホスト側とは異なります。  
これらは Namespace を使って実現されています。

```
root@3a7669ccdce1:/# hostname
3a7669ccdce1
root@3a7669ccdce1:/# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.2  0.0   4108  3440 pts/0    Ss   04:04   0:00 bash
root          10  0.0  0.0   5888  2860 pts/0    R+   04:04   0:00 ps aux
```

## Namespace の確認

コンテナ以外でも Namespace は使われています。現在利用されている Namespace とそのプロセスを一覧するには `lsns` コマンドを利用します。

```
ubuntu@docker:~$ sudo lsns
        NS TYPE   NPROCS   PID USER             COMMAND
4026531835 cgroup    143     1 root             /sbin/init
4026531836 pid       143     1 root             /sbin/init
4026531837 user      143     1 root             /sbin/init
4026531838 uts       139     1 root             /sbin/init
4026531839 ipc       143     1 root             /sbin/init
4026531840 mnt       135     1 root             /sbin/init
4026531860 mnt         1    33 root             kdevtmpfs
4026531992 net       143     1 root             /sbin/init
4026532210 mnt         2   412 root             /lib/systemd/systemd-udevd
4026532211 uts         2   412 root             /lib/systemd/systemd-udevd
4026532212 mnt         1   548 systemd-timesync /lib/systemd/systemd-timesyncd
4026532213 uts         1   548 systemd-timesync /lib/systemd/systemd-timesyncd
4026532214 mnt         1   627 systemd-network  /lib/systemd/systemd-networkd
4026532215 mnt         1   630 systemd-resolve  /lib/systemd/systemd-resolved
4026532272 mnt         1   693 root             /usr/sbin/irqbalance --foreground
4026532273 mnt         1   704 root             /lib/systemd/systemd-logind
4026532275 uts         1   704 root             /lib/systemd/systemd-logind
```

`NS` 列に記載されているのが Namespace の ID で、重複していないことがわかります。  
この Namespace の ID は `/proc/$PID/ns` で確認できます。

```
ubuntu@docker:~$ sudo ls -al /proc/1/ns
total 0
dr-x--x--x 2 root root 0 Nov 13 12:47 .
dr-xr-xr-x 9 root root 0 Nov 13 12:47 ..
lrwxrwxrwx 1 root root 0 Nov 13 12:48 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Nov 13 12:48 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Nov 13 12:47 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Nov 13 12:48 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Nov 13 12:48 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Nov 13 13:01 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Nov 13 12:48 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Nov 13 12:48 uts -> 'uts:[4026531838]'
```

Namespace は `unshare(2)` を利用して作成できます。`unshare(1)` を使うことで簡単に利用できるので試してみましょう。  
UTS namespace を作成し、その Namespace 内で bash を実行します。

```
ubuntu@docker:~$ sudo unshare --uts bash
root@docker:/home/ubuntu# hostname test
root@docker:/home/ubuntu# hostname
test
root@docker:/home/ubuntu#
```

`lsns` コマンドを実行すると Namespace `4026532216` で UTS 名前空間が作成されていることが確認できます。

```
ubuntu@docker:~$ sudo lsns | grep bash
4026532216 uts         1  1441 root             bash
```
