# Capability

Linux には Capability と呼ばれる仕組みがあり、ファイル及びプロセスに対して権限を細かく設定することができます。  

例えばポート番号が1024以下で Listen する場合は特権が必要になります。つまり root として起動する必要があるのですが、 `CAP_NET_BIND_SERVICE` という Capability を与えることで起動することができます。

```sh
ubuntu@docker:~$ cp $(which python3) mypython3
# Capability がないので80番ポートで Listen できない
ubuntu@docker:~$ getcap mypython3
ubuntu@docker:~$ ./mypython3 -m http.server 80
Traceback (most recent call last):
  File "/usr/lib/python3.8/runpy.py", line 194, in _run_module_as_main
    return _run_code(code, main_globals, None,
  File "/usr/lib/python3.8/runpy.py", line 87, in _run_code
    exec(code, run_globals)
  File "/usr/lib/python3.8/http/server.py", line 1294, in <module>
    test(
  File "/usr/lib/python3.8/http/server.py", line 1249, in test
    with ServerClass(addr, HandlerClass) as httpd:
  File "/usr/lib/python3.8/socketserver.py", line 452, in __init__
    self.server_bind()
  File "/usr/lib/python3.8/http/server.py", line 1292, in server_bind
    return super().server_bind()
  File "/usr/lib/python3.8/http/server.py", line 138, in server_bind
    socketserver.TCPServer.server_bind(self)
  File "/usr/lib/python3.8/socketserver.py", line 466, in server_bind
    self.socket.bind(self.server_address)
PermissionError: [Errno 13] Permission denied

# Capability を付与したので Listen できる
ubuntu@docker:~$ sudo setcap 'cap_net_bind_service=+ep' ./mypython3
ubuntu@docker:~$ ./mypython3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Docker コンテナでは `--cap-add` や `--cap-drop` を利用して Capability を付与したり削除したりできます。  
[ping コマンドの実行は CAP_NET_RAW が必要][1]なので、drop することで実行できないことが確認できます。

```sh
$ docker run --rm -it ubuntu:latest bash
root@4e89e243ccee:/# apt-get -qq update ; apt-get -qq -y install iputils-ping
root@4e89e243ccee:/# ps
root@4e89e243ccee:/# getpcaps 1
1: = cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap+eip
root@4e89e243ccee:/# ping -c 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=113 time=17.9 ms

$ docker run --cap-drop net_raw --rm -it ubuntu:latest bash
root@31bb88ca04f8:/# apt-get -qq update ; apt-get -qq -y install iputils-ping
root@31bb88ca04f8:/# ps
root@31bb88ca04f8:/# getpcaps 1
1: = cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap+eiproot@31bb88ca04f8:/# ping 8.8.8.8
root@4e89e243ccee:/# ping 8.8.8.8
bash: /usr/bin/ping: Operation not permitted
```

Capability は数多くあり、中にはコンテナからホストに影響を及ぼすものがあります。  
そのため、過剰な Capability の付与は Breakout の原因となります。

簡単な例として `CAP_SYSLOG` があります。これをコンテナに付与すると `dmesg` 経由でカーネルのログを取得及び消去することができます。  

```sh
ubuntu@docker:~$ docker run --cap-add syslog --rm -it ubuntu:20.04
root@1577bebb133d:/# dmesg | head
[    0.000000] Linux version 5.4.0-47-generic (buildd@lcy01-amd64-014) (gcc version 9.3.0 (Ubuntu 9.3.0-10ubuntu2)) #51-Ubuntu SMP Fri Sep 4 19:50:52 UTC 2020 (Ubuntu 5.4.0-47.51-generic 5.4.55)
[    0.000000] Command line: earlyprintk=serial console=ttyS0 root=/dev/vda1 rw panic=1 no_timer_check
[    0.000000] KERNEL supported cpus:
[    0.000000]   Intel GenuineIntel
[    0.000000]   AMD AuthenticAMD
[    0.000000]   Hygon HygonGenuine
[    0.000000]   Centaur CentaurHauls
[    0.000000]   zhaoxin   Shanghai
[    0.000000] x86/fpu: Supporting XSAVE feature 0x001: 'x87 floating point registers'
[    0.000000] x86/fpu: Supporting XSAVE feature 0x002: 'SSE registers'
root@1577bebb133d:/# dmesg -C
root@1577bebb133d:/# dmesg
```

Docker や LXC などではこのような危険な Capability を[デフォルト][2]で付与しないようになっています。  
危険な Capability を与えた場合に生じる脆弱性については TODO をご参照ください。

[1] https://blog.ssrf.in/post/ping-does-not-require-cap-net-raw-capability/ "net.ipv4.ping_group_range を指定すると CAP_NET_RAW は必要ありません"
[2] https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities "Runtime privilege and Linux capabilities / docker docs"
