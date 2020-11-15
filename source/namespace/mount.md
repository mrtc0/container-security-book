# Mount namespace

Mount namespace はマウントポイントを分離することができます。PID namespace では `procfs` を unshare のプロセスにだけ見えるようにマウントしました。  
このように、プロセスごとに独自のマウントポイントを持つことができます。これを利用することで、例えばプロセスごとに異なる `tmpfs` をマウントすることで、他のプロセスから一切その内容を閲覧できないようにすることができます。  

`unshare(1)` では `--mount` フラグを用いることで Mount Namespace を作成できます。

```sh
ubuntu@docker:~$ sudo unshare --mount bash
root@docker:/home/ubuntu# mkdir /mnt/^C
root@docker:/home/ubuntu# mount -t tmpfs tmpfs /mnt
root@docker:/home/ubuntu# findmnt /mnt
TARGET SOURCE FSTYPE OPTIONS
/mnt   tmpfs  tmpfs  rw,relatime
root@docker:/home/ubuntu# touch /mnt/test
root@docker:/home/ubuntu# ls /mnt
test

# ホスト側で実行
ubuntu@docker:~$ findmnt /mnt
ubuntu@docker:~$ ls /mnt/
ubuntu@docker:~$
```

このように、名前空間にいるプロセスからしかマウントポイントが確認できなくなっています。  
これは Systemd の PrivateTmp でも利用されています。
