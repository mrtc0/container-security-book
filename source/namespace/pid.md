# PID Namespace

PID Namespace はプロセスの PID を分離します。コンテナの中で `ps` コマンドを実行すると PID 1 のプロセスが存在していることが確認できます。  
通常 Linux では重複した PID を持つプロセスを生成することはできませんが、Namespace が異なるため同じ PID を持っているかのように見えるプロセスを作ることができます。  

`unshare(1)` では `--pid` フラグを用いることで PID Namespace を作成できます。  

```sh
$ sudo unshare --pid --fork bash
root@docker:/home/ubuntu# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.2 167632 11704 ?        Ss   12:47   0:02 /sbin/init
root           2  0.0  0.0      0     0 ?        S    12:47   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        I<   12:47   0:00 [rcu_gp]
root           4  0.0  0.0      0     0 ?        I<   12:47   0:00 [rcu_par_gp]
```

`ps` コマンドを実行するとホスト側のプロセスも見えていますが、これは `ps` コマンドが `/proc` を見るからです。  
例えば `kill` コマンドを送信すると「No such process」というエラーが出るため、PID Namespace の分離自体はできていることが確認できます。

```sh
# ホスト側で実行
ubuntu@docker:~$ sleep 100

# Namespace 内で実行
root@docker:/home/ubuntu# ps aux | grep sleep
ubuntu      1545  0.0  0.0   7228   592 pts/1    S+   13:34   0:00 sleep 100
root        1547  0.0  0.0   8160   732 pts/0    S+   13:34   0:00 grep --color=auto sleep
root@docker:/home/ubuntu# kill -9 1545
bash: kill: (1545) - No such process
```

では `ps` コマンドでホスト側のプロセスを見えなくするにはどうすればいいでしょうか。  
PID Namespace で `procfs` を再マウントすればよいのですが、それだとホスト側に影響を与えてしまいます。  
そこで Mount Namespace も分離することでホスト側に影響を与えずに新しくマウントすることができます。  

Mount Namespace については後述するとして、 `unshare(1)` には `--mount-proc` オプションがあるため、これを利用します。  
これにより Mount Namespace を使って `procfs` をマウントしてくれます。

```sh
ubuntu@docker:~$ sudo unshare --pid --fork --mount-proc bash
root@docker:/home/ubuntu# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   8960  3876 pts/0    S    13:42   0:00 bash
root           8  0.0  0.0  10608  3256 pts/0    R+   13:42   0:00 ps aux
```

冒頭で「同じ PID を持っているかのように見える」と書きましたが、これは Namespace 内から見た話であり、ホスト側から見ると規約通り PID は重複していません。  

```sh
root@docker:/home/ubuntu# sleep 100 &
[1] 10
root@docker:/home/ubuntu# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   8960  3940 pts/0    S    13:42   0:00 bash
root          10  0.0  0.0   7228   592 pts/0    S    13:44   0:00 sleep 100
root          11  0.0  0.0  10608  3364 pts/0    R+   13:44   0:00 ps aux

# ホスト側で確認
ubuntu@docker:~$ ps aux | grep sleep
root        1656  0.0  0.0   7228   592 pts/0    S    13:44   0:00 sleep 100
ubuntu      1659  0.0  0.0   8160   736 pts/1    S+   13:44   0:00 grep --color=auto sleep
```
